---
title: "OpenClaw 多台 Docker 主机 Node 节点部署实录"
description: "记录在 Ubuntu ARM64 云主机和 Linux Docker 主机上部署 OpenClaw Node 的完整过程，包括公网 WSS 直连、节点配对、systemd 持久化，以及从 SSH 隧道迁移到公网连接的实践经验。"
publishDate: "2026-08-31"
tags: ["OpenClaw", "Docker", "Linux", "运维"]
---

> 这次的目标，是让多台运行 Docker 的 Linux 主机都成为 OpenClaw Gateway 的远程 Node，由统一的 Gateway 管理各台机器上的容器和系统任务。

## 一、最终架构

整体采用一台 Gateway 管理多台 Node 的结构：

```text
OpenClaw Gateway
        │
        ├── Docker PC Node
        │
        └── Ubuntu ARM64 Node
```

两台 Node 都通过公网 WSS 连接 Gateway：

```text
Node
  │
  └── wss://<gateway-domain>:443
          │
          ▼
       Caddy
          │
          ▼
       Relay
          │
          ▼
   OpenClaw Gateway
```

Node 只需要主动访问 Gateway 的 HTTPS 端口，不需要在 Node 所在机器上开放 OpenClaw 入站端口，也不需要为 Node 做入站端口转发。

## 二、为什么选择公网 WSS

最开始，Docker PC 使用的是 SSH 本地端口转发：

```text
Node 127.0.0.1:18789
        │
        └── SSH 隧道
                │
                ▼
      Gateway 127.0.0.1:18789
```

这种方式安全可靠，但需要同时维护 OpenClaw Node 服务和 SSH/autossh 隧道服务。后来改为公网 WSS：

```text
Node → WSS :443 → Caddy → Relay → Gateway
```

这样每台 Node 只需要维护一个 `openclaw-node.service`，结构更简单，也更适合多台机器统一管理。

公网 WSS 不等于直接裸露 Gateway 端口。Gateway 仍然只监听本机回环地址，由 Caddy 和 relay 负责转发，并通过 TLS 和 Gateway 鉴权保护连接。

## 三、Gateway 侧准备反代

Gateway 本身监听本机地址：

```text
127.0.0.1:18789
```

由于 Caddy 运行在 Docker 网络中，无法直接访问 Gateway 的回环地址，因此增加一个本机 relay，将 Docker 网关地址上的端口转发到 Gateway：

```ini
[Unit]
Description=OpenClaw Gateway relay
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/socat TCP-LISTEN:18789,bind=<docker-gateway-ip>,reuseaddr,fork TCP:127.0.0.1:18789
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
```

Caddy 反代到 relay 地址：

```caddyfile
<gateway-domain> {
    reverse_proxy <docker-gateway-ip>:18789
}
```

### 一个容易踩坑的地方

如果 relay unit 写着：

```ini
Requires=openclaw-gateway.service
```

而 Gateway 后来改成用户级 systemd 服务，系统级的 `openclaw-gateway.service` 不再存在，relay 就可能被连带停止。此时应删除失效依赖，只保留网络依赖：

```ini
After=network-online.target
Wants=network-online.target
```

然后重新加载并启动：

```bash
systemctl daemon-reload
systemctl enable --now openclaw-caddy-relay.service
```

验证：

```bash
systemctl status openclaw-caddy-relay.service
ss -tlnp | grep 18789
```

## 四、处理反向代理身份校验

Gateway 经过 Caddy 和 relay 后，会看到代理转发的客户端请求头。新版 Gateway 对客户端来源识别更加严格，如果没有配置可信代理，可能出现：

```text
proxy_attribution_required
```

在 Gateway 配置中加入严格的本机可信代理范围：

```json
{
  "gateway": {
    "trustedProxies": [
      "127.0.0.1",
      "::1"
    ]
  }
}
```

配置修改后重启 Gateway：

```bash
systemctl --user restart openclaw-gateway.service
```

可信代理范围应尽可能小，不要使用允许任意来源的宽泛网段，避免不可信请求伪造客户端身份。

## 五、在新 Ubuntu ARM64 主机上安装 Node

本次新主机运行 Ubuntu LTS ARM64，使用 root 用户的用户级安装方式。推荐目录结构如下：

```text
/root/.nvm/
    └── Node.js 24.x

/root/.config/systemd/user/
    └── openclaw-node.service

/root/.openclaw/
    └── Node 身份和运行状态
```

OpenClaw 版本应尽量与 Gateway 保持一致：

```bash
openclaw --version
```

确认版本后，查看 Node 启动参数：

```bash
openclaw node run --help
```

## 六、首次使用公网 WSS 连接

首次连接不要直接把长期 Gateway Token 写进命令。推荐在 Gateway 的 Control UI 中进入：

```text
Devices → Pair device → Node host
```

生成一次性的 Node 配对命令，然后直接复制到远程主机终端执行：

```bash
openclaw node run --pair "oc-pair://<one-time-setup-code>"
```

配对链接具有以下特点：

- 有效时间有限
- 只能用于首次加入
- 不应该写入 systemd 服务参数
- 不应该发送到聊天窗口、工单或公开日志
- 配对完成后，Node 会保存自己的长期设备凭据

每台机器都必须使用独立的 Node 身份，不能复制其他机器的状态文件或密钥。

如果直接使用普通启动命令而没有认证信息，通常会看到：

```text
unauthorized: gateway token missing
```

这表示网络已经到达 Gateway，但缺少首次连接所需的认证信息，不是端口不通。

## 七、Gateway 侧批准 Node

首次连接后，在 Gateway 上查看节点状态：

```bash
openclaw nodes status
```

新节点可能显示为：

```text
paired · connected · pending approval
```

或者：

```text
reapproval pending
```

使用 Gateway 返回的精确请求 ID 批准：

```bash
openclaw nodes approve <request-id>
```

批准后重新检查：

```bash
openclaw nodes status
```

预期状态：

```text
paired · connected · approved
```

这里要区分两个概念：

- **设备配对**：允许 Node 连接 Gateway
- **能力审批**：决定 Node 可以暴露哪些命令和能力

配对成功并不代表已经获得 Docker 或 Shell 管理权限。

## 八、让 Node 变成持久化服务

首次前台运行验证成功后，使用 OpenClaw 自己的安装命令生成用户级服务：

```bash
openclaw node install \
  --force \
  --host <gateway-domain> \
  --port 443 \
  --tls \
  --display-name "Ubuntu ARM64"
```

启动并设置自启：

```bash
systemctl --user daemon-reload
systemctl --user enable --now openclaw-node.service
```

检查服务和连接日志：

```bash
systemctl --user status openclaw-node.service --no-pager
journalctl --user -u openclaw-node.service --no-pager -n 20
```

正常情况下应看到：

```text
node host gateway connected: wss://<gateway-domain>:443
```

## 九、root 用户的 systemd lingering

如果 Node 使用 root 用户级 systemd，需要确认用户退出 SSH 后用户级 systemd 不会被销毁：

```bash
loginctl enable-linger root
loginctl show-user root -p Linger
```

预期：

```text
Linger=yes
```

最终 Node 服务位于：

```text
/root/.config/systemd/user/openclaw-node.service
```

不要手工把它写进 `/etc/systemd/system/`，用户级 Node 服务应由 `openclaw node install` 自动生成和维护。

## 十、Docker 权限验证

Node 显示在线还不够，还要验证实际命令执行能力。在 Gateway 侧通过 Node 执行：

```bash
docker ps
docker compose ls
docker version
hostname
uname -a
```

如果能返回目标主机上的容器列表，说明 Node 连接、能力审批、远程执行链路和操作系统 Docker 权限都正常。

## 十一、把旧 Node 从 SSH 隧道迁移到公网 WSS

原来的 Docker PC 使用 SSH 隧道，迁移时不需要删除设备，也不需要重新配对。

### 迁移前

```text
openclaw-node.service
    → 127.0.0.1:18789
    → SSH 隧道
    → Gateway
```

### 迁移后

```text
openclaw-node.service
    → wss://<gateway-domain>:443
    → Caddy
    → Relay
    → Gateway
```

先确认当前身份和服务：

```bash
openclaw node status
```

停止 Node 和旧 SSH 隧道：

```bash
systemctl --user stop openclaw-node.service
sudo systemctl disable --now openclaw-tunnel.service
```

确认本地转发端口释放：

```bash
ss -tlnp | grep 18789 || echo "本地 18789 已释放"
```

重新生成 Node 服务：

```bash
openclaw node install \
  --force \
  --host <gateway-domain> \
  --port 443 \
  --tls \
  --display-name "Docker PC"
```

启动服务：

```bash
systemctl --user daemon-reload
systemctl --user enable --now openclaw-node.service
```

确认日志：

```bash
journalctl --user -u openclaw-node.service --no-pager -n 20
```

如果看到：

```text
node host gateway connected: wss://<gateway-domain>:443
```

说明迁移完成。

### 迁移过程中不需要做的事

- 不要删除原来的 Node 设备
- 不要删除 Node 身份文件
- 不要重新生成节点 ID
- 不要复制其他 Node 的密钥
- 不要删除执行审批配置
- 不要同时运行旧 SSH 隧道和新 Node 服务

因为连接地址改变，不代表节点身份改变。

## 十二、最终检查清单

### Gateway 侧

```bash
systemctl --user is-active openclaw-gateway.service
systemctl is-active openclaw-caddy-relay.service
openclaw nodes status
```

### Node 侧

```bash
systemctl --user is-active openclaw-node.service
systemctl --user is-enabled openclaw-node.service
loginctl show-user root -p Linger
journalctl --user -u openclaw-node.service --no-pager -n 20
```

### 远程执行

```bash
hostname
docker ps
docker compose ls
```

### 预期结果

```text
Gateway: active
Relay: active
Node: paired · connected · approved
Node service: active + enabled
Linger: yes
Docker command: successful
```

## 总结

这次部署的关键经验有三点。

第一，公网 WSS 直连比 SSH 本地端口转发更容易扩展。每台 Node 只维护一个用户级 `openclaw-node.service`，不需要额外维护 autossh 隧道。

第二，Node 的“能连接”和“能管理”是两回事。必须同时完成：

```text
网络连接 + 设备配对 + 能力审批 + 操作系统权限
```

第三，Node 迁移连接地址时，不需要删除原设备。只要保留原有 Node 身份，重新生成服务配置即可继续使用原来的审批记录。

最终，多台 Linux Docker 主机都可以作为 OpenClaw Node 接入统一 Gateway，由 Gateway 集中管理各台主机上的容器和系统任务。