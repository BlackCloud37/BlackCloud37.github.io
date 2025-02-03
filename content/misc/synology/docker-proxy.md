---
title: synology docker proxy
date: 2025-02-03
taxonomies:
  categories: [selfhost]
  tags: [synology, docker]
---

DSM 7.2.2，配置 docker pull 走代理

参考：[【分享】群辉Docker pull代理设置方法](https://linux.do/t/topic/109710)，只是配置文件位置不一样

```bash
vim /usr/local/lib/systemd/system/pkg-ContainerManager-dockerd.service
```

`HTTP(S)_PROXY` 改成自己的代理地址，`NO_PROXY` 不用动
```
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:1080"
Environment="HTTPS_PROXY=http://127.0.0.1:1080"
Environment="NO_PROXY=localhost,127.0.0.0/8,192.168.0.0/16,172.16.0.0/12,10.0.0.0/8"
```

重启服务
```bash
sudo synosystemctl restart pkg-ContainerManager
```
