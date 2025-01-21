---
title: synology btrfs 修复
date: 2024-10-21
taxonomies:
  categories: [selfhost]
  tags: [synology, bug, btrfs]
---

买的二手 DS918+，两条内存中有一条坏，由于开机时没 memtest 且能点亮，迁数据过程中一直没发现问题

中途报了个 checksum mismatch，还不以为然

然后某次重启之后一个存储池变成 Read-Only 了，但其实此时访问那个卷的数据都会 IO 错误

查资料看到 [Btrfs修复](https://zyyme.com/btrfs-fix.html)

照着操作一遍之后重启就能用了，不过后续又报了几个 checksum mismatch，相关的照片确实也损坏了
