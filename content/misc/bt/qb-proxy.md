---
title: "用公网服务器代理 qBittorrent 的上传"
date: 2025-02-16
taxonomies:
  categories: ["selfhost"]
  tags: ["qbittorrent"]
---

思考用 qb 做种时，本地没有公网 IP 导致上传很慢的情景，能否通过公网服务器代理上传。

## 思路

1. qb 使用公网服务器作为[代理](#proxy)，向 peer 通告公网服务器的 IP
2. 公网服务器设置一个[端口转发](#port-forward)，将 peer 的连接转发到本地

## 可行性验证

### 1. 代理 {#proxy}

首先在本地的 8088 端口开一个走公网服务器的 socks5 代理：

```bash
ssh -D 0.0.0.0:8088 -N -f user@your_server
```

在 qb 中设置代理：socks5, 127.0.0.1:8088

代理下面的三个选项中(这里参考了 [qBittorrent反代解决NAT连通性](https://cherr.cc/qb_proxy))

勾选：
- 使用代理服务器进行主机名查询
- 只对 torrent 使用代理
    - 这个选项决定上报到 tracker 的 IP

不勾选：
- 使用代理服务器进行用户连接
    - 这个选项会导致下载也走代理。另外，上报的 port 会变成 1，导致 peer 无法连接

此外，还需要在 qb 高级设置里勾选 允许来自同一 IP 地址的多个连接，因为在本机看来所有 peer 都是公网服务器的 IP。

这样，qb 就会上报公网服务器的 IP 了，但是 peer 还不能从公网服务器连接到我们。

### 2. 端口转发 {#port-forward}

我比较习惯用 firewalld + tailscale 来端口转发，假设要将公网服务器的 60123 端口转发到本地 60123 端口：

安装 tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

然后在本机和公网服务器上都认证并开启 tailscale，此时二者应该已经在 ts 的局域网中，并且可以 DIRECT 互访。假设本机的 ts IP 是 TAILSCALE_IP_LOCAL。

然后在服务器上设置端口转发：
```bash
TAILSCALE_IP_LOCAL=100.100.100.100
PORT=60123
# 打开 60123 端口
firewall-cmd --add-port=${PORT}/tcp --add-port=${PORT}/udp --permanent
# 将 60123 端口转发到 TAILSCALE_IP_LOCAL:60123
firewall-cmd --zone=public --add-forward-port=port=${PORT}:proto=tcp:toaddr=${TAILSCALE_IP_LOCAL}:${PORT} --permanent
firewall-cmd --zone=public --add-forward-port=port=${PORT}:proto=udp:toaddr=${TAILSCALE_IP_LOCAL}:${PORT} --permanent
# 启用 masquerade
firewall-cmd --zone=public --add-masquerade --permanent
# 重启防火墙
firewall-cmd --reload
```

如果服务器有安全组规则，记得打开对应的端口。

可以通过在本机 `nc -l ${TAILSCALE_IP_LOCAL} ${PORT}`，然后再在本机 `nc ${SERVER_IP} ${PORT}` 检查是否能连接来判断端口转发是否正确设置。

如果正确设置，那么之后 peer 就会通过公网服务器连接到我们，也就完成了整个代理过程。
