---
title: "Synology 系统无响应[pending]"
date: 2025-01-21
taxonomies:
  categories: [selfhost]
  tags: [synology, bug]
---

记录一次群晖 debug 的经历

## 背景

DS918+，DSM7，装了一块 ZHITAI 的 2T NVME SSD，通过 [Synology_M2_volume](https://github.com/007revad/Synology_M2_volume) 挂载为存储盘，参考 [synology-m2-ssd](../m2-ssd)。

在连续运行一段时间后，系统偶尔会进入如下**状态**：
- 机器物理上还在运行
  - 风扇转，硬盘转，status led 常绿，WAN 口指示灯绿，等等
- 系统无任何响应
  - ping host is down
  - ssh 无法连接
  - webui 无法访问
  - 路由器里 NAS 的 lease 都消失了

发生原因不明，一般都是在没有什么操作/使用的状态下，突然发现访问不了了。

发生之后只能长按电源键直到强制关机，然后重启。

重启之后 DSM 层面没有错误日志，硬盘状态也都是正常。

## (可能的)原因 {#possible-reason}

因为没法复现，只能先认为是这个原因，如果能稳定一两个月不复现，就认为解决。

看 kernel log 发现在系统无响应时，有如下错误：
```
./dsm/var/log/kern.log:2025-01-19T00:02:56+08:00 Ship kernel: [992391.293034] nvme nvme0: I/O 262 QID 1 timeout, aborting
./dsm/var/log/kern.log:2025-01-19T00:02:56+08:00 Ship kernel: [992391.299025] nvme nvme0: I/O 263 QID 1 timeout, aborting
./dsm/var/log/kern.log:2025-01-19T00:02:56+08:00 Ship kernel: [992391.305010] nvme nvme0: I/O 264 QID 1 timeout, aborting
./dsm/var/log/kern.log:2025-01-19T00:02:57+08:00 Ship kernel: [992392.277052] nvme nvme0: I/O 13 QID 0 timeout, reset controller
./dsm/var/log/kern.log:2025-01-19T00:03:57+08:00 Ship kernel: [992452.286778] nvme nvme0: I/O 262 QID 1 timeout, reset controller
./dsm/var/log/kern.log:2025-01-19T00:05:35+08:00 Ship kernel: [992549.781624] INFO: task md5_raid1:9304 blocked for more than 120 seconds.
```

搜索 `NVME QID timeout`，几个帖子[[1](https://bbs.archlinux.org/viewtopic.php?id=286274), [2](https://forum.proxmox.com/threads/problem-with-nvme-timeout-and-aborting.144492/)]都指向一个原因：
[arch wiki ssd/nvme - troubleshooting](https://wiki.archlinux.org/title/Solid_state_drive/NVMe#Troubleshooting)。且表现基本相同：报错后系统挂起，直到重启为止。

解决方案也在[其中](https://wiki.archlinux.org/title/Solid_state_drive/NVMe#Troubleshooting)：
- 添加内核参数 `nvme_core.default_ps_max_latency_us=0`

在 DSM7 里，似乎没有 `nvme_core`，但是可以看到有 `/sys/module/nvme/parameters/default_ps_max_latency_us`，这个 parameter，猜测是一样的

热修改：
```bash
echo 0 | sudo tee /sys/module/nvme/parameters/default_ps_max_latency_us
```

持久化，本来是应该改 grub 的，但不确定群晖让不让改，于是加到了系统启动脚本里：
- 控制面板，任务计划，新增，触发的任务，用户定义的脚本
- 用户账号 root，事件 开机
- 脚本
  ```bash
  echo "0" > /sys/module/nvme/parameters/default_ps_max_latency_us
  ```

## Debug 历程

第一次发现(2024-12-12)，以为是群晖系统问题，但开机后没找到什么日志，于是没有管。

第二次发现(2025-01-05)，因为以为是群晖系统问题，拍下机器当时的表现提了个工单到群晖支持(还要下载 debug log 一并上传)，工程师回复还挺快的，回复：
1. 能看到 improper shutdown(强制关机) 和一些 hung task，但由于需要开启 定時记录系统运行状态 这个选项才有更详细的信息，所以暂时没法确认原因(此时我刚打开该选项)
2. 建议做三次 memtest，排除内存问题

此时确实怀疑是不是内存问题，因为用了一条 kingston 的内存，但是当初装上去的时候 memtest 是通过的(因为坏的内存条丢数据，参考 [synology-btrfs-repair](../btrfs-repair))。不过做了三次 memtest，都通过了。只好等下次再复现的时候把详细日志给工程师。

第三次发现(2025-01-18)，在看 nas 上电影的时候又 hung 了，这次工程师的回复是：
> 經過檢查日誌，我們發現 NVMe 裝置（序號：ZTA22T0KA2245509Z1）已達到 100% 的使用率，並發生了 I/O timeout error。這可能導致系統無法取得必要資源，進而引發系統卡住的問題。
> <br><br>
> 此外，DS918+ 並不支援使用 M.2 NVMe 裝置來建立 Storage Pool（僅支援此清單中列出的裝置），僅能用來建立 SSD 快取。我們不確定您是如何使用 NVMe 裝置建立 Storage Pool1/Volume1。為確保系統穩定性，我們建議您備份 Volume 1 上的所有資料，並使用 SATA HDD/SSD 重新建立 Storage Pool1/Volume1。另外，請選擇此清單中列出的相容硬碟 ，以確保資料完整性。

第二段话让我有种作弊被抓的感觉😥，不过确实提醒我是 I/O timeout error 的问题，于是去看这次下载下来的日志：
- 下载下来的日志是 `.dat` 文件，其实是个 zip，unzip 一下就好
- 里面包含很多日志，由于我知道问题发生的大概时间是 2025-01-19 凌晨那会，于是 `grep -P "2025-01-19\w+00:[01].*timeout" -r .`，就找到了[上面](#possible-reason)的错误日志

