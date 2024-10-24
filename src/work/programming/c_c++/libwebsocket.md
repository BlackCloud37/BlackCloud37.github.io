# libwebsocket server 开发的坑

需要用 lws 写一个 server，踩了一些坑，记录如下。

1. 自定义的 user* 在任何情况下都不会由 lws 构造销毁，如果在里面放容器等依赖构造析构的类，需要自己初始化
2. 官方文档说不允许在 write callback 以外的场景写入数据。但似乎先检查 `lws_send_pipe_choked()` 再写入是没问题的，如果写不了了再缓存
3. 回调里的 user* 可能是 nullptr，取决于回调对应的连接状态

