# wolfSSL 使用

## 背景

c++ 写 TLS 客户端，openssl 实在是有些丑，另外也希望优化加解密性能，遂研究了一下 [wolfSSL](https://www.wolfssl.com/) 的使用，因为看它：
- [文档](https://www.wolfssl.com/documentation/manuals/wolfssl/index.html)齐全
- 有个[支持社区](https://www.wolfssl.com/forums/)且上面有人回答
- 据说性能比 openssl 好，参见 [benchmark](https://www.wolfssl.com/docs/benchmarks/)

## 安装及编译选项

### 源码拉取

官网有个 [Download](https://www.wolfssl.com/download/) 页面，可以下载源码，其 [github release](https://github.com/wolfSSL/wolfssl/releases/tag/v5.7.4-stable) 页也能拉源码，二者在 `configure` 时略有不同：前者可以直接 `configure`，后者需要先 `./autogen.sh`

以后者为例

```bash
wget https://github.com/wolfSSL/wolfssl/archive/refs/tags/v5.7.4-stable.tar.gz -O wolfssl-5.7.4-stable.tar.gz
tar -xf wolfssl-5.7.4-stable.tar.gz
cd wolfssl-5.7.4-stable && ./autogen.sh
```

### 编译

大概是因为 wolfSSL 在设计之初是考虑给嵌入式设备用的，支持非常多的编译选项，所以非常可定制，当然也会导致刚开始用的时候也会比较 confusing，所有选项参考 [Building wolfSSL](https://www.wolfssl.com/documentation/manuals/wolfssl/chapter02.html)

根据我的用例，我选择了如下选项，它们的作用分别是
- `--enable-tls13` 启用 tls1.3
- `--disable-harden` (性能)禁用安全强化
- `--enable-intelasm` (性能)启用 intel 指令集加速，在我的 intel 机器上测试性能改进很显著
- `--enable-aesni --enable-sp --enable-sp-asm` (性能)启用各种加速，不过这些在我的测试中并没有带来显著提升
- `--enable-singlethreaded` (性能)如果能保证进程不会并发访问 wolfssl，可以启用
- `--enable-ed25519` 因为用到了 ed25519 相关功能
- `--enable-opensslall` 因为旧的代码是 openssl 写的，启用这个后会暴露非常多 openssl 的兼容接口，基本上可以无缝迁移
    - `--enable-opensslextra` 我没有启用这个选项，但是提一嘴，因为它实际上是暴露比 `opensslall` 更多的兼容接口(`opensslall` 虽然叫 all 但不是它的超集)
- `CFLAGS="-DLARGE_STATIC_BUFFERS"` 启用这个选项后可以减少 `malloc`，参考 [Library Design](https://www.wolfssl.com/documentation/manuals/wolfssl/chapter09.html) 的 Input and Output Buffers 章节
- `--libdir=/usr/local/lib64` 指定安装路径

上面的选项里，性能相关的选项，除了 `enable-intelasm` 测试后有明显提升，其他选项均是看文档写了可能提升 performance 才启用的，但实际测试改进可能不显著，建议自己写个 benchmark 然后每启用一个选项测试一遍

另外我只关心 TLS Application 相关的性能，即握手完成后对称加解密的过程，所以只测了这个

完整的 configure 命令如下，包含了编译安装
```bash
./configure --enable-tls13 \
    --disable-harden \
    --enable-intelasm \      
    --enable-aesni \
    --enable-sp --enable-sp-asm \
    --enable-singlethreaded \
    --enable-opensslall \
    --enable-ed25519 \
    --libdir=/usr/local/lib64 \
    CFLAGS="-DLARGE_STATIC_BUFFERS"

make -j8 && make install
```

### 使用

参考 [wolfssl-examples](https://github.com/wolfSSL/wolfssl-examples/tree/master)，用例非常全

在我的使用场景中，基本上只要把 openssl 头文件换成
```c
#include <wolfssl/options.h> // 必须在所有 wolfssl include 之前
#include <wolfssl/ssl.h>
```

然后编译链接时把 `-lcrypto -lssl` 换成 `-lwolfssl` 即可，由于启用了 `--enable-opensslall`，所以基本上绝大多数 openssl 的 symbol 都可以直接使用，其会被替换成 wolfssl 的，例如
```c
#define SSL_CTX WOLFSSL_CTX
#define SSL_new wolfSSL_new
```

#### I/O Callback

原本的 openssl 的代码里，I/O 用了 BIO，在启用 `--enable-opensslall` 后，wolfSSL 也提供 BIO 接口，也是可以无缝迁移的

但是看文档发现它还提供了 I/O Callback 接口，参考 [Portability](https://www.wolfssl.com/documentation/manuals/wolfssl/chapter05.html) 的 Custom Input/Output Abstraction Layer 章节

此外，在看源码时发现，其所有 I/O 都是以 I/O Callback 实现的，例如设置 BIO 其实只是设置了 BIO callback，在需要读写时将数据写到 BIO 层。直接用 callback 相比 BIO 应该少了一次 memcpy，出于性能的考量打算使用这个接口

使用也比较简单，可以参考 [wolfssl-examples/custom-io-callbacks](https://github.com/wolfSSL/wolfssl-examples/blob/master/custom-io-callbacks)，它实现了通过文件而非 socket 作为 I/O 进行 SSL 通信的例子

两个 callback 的定义如下
```c
typedef int (*CallbackIORecv)(WOLFSSL *ssl, char *buf, int sz, void *ctx);
typedef int (*CallbackIOSend)(WOLFSSL *ssl, char *buf, int sz, void *ctx);
```

这两个 callback 的语义是
- CallbackIORecv: 当 SSL 希望读取数据时，会调这个 cb
    - 其中 buf 是 SSL 内部的 buffer，sz 是 SSL 希望读取的字节数，我们需要将读到的指定字节数写入 buf
    - 返回实际读到的字节数或者 `WOLFSSL_CBIO_ERR_WANT_READ` 表示暂无数据
    - 例如，在 `SSL_connect` 之后，或者调用 `SSL_read` 时，即 ssl 希望读取控制消息或者 application data 时，会调用这个 cb
    - 在握手完成后的 `SSL_read` 过程中，wolfSSL 通常会先调一次 sz 为 5 的 cb 来试图读取消息头，然后再根据消息头的数据长度来读剩下的 payload
- CallbackIOSend: 当 SSL 希望发送数据时，会调这个 cb
    - 其中 buf 是 SSL 希望发出的数据(加密后)
    - 返回实际发送的字节数或者 WOLFSSL_CBIO_ERR_WANT_WRITE 表示需要重试即可
    - 例如，在调用 `SSL_connect` 之后，或者调用 `SSL_write` 时，即 ssl 希望发送控制消息或者加密后的 application data 时，会调用这个 cb

`ctx` 则是用户自己设置的 userdata

在实现了自己的 callback 后，通过如下代码设置即可
```cpp
wolfSSL_SSLSetIORecv(ssl, CBIORecv);
wolfSSL_SSLSetIOSend(ssl, CBIOSend);
// wolfSSL_SetIOReadCtx(ssl, userdata);
// wolfSSL_SetIOWriteCtx(ssl, userdata);
```

#### 减少 malloc

wolfSSL 支持自定义 allocator(malloc, free, realloc)，参考 [Portability](https://www.wolfssl.com/documentation/manuals/wolfssl/chapter05.html) 的 Memory Use 章节

```c
int wolfSSL_SetAllocators(wolfSSL_Malloc_cb  malloc_function,
                         wolfSSL_Free_cb    free_function,
                         wolfSSL_Realloc_cb realloc_function);
```

也因此，我尝试用一个包含调用计数的 malloc 来观察 malloc 的次数，然后发现在不启用 `LARGE_STATIC_BUFFERS` 的情况下，除了握手阶段以外，每次读或者写 SSL 都会出现一次 malloc(和相应的 free)
- 相关讨论 [Best practice to avoid dynamic memory allocations](https://www.wolfssl.com/forums/post8115.html)

如果启用 `LARGE_STATIC_BUFFERS`，每个 `SSL` 都会有一个固定大小的 `staticBuffer`，其大小应该是 `MAX_RECORD_SIZE`，即 16KB，在读写数据时只要这个 buffer 够用，就不会出现 malloc

另一个方法则复杂一些，参考 [Features](https://www.wolfssl.com/documentation/manuals/wolfssl/chapter04.html#enabling-static-buffer-allocation) 的 Static Buffer Allocation Option 章节，大概流程是：
- 在 `configure` 时启用 `--enable-staticmemory`
- 预先分配两个内存区域，并传递给 wolfSSL，需要调用两遍 `wolfSSL_CTX_load_static_memory`
  - 下面这个 example 是 [Features](https://www.wolfssl.com/documentation/manuals/wolfssl/chapter04.html#enabling-static-buffer-allocation) 里抄的，但 `WOLFMEM_IO_FIXED` 应该改成 `WOLFMEM_IO_POOL_FIXED`
    ```c
    WOLFSSL_CTX* ctx = NULL; /* pass NULL to generate WOLFSSL_CTX */
    int ret;

    #define MAX_CONCURRENT_TLS  0
    #define MAX_CONCURRENT_IO   0

    unsigned char GEN_MEM[GEN_MEM_SIZE];
    unsigned char IO_MEM[IO_MEM_SIZE];　

    /* set up a general-purpose buffer and generate WOLFSSL_CTX from it on the first call. */
    ret = wolfSSL_CTX_load_static_memory(
            &ctx,                               /* set NULL to ctx */
            wolfSSLv23_client_method_ex(),  /* use function with "_ex" */
            GEN_MEM, GEN_MEM_SIZE,            /* buffer and its size */
            WOLFMEM_GENERAL,                  /* general purpose */
            MAX_CONCURRENT_TLS);              /* max concurrent objects */

    /* set up a I/O-purpose buffer on the second call. */
    ret = wolfSSL_CTX_load_static_memory(
            &ctx,                /* make sure ctx is holding the object */
            NULL,                           /* pass it to NULL this time */
            IO_MEM, IO_MEM_SIZE,                /* buffer and its size */
            WOLFMEM_IO_FIXED,                             /* I/O purpose */
            MAX_CONCURRENT_IO);               /* max concurrent objects */

    if (ret != 0)
    {
        auto error = wolfSSL_ERR_get_error();
        if (error != WOLFSSL_ERROR_NONE)
        {
            char error_str[256];
            wolfSSL_ERR_error_string(error, error_str);
            throw std::runtime_error("wolfSSL_CTX_load_static_memory error: " + std::string(error_str));
        }
    }
    ```
- 之后的使用应该不需要修改，在 `statcmemory` 够大的情况下，wolfSSL 会自动在上面拿内存，本质上就是实现了一个预分配的 malloc
  - 如果内存不够，可能会回退到 malloc，好像可以通过 `WOLFSSL_NO_MALLOC` 来强制禁止，参考 [Doing Crypto Without Malloc’s](https://www.wolfssl.com/crypto-without-mallocs/)

不过其实 `staticmemory` 本质上还是要每次都分配内存(只是在预分配的区域上分配)，相比 `LARGE_STATIC_BUFFERS` 在理想情况下可以完全不分配内存，后者应该更符合我的需求

在我的测试里，在收发数据包较小(因此单次 malloc 成本很低)的情况下，性能表现 `LARGE_STATIC_BUFFERS` > `malloc` > `staticmemory`

## Benchmark

简单测试了 TLS Client 端，在 TLS1.3 用 `TLS_AES_128_GCM_SHA256` cipher 的情况下，连续收发 256B payload，加解密的耗时(ns)

openssl 作为 baseline, 在使用 BIO 的情况下
```
encryption_costs: min: 357, max: 11197, avg: 388, p50: 370, p99: 546
decryption_costs: min: 316, max: 16816, avg: 350, p50: 330, p99: 532
```

wolfSSL 不启用 `intelasm` 且使用 BIO
```
encryption_costs: min: 1074, max: 16198, avg: 1132, p50: 1098, p99: 1335
decryption_costs: min: 1143, max: 16839, avg: 1205, p50: 1169, p99: 1427
```

启用 `--disable-harden`，没有观察到提升
```
encryption_costs: min: 1067, max: 11969, avg: 1127, p50: 1092, p99: 1338
decryption_costs: min: 1139, max: 21007, avg: 1206, p50: 1168, p99: 1433
```

启用 `--enable-intelasm`，性能提升显著
```
encryption_costs: min: 200, max: 10915, avg: 228, p50: 214, p99: 304
decryption_costs: min: 263, max: 8928, avg: 309, p50: 296, p99: 407
```

启用 `--enable-sp-asm`，没有观察到提升
```
encryption_costs: min: 205, max: 16980, avg: 244, p50: 235, p99: 319
decryption_costs: min: 258, max: 15187, avg: 300, p50: 286, p99: 388
```

启用 `--enable-singlethreaded`，提升不大，但似乎是有些提升
```
encryption_costs: min: 211, max: 10392, avg: 249, p50: 240, p99: 331
decryption_costs: min: 238, max: 12050, avg: 281, p50: 261, p99: 373
```

在发送方向上用 Callback，接收方向上仍然用 BIO，且启用 `LARGE_STATIC_BUFFERS`，提升较大
```
encryption_costs: min: 124, max: 10720, avg: 131, p50: 131, p99: 138
decryption_costs: min: 172, max: 11071, avg: 184, p50: 183, p99: 208
```

再启用 `fast-math` 和 `fast-huge-math`，没有观察到提升
```
encryption_costs: min: 124, max: 17766, avg: 132, p50: 131, p99: 147
decryption_costs: min: 172, max: 16989, avg: 184, p50: 183, p99: 208
```

总结就是使用 IOCallback，启用 `LARGE_STATIC_BUFFERS`，启用 `intelasm` 和 `singlethreaded`(如果确认不会多线程使用)，能达到比较好的加解密性能，比 openssl 快

