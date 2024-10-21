# M2 SSD 作为存储盘

使用 007revad 的脚本
- [Synology_M2_volume](https://github.com/007revad/Synology_M2_volume)
- [Synology_HDD_db](https://github.com/007revad/Synology_HDD_db)

在 ds918+, DSM 7.2 上测试

1. 先跑 `Synology_M2_volume` 脚本, 跑完就可以直接建存储池了, 不用重启
2. 然后跑 `Synology_HDD_db` 脚本, 用的 `-nr` flag
3. 在计划任务里加一个 开机触发、root 运行的计划, 内容为 `/path-to-script/syno_hdd_db.sh -nr`
4. 加一个备份任务, 定期把 SSD 的数据备份到 HDD 盘, 免得突然挂了
