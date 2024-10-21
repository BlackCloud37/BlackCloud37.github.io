# qBittorrent

用 docker 安装, docker compose 如下

```yaml
services:
  qbittorrent:
    image: linuxserver/qbittorrent
    network_mode: host
    container_name: qbittorrent
    environment:
      - WEBUI_PORT=10095    # Customize if needed
      - PUID=          # Change to your user ID
      - PGID=          # Change to your group ID
      - TZ=Asia/Shanghai    # Change to your timezone
    volumes:
      - /volume4/AppData/qBittorrent:/config/qBittorrent # 配置目录
      - /volume2/Download:/downloads # 数据目录
    restart: unless-stopped
```

如果有旧的种子信息需要导入, 直接把旧的 qBittorrent 配置目录复制过来就行, 主要是其中的 BT_Backup 文件夹