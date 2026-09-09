---
title: "Caddy-L4：Reality SNI 分流与自动 HTTPS 部署指南"
description: "在单台 VPS 的公网 443 端口上，通过 Caddy-L4 按 TLS SNI 分流 Reality 至 sing-box，同时为 Web/API 提供自动 HTTPS、HTTP/2、HTTP/3 与 Docker 容器反代。"
publishDate: "2026-09-08"
tags: ["Caddy", "Reality", "sing-box", "Docker", "HTTPS", "VPS"]
---

# Caddy-L4：Reality SNI 分流部署指南

## 目标

在一台 VPS 上只使用公网 `443` 作为统一入口：

- Reality 流量按 TLS SNI 在四层透传给本机 sing-box。
- 普通 Web/API 域名由 Caddy 终止 TLS，并自动申请、续期 ACME 证书。
- 普通 Web/API 同时支持 HTTP/1.1、HTTP/2 与 HTTP/3。
- Caddy 的证书、私钥、ACME 账户等数据持久化到宿主机。

```text
公网 TCP 443
  ├─ Reality SNI
  │    └─ Caddy-L4：读取 ClientHello，不终止 TLS
  │         └─ 宿主机 sing-box:8443
  │
  └─ Web/API 域名
       └─ Caddy：TLS termination + ACME + reverse_proxy
            └─ Docker 内网应用

公网 UDP 443
  └─ Caddy HTTP/3（仅 Web/API；Reality 不使用 UDP）
```

---

## 前提条件

1. Caddy 与需要反代的 Web/API 容器位于同一个 Docker 网络。
2. sing-box 作为宿主机服务监听某个 TCP 端口，例如 `8443`。
3. Web/API 域名的 DNS A/AAAA 记录正确指向 VPS。
4. 防火墙允许公网 TCP 443 和 UDP 443；TCP 80 可按需保留。
5. Reality 使用的 SNI 不得与 Caddy 托管的 Web/API 域名重叠。

检查宿主机 sing-box 是否监听：

```bash
ss -lntp | grep ':8443'
```

检查容器能否访问宿主机 sing-box：

```bash
docker run --rm \
  --network <docker_network> \
  --add-host=host.docker.internal:host-gateway \
  alpine:3.20 \
  sh -c 'apk add --no-cache busybox-extras >/dev/null && nc -vz host.docker.internal 8443'
```

预期包含：

```text
host.docker.internal (...:8443) open
```

---

## Docker Compose

以下示例使用已包含 Caddy-L4 模块的镜像。`host.docker.internal:host-gateway` 用于让 Linux Docker 容器访问宿主机上的 sing-box 服务。

```yaml
services:
  caddy:
    image: livekit/caddyl4:latest
    container_name: caddy
    restart: unless-stopped

    ports:
      - "80:80/tcp"
      - "443:443/tcp"
      - "443:443/udp"

    volumes:
      - /opt/apps/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - /opt/apps/caddy/data:/data
      - /opt/apps/caddy/config:/config

    command:
      - run
      - --config
      - /etc/caddy/Caddyfile
      - --adapter
      - caddyfile

    extra_hosts:
      - "host.docker.internal:host-gateway"

    networks:
      - app_network

networks:
  app_network:
    external: true
```

说明：

- `443/tcp` 用于 Reality、HTTPS、HTTP/2。
- `443/udp` 用于 HTTP/3/QUIC；开启 HTTP/3 时必须存在。
- `/data` 必须持久化；它保存证书、私钥、ACME 账户、OCSP 和锁文件。
- Caddyfile 保持 `:ro` 是正确的。运行容器只读配置；编辑和格式化应由宿主机或临时容器完成。

---

## Caddyfile 模板

将 `<REALITY_SNI>`、`<SINGBOX_PORT>`、`<WEB_DOMAIN>`、`<WEB_SERVICE>` 和 `<WEB_PORT>` 替换为实际值。

```caddyfile
{
    storage file_system {
        root /data
    }

    servers :443 {
        protocols h1 h2 h3

        listener_wrappers {
            layer4 {
                @reality tls sni <REALITY_SNI>

                route @reality {
                    proxy host.docker.internal:<SINGBOX_PORT>
                }
            }

            tls
        }
    }
}

<WEB_DOMAIN> {
    reverse_proxy <WEB_SERVICE>:<WEB_PORT>
}
```

<p align="center">
  <img src="https://image.xmlys.de/file/1788954711721_2942F2CE-D476-4BD1-9C41-6D1C665FE592.png" alt="Caddy-L4 Reality SNI 分流与自动 HTTPS 架构示意图" style="display:block;width:100%;max-width:900px;height:auto;margin:1.5rem auto;border-radius:8px;" />
</p>

### 配置要点

- `storage file_system { root /data }` 是 Caddy 的证书/ACME 数据存储路径，不是静态网站目录。
- `layer4` 必须排在 `tls` 前面。这样 Reality 连接才会在 Caddy TLS 握手前被识别并透传。
- 匹配 `<REALITY_SNI>` 的连接只进行 TCP proxy；Caddy 不会给该 SNI 签证书。
- 未匹配 Reality 的 TLS 连接会继续进入 `tls` wrapper，由 Caddy 为 `<WEB_DOMAIN>` 自动管理 HTTPS。
- `protocols h1 h2 h3` 明确启用 HTTP/1.1、HTTP/2、HTTP/3。若不希望开启 HTTP/3，应同时删除 UDP 443 映射，并改为 `protocols h1 h2`。

---

## 部署与更新流程

### 1. 格式化 Caddyfile

由于运行容器将 `/etc/caddy/Caddyfile` 挂载为只读，不能在运行容器内执行 `caddy fmt --overwrite`。

正确做法是启动临时容器，以可写 bind mount 格式化宿主机文件：

```bash
docker run --rm \
  -v /opt/apps/caddy/Caddyfile:/etc/caddy/Caddyfile \
  livekit/caddyl4:latest \
  fmt --overwrite /etc/caddy/Caddyfile
```

### 2. 验证配置

任何改动后先验证：

```bash
docker run --rm \
  -v /opt/apps/caddy/Caddyfile:/etc/caddy/Caddyfile:ro \
  -v /opt/apps/caddy/data:/data \
  livekit/caddyl4:latest \
  adapt \
  --config /etc/caddy/Caddyfile \
  --adapter caddyfile \
  --validate
```

HTTP/3 已启用时，输出 JSON 应包含：

```json
"protocols":["h1","h2","h3"]
```

### 3. 应用变更

仅修改 Caddyfile 时，热加载即可：

```bash
docker exec caddy caddy reload \
  --config /etc/caddy/Caddyfile \
  --adapter caddyfile
```

如果修改了 Compose 中的端口、镜像、挂载或网络，必须重建容器：

```bash
# 进入实际保存 compose.yaml 的目录
docker compose up -d --force-recreate caddy
```

### 4. 查看运行状态

```bash
docker logs --tail 100 caddy
docker ps --filter name=caddy --format 'table {{.Names}}\t{{.Ports}}'
```

启用 HTTP/3 后，日志应包含：

```text
enabling HTTP/3 listener
server running ... protocols:["h1","h2","h3"]
```

---

## 验证清单

### Web/API HTTPS

```bash
curl -Iv https://<WEB_DOMAIN>
```

应确认：

- 证书 Subject/SAN 包含 `<WEB_DOMAIN>`。
- 签发者是受信任 CA，例如 Let's Encrypt。
- TLS 验证成功。
- 响应来自后端应用；业务接口返回 404 不等于 Caddy 反代失败。真正的后端不可达通常表现为 `502 Bad Gateway`。

### HTTP/3

```bash
curl -sI https://<WEB_DOMAIN> | grep -i alt-svc
```

启用 HTTP/3 后，通常应出现：

```text
alt-svc: h3=":443"; ...
```

如果本机 curl 带有 HTTP/3 支持：

```bash
curl --http3-only -Iv https://<WEB_DOMAIN>
```

关键是响应协议显示为 `HTTP/3`。若失败，依次检查 Docker 是否发布 UDP 443、VPS 防火墙、云防火墙及 DNS IPv6 路径。

### Caddy storage

```bash
docker exec caddy sh -c 'find /data -type f | sort'
```

应能看到：

```text
/data/acme/
/data/certificates/
```

Caddy 日志应显示：

```text
FileStorage:/data
```

### Reality

在服务器上观察：

```bash
journalctl -u sing-box -f
```

随后用真实 Reality 客户端连接。连接到达 sing-box 即表示完整路径成功：

```text
Reality Client -> TCP 443 -> Caddy-L4 -> sing-box
```

偶发的 Layer4 `EOF` 日志通常是扫描器或在完整 TLS ClientHello 前断开的连接；只有客户端实际无法连接且错误持续出现时才需要深入排查。

---

## 新增服务

### 新增 Web/API 域名

确保新容器加入同一 Docker 网络，并在 Caddyfile 末尾增加：

```caddyfile
<NEW_DOMAIN> {
    reverse_proxy <NEW_SERVICE>:<NEW_PORT>
}
```

Caddy 将自动为该域名申请和续期证书。随后执行“格式化 → 验证 → reload”。

### 新增 Reality SNI：同一 sing-box 后端

将多个 SNI 写入同一个 matcher：

```caddyfile
@reality tls sni <REALITY_SNI_1> <REALITY_SNI_2> <REALITY_SNI_3>
```

它们都将转发到同一个 `host.docker.internal:<SINGBOX_PORT>`。

### 新增 Reality SNI：不同后端端口

为每个后端创建独立且不重叠的 matcher/route：

```caddyfile
layer4 {
    @reality_a tls sni <SNI_A>
    route @reality_a {
        proxy host.docker.internal:<PORT_A>
    }

    @reality_b tls sni <SNI_B>
    route @reality_b {
        proxy host.docker.internal:<PORT_B>
    }
}
```

每个宿主机后端端口都应先使用 `ss -lntp` 和临时 Docker 容器 `nc -vz` 验证可达。

---

## 安全与运维

- 若目标是只从公网暴露 443，应检查宿主机 sing-box 后端端口是否被公网直接访问。sing-box 监听 `*:8443` 仅说明其接受所有本地接口连接；是否暴露公网取决于 VPS 防火墙与云防火墙规则。
- 应允许 Docker host gateway 访问 sing-box 后端端口，同时拒绝公网直连该后端端口。
- 定期备份 `/opt/apps/caddy/data`；其中包含证书私钥与 ACME 账户。
- `livekit/caddyl4:latest` 可用但会随拉取而变化。升级前备份 `/data`、执行 `adapt --validate`，再重建容器。
- 普通 Web 域名与 Reality SNI 必须保持互不重叠；不要让 Caddy 尝试为 Reality SNI 申请证书。

```bash
# 备份 Caddy TLS/ACME 数据
tar -C /opt/apps/caddy -czf caddy-data-$(date +%F).tar.gz data
```

## 固定操作原则

```text
编辑 Caddyfile
  -> fmt
  -> adapt --validate
  -> reload（仅配置改动）/ force-recreate（Compose 改动）
  -> 查看日志
  -> HTTPS、HTTP/3、Reality 实测
```
