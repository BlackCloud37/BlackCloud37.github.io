# Linux 网络时间戳

为了衡量机内延迟，希望拿到收发包的时间戳

## 文档



## 网卡支持

```bash
$ ethtool -T eth0
Time stamping parameters for eth0:
Capabilities:
	software-transmit     (SOF_TIMESTAMPING_TX_SOFTWARE)
	software-receive      (SOF_TIMESTAMPING_RX_SOFTWARE)
	software-system-clock (SOF_TIMESTAMPING_SOFTWARE)
PTP Hardware Clock: none
Hardware Transmit Timestamp Modes: none
Hardware Receive Filter Modes: none
```

即服务器只支持软件时间戳

## 时间戳获取

以 TCP 为例

### RX
```c++
// 创建 socket 后
// 设置 SO_TIMESTAMPING 选项
{
    int timestamps = SOF_TIMESTAMPING_RX_SOFTWARE | SOF_TIMESTAMPING_SOFTWARE;
    if (setsockopt(sockfd, SOL_SOCKET, SO_TIMESTAMPING, &timestamps, sizeof(timestamps)) < 0)
    {
        perror("SO_TIMESTAMPING");
    }
}

// 用 recvmsg 接收数据
char *buf = new char[1024];
struct msghdr msg = {0};
struct iovec iov = {buf, 1024};
char control[CMSG_SPACE(sizeof(struct timespec))];

msg.msg_iov = &iov;
msg.msg_iovlen = 1;
msg.msg_control = control;
msg.msg_controllen = sizeof(control);

while (true)
{
    int size = recvmsg(sockfd, &msg, MSG_DONTWAIT);
    if (size == -1 && errno == EAGAIN)
        continue;

    for (auto cmsg = CMSG_FIRSTHDR(&msg); cmsg != NULL; cmsg = CMSG_NXTHDR(&msg, cmsg))
    {
        if (cmsg->cmsg_level == SOL_SOCKET && cmsg->cmsg_type == SO_TIMESTAMPING)
        {
            struct timespec* ts = (struct timespec*)CMSG_DATA(cmsg);
            printf("recv at %.9lf\n", ts->tv_sec + ts->tv_nsec * 1e-9);
        }
    }

    if (size > 0)
        break;
}
```

### TX

```c++
// 创建 socket 后
// 设置 SO_TIMESTAMPING 选项
{
    int timestamps = SOF_TIMESTAMPING_TX_SOFTWARE | SOF_TIMESTAMPING_SOFTWARE;
    if (setsockopt(connfd, SOL_SOCKET, SO_TIMESTAMPING, &timestamps, sizeof(timestamps)) < 0)
    {
        perror("SO_TIMESTAMPING");
    }
}

// 发送数据后, 时间戳会被写入到 ERRQUEUE 中
char buf[1024];
// ...
while (true)
{
    write(connfd, buf, size);

    // read tx timestamp
    {
        /* For transmit timestamps the outgoing packet is looped back to the socket’s error queue with the send timestamp(s) attached.
         * A process receives the timestamps by calling recvmsg() with flag MSG_ERRQUEUE set and with a msg_control buffer sufficiently large to receive the relevant metadata structures.
         * The recvmsg call returns the original outgoing data packet with two ancillary messages attached. */
        struct msghdr msg = {};
        char control[CMSG_SPACE(sizeof(struct timespec))];
        msg.msg_control = control;
        msg.msg_controllen = sizeof(control);

        if (recvmsg(connfd, &msg, MSG_ERRQUEUE) < 0)
        {
            perror("recvmsg");
            exit(1);
        }
        for (auto cmsg = CMSG_FIRSTHDR(&msg); cmsg != NULL; cmsg = CMSG_NXTHDR(&msg, cmsg))
        {
            if (cmsg->cmsg_level == SOL_SOCKET && cmsg->cmsg_type == SCM_TIMESTAMPING)
            {
                struct timespec *ts = (struct timespec *)CMSG_DATA(cmsg);
                printf("write %d bytes, tx timestamp: %.9lf\n", size, ts->tv_sec + ts->tv_nsec * 1e-9);
            }
        }
    }
}
```

## 测试

### 软件时间戳 overhead

在测试机上开启 RX 时间戳选项并通过 `recvmsg` 读取时间戳的 overhead 约为 100ns

### 软件时间戳生成时间点

文档描述
> Request rx timestamps when data enters the kernel. These timestamps are generated just after a device driver hands a packet to the kernel receive stack.
>
> 即内核从驱动拿到数据包后就生成时间戳

对于字节流
> The SO_TIMESTAMPING interface supports timestamping of bytes in a bytestream. Each request is interpreted as a request for when the entire contents of the buffer has passed a timestamping point. That is, for streams option SOF_TIMESTAMPING_TX_SOFTWARE will record when all bytes have reached the device driver, regardless of how many packets the data has been converted into.
>
> 即字节流的时间戳会 >= 最后一个字节对应的包的时间戳

实际测试如下场景：
- 发送端几次发送间隔较久(1s)，预期几个包也会间隔较久到达，如果内核立即处理并给每个包打时间戳，那么读到的 RX 时间戳间隔也应当是 1s。
- 但接收方如果不及时读数据，而是 sleep 一段时间等发送方发完再读，会发现有连续好几秒的包(定义包为发送端的一次发送)有相同的时间戳，且值大于等于最后一个包的时间戳。

说明这些数据可能还是在驱动或者内核排队了，并且内核进入打时间戳的时机时，这些数据包已经被整合成了一个 skb 或是什么结构，因此一起得到了较晚的时间戳

暂时没有找到获取更准确时间戳的方法
