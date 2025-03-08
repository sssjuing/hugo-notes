---
title: WireGuard
date: "2025-03-08"
---

---

### docker-compose.yml

```yaml
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE #optional
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - SERVERURL=120.46.220.100 #optional
      - SERVERPORT=51820 #optional
      - PEERS=openwrt,routeropenwrt,client01,client02,client03,client04 #optional
      - PEERDNS=auto #optional
      - INTERNAL_SUBNET=10.8.0.0 #optional
      - PERSISTENTKEEPALIVE_PEERS=openwrt
      - SERVER_ALLOWEDIPS_PEER_openwrt=10.9.2.0/24
      - SERVER_ALLOWEDIPS_PEER_routeropenwrt=10.9.22.0/24
    volumes:
      - ./config:/config
      - /lib/modules:/lib/modules #optional
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

如果将 OpenWrt 作为旁路由使用, 还需在 `防火墙` 的 `自定义规则` 中, 添加以下 iptables 命令:

```sh
# 注意此条防火墙网段 192.168.100.0/24 需和上文服务端 IP 网段保持一致.
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o br-lan -j MASQUERADE
```
