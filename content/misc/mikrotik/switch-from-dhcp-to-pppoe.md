---
title: "mikrotik 从 dhcp 切换到 pppoe"
date: 2025-07-02
taxonomies:
  categories: ["selfhost"]
  tags: ["mikrotik"]
---

把家里的宽带换了，从联通 fttr 改成了非 fttr，可以改桥接。因此需要从 dhcp 切换到 pppoe。

步骤要点记录如下：

1. 新建一个 pppoe interface (Interfaces -> Interface -> Add New -> PPPoE Client)
    - 拨号成功的话应该可以拿到 IP
2. 禁用原来的 DHCP Client (IP -> DHCP Client)
3. 修改之前的的防火墙规则，将原本的 wan interface 改成 pppoe interface，包括但不限于
    - filter rules
    - nat
        - masquerade
        - dst-nat
