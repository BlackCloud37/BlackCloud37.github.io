# Plex

记录装 plex 的注意点，参考 [群晖 Docker 安装 qBittorrent (DSM 7.2)](https://blog.mitsea.com/f34d4a3f981846469cac041c1df0881d/)

1. 加一个计划任务、开机时以 root 运行 `chmod 666 /dev/dri/*`
2. docker 安装 plex，参考 [Plex in container manager on a Synology NAS hardware transcoding](https://drfrankenstein.co.uk/plex-in-container-manager-on-a-synology-nas-hardware-transcoding/)
3. 我的 docker compose 如下，为 plex 单独创建了一个用户

```yaml
services:
  plex:
    image: linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID= #CHANGE_TO_YOUR_UID
      - PGID= #CHANGE_TO_YOUR_GID
      - TZ=Asia/Shanghai #CHANGE_TO_YOUR_TZ
      - UMASK=022
      - VERSION=docker
      - PLEX_CLAIM= #Your Plex Claim Code
    volumes:
      - /volume3/docker/plex:/config
      - /volume2/Movie:/media/Movie
      - /volume2/Series:/media/Series
      - /volume2/Music:/media/Music
      - /volume2/Video:/media/Video
      - /volume2/Porn:/media/Porn
      - /volume2/Anime:/media/Anime
    devices:
      - /dev/dri:/dev/dri
    security_opt:
      - no-new-privileges:true
    restart: always
```

服务启动后可以禁用一下 relay
