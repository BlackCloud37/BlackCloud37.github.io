# io_uring 网络场景使用

## 背景
这两天在工作中发现 `epoll` + `recvmsg` 的组合在处理大量网络连接收数据的场景中耗时较长以至于成为了 bottleneck，思考可能是因为 syscall 次数太多，于是考虑 io_uring 能不能解决这个问题。

由于 io_uring 比较新，使用比 `epoll` 复杂但文档很少，记录一下个人学习后认为比较好的使用方式。仅考虑 C++/TCP 场景。这里不会注重解释框架的设计和细节，只是从使用者视角记录 roadmap。

可以先看[总结](#总结)。

## 上手
> 几乎所有 API 都用 [liburing](https://github.com/axboe/liburing) 封装的即可

io_uring 本质上是一个 *syscall queue*：用户可以往请求队列(*submission queue*)里添加请求，然后期望内核在收到事件并处理后将执行结果写入到响应队列(*completion queue*)里。队列里的元素分别叫 *submission queue entry(SQE)* 和 *completion queue entry(CQE)*。因为这两个队列是内核和用户空间共享内存的，所以读写队列不需要 syscall，这也是我期望其能有高性能的基础，不过在使用之后才发现这不是免费的 :(

使用上围绕一个 `struct io_uring` 展开，先设置必要的参数初始化这个结构

```c++
constexpr int ENTRIES = 1024;
struct io_uring ring;
struct io_uring_params params;
memset(&params, 0, sizeof(params));
// params.flags = ...
if (io_uring_queue_init_params(ENTRIES, &ring, &params) < 0)
{
    perror("io_uring_queue_init_params");
    exit(EXIT_FAILURE);
}
```

`ENTRIES` 就是 SQ 的大小，而 CQ 默认是两倍 `ENTRIES` 大，因为考虑一个请求可能会产生多个完成结果。

之后只需要往里面添加 syscall 请求即可，例如希望 TCP 建连：
```c++
/* 正常创建 fd 即可
* 值得注意的是，在使用 epoll 的时候，习惯把 fd 设成 non-blocking，在使用 io_uring 时就不需要了 */
int sockfd = ...;
struct sockaddr_in server_addr;
/* 填充 server_addr */

/* 生成一个 connect 请求 */

/* 由于可用的 sqe 是有限的，需要从 uring 里分配，如果用完了会返回 nullptr */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
assert(sqe != nullptr);

/* 基本上所有支持的 syscall 都会被命名成 io_uring_prep_xxx 的形式 */
io_uring_prep_connect(sqe, sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr));

/* 可以像 epoll 一样设置一个 u64 大小的 user_data, 例如指针 */
// io_uring_sqe_set_data(sqe, some_ptr);
struct user_data
{
    enum Op: uint32_t {
        CONNECT,
        RECV,
    };
    Op op;
    int sockfd;
};
static_assert(sizeof(user_data) == sizeof(__u64));
struct user_data data;
data.op = user_data::Op::CONNECT;
data.sockfd = sockfd;
memcpy(sqe->user_data, &data, sizeof(data));

/* 最后需要提交这个请求，内核才会看到这个请求并执行
* 注意这是个 syscall，因此如果有多个 sqe 需要提交，在所有 prep 之后调一次即可 */
io_uring_submit(&ring);
```

在默认模式下，内核收到这个请求后，会先立马检查请求需不需要等待，如果不需要就直接执行，否则会延后执行：在较老的 io_uring 实现里，会开一系列内核线程来不断地 `poll` 目标事件；在新的实现里，在事件就绪后内核会中断用户进程并切到内核态去执行。所谓的执行就是完成对应的 syscall 并把结果写入到 CQ 里这整个动作。

由于这些动作都在内核处理，在用户看来，提交任务之后过一段时间后去读 CQ 就能惊讶地发现请求已经完成了
```c++
while (true)
{
    struct io_uring_cqe *cqe;
    io_uring_peek_cqe(&ring, &cqe);

    if (cqe != nullptr) // 一段时间之后
    {
        /* 处理事件, 主要看 cqe->res 和 cqe->user_data
        * cqe->res 可以理解为对应 syscall 的返回值
        * cqe->user_data 用来对应自己的事件 */
        struct user_data data;
        memcpy(&data, cqe->user_data, sizeof(data));
        printf("op: %d, sockfd: %d, res: %d\n", data.op, data.sockfd, cqe->res);

        /* 处理完后，需要消费掉这个 CQE */
        io_uring_cqe_seen(&ring, cqe);
    }
}

```

这就是 io_uring 的基本使用方式了，连接建立之后，当然希望开始收数据，流程是类似的：
```c++
while (true)
{
    struct io_uring_cqe *cqe;
    io_uring_peek_cqe(&ring, &cqe);

    if (cqe != nullptr) // 一段时间之后
    {
        struct user_data data;
        memcpy(&data, cqe->user_data, sizeof(data));
        /* 处理完后，需要消费掉这个 CQE */
        io_uring_cqe_seen(&ring, cqe);

        /* 处理事件, 主要看 cqe->res 和 cqe->user_data
        * cqe->res 可以理解为对应 syscall 的返回值
        * cqe->user_data 用来对应自己的事件 */
        if (data.op == user_data::Op::CONNECT)
        {
            if (cqe->res != 0)
            {
                /* 错误处理 */
                // ...
                continue;
            }
            /* 建连成功, 准备收数据 */
            struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
            assert(sqe != nullptr);
            /* 注意, 这里的 buffer 需要是每个 sqe 独占的
            * 因此可以管理一个 per-connection 的 buffer */
            io_uring_prep_recv(sqe, data.sockfd, buffer, sizeof(buffer), 0);

            struct user_data data;
            data.op = user_data::Op::RECV;
            data.sockfd = data.sockfd;
            memcpy(sqe->user_data, &data, sizeof(data));
        }
        else if (data.op == user_data::Op::RECV)
        {
            if (cqe->res != 0)
            {
                /* 错误或者连接关闭 */
                // ...
                continue;
            }
            /* 处理收到的数据 */
            printf("recv %d bytes: %*s\n", cqe->res, cqe->res, buffer);
            /* 重新准备一个 recv 请求 */
            // struct io_uring_sqe *sqe = ...
        }
    }

    /* 提交刚才 prep 的请求 */
    io_uring_submit(&ring);
}
```

## 最佳实践
参考 [io_uring and networking in 2023](https://github.com/axboe/liburing/wiki/io_uring-and-networking-in-2023)

### IORING_SETUP_DEFER_TASKRUN
前面提到，io_uring 较新实现的默认模式里，事件 ready 后内核会中断用户进程，切换到内核态执行任务，所以看上去用户没有执行任何 syscall 就能 peek 到 CQE，但其实中间已经被打断过了。这样不仅没有减少切换到内核的开销，而且被打断的时间是不可控的。所以在默认模式下，io_uring 性能大概率不如 epoll。

为了避免这个问题，可以在初始化 io_uring 的时候设置 `IORING_SETUP_DEFER_TASKRUN` 标志(需要和 `IORING_SETUP_SINGLE_ISSUER` 一起设置)。这个标志的含义是，所有事件都不会再打断用户进程，而是会在用户显式指定内核处理之后才会被处理。

```c++
struct io_uring_params params;
memset(&params, 0, sizeof(params));
params.flags = IORING_SETUP_DEFER_TASKRUN | IORING_SETUP_SINGLE_ISSUER;
io_uring_queue_init_params(ENTRIES, &ring, &params);

// ...

/* 告知内核执行 pending 的请求
* 最原始的方法是用 io_uring_enter 结合 IORING_ENTER_GETEVENTS
* 不过 liburing 里有很多替代, 例如 io_uring_wait_*, io_uring_submit_and_wait 等等
*
* 因为我的使用场景中, 最常见的 case 就是在事件循环里不断地 tick, 每次 tick 都会 submit 和 enter(GETEVENTS)
* 所以用 io_uring_submit_and_get_events 是最符合语义的 */
while (true)
{
    io_uring_submit_and_get_events(&ring);
    
    // peek CQEs
    // ...
}
```

我觉得可以把 `io_uring_submit_and_get_events` 理解成 `epoll_wait` 和遍历所有 `epoll_event` 并处理 IO 这两个动作打包到一起。

### IORING_FEAT_FAST_POLL
io_uring 内部到底是使用多个内核线程还是中断用户进程，和这个标志相关。`FAST_POLL` 指代的是 io_uring 的 `internal polling mechanism`，可以理解为在内核做了类似 `epoll` 和 IO 的工作，来完成检测事件 ready 和处理 IO 并生产 CQE 的整套动作，这个实现应当比产生多个内核线程更高效。这个特性和内核版本有关，在 5.7 以后可用。

这个功能不需要手动开启，但是可以在 `io_uring_queue_init_params` 后检查 `params.features & IORING_FEAT_FAST_POLL` 来判断是否支持。

### 遍历 CQEs
比起每次 peek 一个 CQE 再 seen，可以使用 `io_uring_for_each_cqe` 来遍历所有 CQE，然后用 `io_uring_cq_advance` 来一次性消费。

```c++
while (true)
{
    io_uring_submit_and_get_events(&ring);

    unsigned head;
    unsigned count = 0;
    struct io_uring_cqe *cqe;
    io_uring_for_each_cqe(&ring, head, cqe) {
        ++count;
        /* 事件处理 */
    }
    io_uring_cq_advance(&ring, count);
}
```

### Ring Provide Buffer
之前提到收数据时，需要每个 sqe 独占一个 buffer，因为不知道哪个请求会先完成，因此也没法保证不会竞争使用 buffer。provide buffer 可以解决这个问题。

其思想是
1. 初始化一个 buffer pool，里面有一堆可用的 buffer entry
2. 在 prep 需要 buffer 的请求时告知 io_uring，在请求需要 IO 时自己从 buffer pool 里取用
3. 用户能在 CQE 里知道这次 IO 的数据在哪个 buffer 里
4. 消费完成后将 buffer 归还到 buffer pool 里

代码例子
```c++
#define BUF_SHIFT       12
#define BUFFER_SIZE     (1 << BUF_SHIFT)
#define BUFFERS         4096
#define BGID            0

/* 保存 provide buffer 相关的上下文 */
struct provide_buffer {
    struct io_uring_buf_ring *buf_ring;
    unsigned char *buffer_base;
    int buf_ring_size;
} pb;

/* 获取某个 buffer, index 可以唯一标识一个 buffer */
static unsigned char *get_buffer(struct provide_buffer *pb, int idx)
{
    return pb->buffer_base + (idx << BUF_SHIFT);
}

/* 初始化 buffer pool, 其中
* BUFFERS 是 buffer pool 里 entry 的数量, 每个 entry 可以用 index 唯一标示
* BGID 是这个 buffer pool 的 唯一标示 */
static int setup_buffer_pool(struct io_uring *ring)
{
    int ret, i;
    void *mapped;
    struct io_uring_buf_reg reg = {
        .ring_addr = 0,
        .ring_entries = BUFFERS,
        .bgid = BGID,
    };

    pb.buf_ring_size = (sizeof(struct io_uring_buf) + BUFFER_SIZE) * BUFFERS;
    mapped = mmap(NULL, pb.buf_ring_size, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_PRIVATE, 0, 0);
    if (mapped == MAP_FAILED) {
        perror("buf_ring mmap");
        exit(EXIT_FAILURE);
    }
    pb.buf_ring = (struct io_uring_buf_ring *)mapped;
    io_uring_buf_ring_init(pb.buf_ring);

    reg = (struct io_uring_buf_reg) {
        .ring_addr = (unsigned long)pb.buf_ring,
        .ring_entries = BUFFERS,
        .bgid = BGID,
    };
    pb.buffer_base = (unsigned char *)pb.buf_ring + sizeof(struct io_uring_buf) * BUFFERS;
    ret = io_uring_register_buf_ring(ring, &reg, 0);
    if (ret) {
        perror("buf_ring init failed");
        exit(EXIT_FAILURE);
    }

    for (i = 0; i < BUFFERS; i++) {
        io_uring_buf_ring_add(pb.buf_ring, get_buffer(&pb, i), BUFFER_SIZE, i, io_uring_buf_ring_mask(BUFFERS), i);
    }
    io_uring_buf_ring_advance(pb.buf_ring, BUFFERS);

    return 0;
}

/* 归还一个 buffer */
static void recycle_buffer(int idx)
{
    io_uring_buf_ring_add(pb.buf_ring, get_buffer(&pb, idx), BUFFER_SIZE, idx, io_uring_buf_ring_mask(BUFFERS), 0);
    io_uring_buf_ring_advance(pb.buf_ring, 1);
}

/* 使用 provide buffer 的 recv 例子 */
{
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    assert(sqe != nullptr);

    io_uring_prep_recv(sqe, sockfd, nullptr, 0, 0);
    io_uring_sqe_set_flags(sqe, IOSQE_BUFFER_SELECT); // 告知使用 provide buffer
    sqe->buf_group = BGID; // 告知使用哪个 buffer pool
}
```

### Multi-shot
像收数据或是 TCP server accept 这种场景，基本上写起来都是 prep 并等待完成，完成之后马上重新 prep 一次。对于这种操作，io_uring 提供了 multishot 模式：prep 一次之后，这个请求可以被重复触发，直到出错或被用户取消。

对于 `recv_multishot`，因为每次触发都需要一个额外的 buffer，所以必须配合 [provide buffer](#ring-provide-buffer) 使用。

```c++
void add_socket_recv_multishot(struct io_uring *ring, int fd, unsigned flags)
{
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    assert(sqe != nullptr);
    io_uring_prep_recv_multishot(sqe, fd, 0, 0, 0);
    io_uring_sqe_set_flags(sqe, flags | IOSQE_BUFFER_SELECT);
    sqe->buf_group = BGID;
}
```

### Timeout
对于 connect，可能会希望给其设置一个 timeout，`io_uring_prep_link_timeout` 挺适合这种场景。

基本思路是，将一个 timeout 请求和另一个(组)请求绑定到一起，如果时间到了对应事件还没(全部)完成，就触发 timeout 并取消所有绑定的请求；否则取消 timeout。

例子可以在 [liburing/test/link-timeout.c](https://github.com/axboe/liburing/blob/master/test/link-timeout.c) 测试代码里找到。

### Cancel
这其实是一个疑点记录，io_uring 可以通过 fd 或者 user_data 作为 key 来 cancel 请求，其[文档](https://man7.org/linux/man-pages/man3/io_uring_prep_cancel.3.html#top_of_page)描述：

> Although the cancelation request uses async request syntax, the
> kernel side of the cancelation is always run synchronously. It is
> guaranteed that a CQE is always generated by the time the cancel
> request has been submitted. If the cancelation is successful, the
> completion for the request targeted for cancelation will have
> been posted by the time submission returns. For -EALREADY it may
> take a bit of time to do so. For this case, the caller must wait
> for the canceled request to post its completion event.

即 cancel 被 submit 的时候就会被执行，保证 submit 完 CQ 里就会有 cancel 成功与否的结果。如果事件被成功 cancel，CQ 里也会有这些事件的 CQE。如果 cancel 返回 -EALREADY，则说明其他事件的 CQE 还没生成，但是已经在 cancel 中了。

疑点就在于，文档似乎没有说明如果一个事件已经在 CQ 里但还未被消费，此时 cancel 会不会影响这个 CQE，这决定了我能否认为 cancel 之后再消费 CQ 一定不会消费到 cancel 的事件，进而影响了用户空间 user_data 的生命周期管理。

## 总结
说了那么多，尝试用 io_uring 写了 TCP echo 吞吐测试，发现在连接数较低或者默认模式下，io_uring 性能不如 epoll，只有在连接数较多且配合 defer taskrun 的情况下，io_uring 才有优势。用 multishot 也有少量收益，但不如 defer taskrun 大，而且必须配合 provide buffer 也增加了使用难度。

在实际项目里，用 io_uring，结合 defer taskrun 并支持 fast poll，不启用 multishot 和 provide buffer 的情景下，性能比 epoll 差 :/
