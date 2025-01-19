---
title: "mikrotik 端口转发保留 srcip"
date: 2024-11-01
taxonomies:
  categories: ["selfhost"]
  tags: ["mikrotik"]
---

为了暴露家里的服务，做了 9443 上的端口转发，在设置访问规则时，发现即使设置了类似 LAN 免密这样的规则，从外网访问也会命中

应该是因为 dst-nat 配置错误导致 src-ip 被改成了网关 ip

参考 [NAT IP don't show real visitors IP](https://forum.mikrotik.com/viewtopic.php?t=158480)，将 masquerade 的 out-interface 设置为 wan 后解决
