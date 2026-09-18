---
title: "Debian 宿主机上的 sing-box、WireGuard 与网络转发设计"
description: "记录我在 Debian 宿主机上组合使用 sing-box、WireGuard、Linux 转发和 Docker 防火墙的过程，重点说明 WireGuard 为什么能握手，却仍然访问不了内网和互联网。"
publishDate: "2026-09-17"
tags: ["Debian", "sing-box", "WireGuard", "网络"]
---

> WireGuard 只负责把客户端接进来。数据包能不能继续走到内网或互联网，还要看 Linux 转发、防火墙、NAT，以及 sing-box 的路由规则。

## 这套方案解决什么问题

我把一台 Debian 主机当作远程接入和网络转发节点：

- sing-box 用 TUN 接口接管需要代理的外网流量。
- WireGuard 给远程客户端提供加密入口。
- Linux 内核、iptables 和 NAT 负责把客户端流量转到家庭局域网或互联网。

流量关系大致如下：

```text
远程客户端
    │ WireGuard 加密隧道
    ▼
wg0
    │ Linux 内核转发
    ▼
宿主机路由与防火墙
    │
    ├── 局域网：经物理网卡访问内网设备
    │
    └── 公网：按规则进入 sing-box 的 tun0
```

这几个组件分工很明确：WireGuard 提供虚拟接口和加密传输，sing-box 决定流量直连还是代理，Linux 路由、防火墙和 NAT 负责让请求和回包走完一条完整路径。只配置 WireGuard，通常只能说明隧道建立了，不能说明网络已经打通。

## sing-box：用 TUN 接管流量

主配置文件通常放在：

```text
/etc/sing-box/config.json
```

我在这类配置里会重点检查以下内容：

- `tun0` 是否创建成功，并且地址段没有和其他网络重叠。
- `auto_route` 是否启用。
- 是否需要 `strict_route` 来减少流量绕过 TUN。
- DNS 请求是否进入 sing-box，分流规则是否符合预期。
- 私有地址是否直连，公网地址是否交给代理出口。

示例地址只用于说明格式：

```text
IPv4：172.31.255.1/30
IPv6：fdxx:xxxx:xxxx::1/126
```

`auto_route` 不只是给 `tun0` 加一条默认路由。在 Linux 上，它可能会同时设置策略路由、路由表和规则优先级。`strict_route` 会进一步收紧流量接管，但它不代表所有流量在任何情况下都会经过代理。直连规则、DNS 路径、宿主机路由和上游设备的配置都会影响最终结果。

参考：[sing-box 客户端代理文档](https://sing-box.sagernet.org/manual/proxy/client/)

### 需要较高权限时

如果 sing-box 同时负责创建 TUN、设置系统路由和配合防火墙，默认的受限用户可能权限不够。可以先判断是否只需要 `CAP_NET_ADMIN` 或创建 TUN 的权限，不要一遇到启动失败就直接改成 root。

如果确实使用 root，至少要检查两件事：管理 API 不要暴露到不可信网络，服务本身也要尽量减少不必要的权限。

需要配合 Docker 规则时，可以通过 systemd 覆盖配置执行启动和停止动作：

```text
/etc/systemd/system/sing-box.service.d/override.conf
```

例如：

```ini
[Service]
User=root
ExecStartPost=/sbin/iptables -I DOCKER-USER -i <WAN_IF> -j ACCEPT
ExecStopPost=-/sbin/iptables -D DOCKER-USER -i <WAN_IF> -j ACCEPT
```

`<WAN_IF>` 必须替换成当前机器真正参与转发的接口名。`ExecStopPost` 前的 `-` 表示删除失败时不阻断服务停止流程。

## Docker 的转发规则放在哪里

Docker 会调整主机的转发规则，某些环境下还会把 `FORWARD` 链的默认策略设为 `DROP`。管理员自己的转发规则应放在 `DOCKER-USER` 链中，这样会在 Docker 自身规则之前处理。

最基本的双向放行规则是：

```shell
iptables -I DOCKER-USER -i wg0 -j ACCEPT
iptables -I DOCKER-USER -o wg0 -j ACCEPT
```

第一条允许流量从 WireGuard 进入转发路径，第二条允许返回流量发回客户端。生产环境不建议无条件放开所有流量，可以限制 WireGuard 网段：

```shell
iptables -I DOCKER-USER \
  -i wg0 \
  -s <WG_SUBNET> \
  -j ACCEPT
```

参考：[Docker 与 iptables](https://docs.docker.com/engine/network/firewall-iptables/)

## WireGuard 配置

服务端配置文件通常是：

```text
/etc/wireguard/wg0.conf
```

其中包含私钥，权限至少应设置为：

```shell
chmod 600 /etc/wireguard/wg0.conf
```

发布配置时，我会把服务端和客户端私钥、真实公网 IP、动态域名、端口、局域网拓扑和设备名称全部换成占位符。公钥本身通常不能直接用来冒充隧道，但为了减少环境指纹，也可以统一替换成 `<SERVER_PUBLIC_KEY>`、`<CLIENT_PUBLIC_KEY>`。

### 接口和客户端地址

```ini
[Interface]
Address = <WG_SERVER_IP>/<PREFIX>
PrivateKey = <SERVER_PRIVATE_KEY>
ListenPort = <WG_PORT>
```

每个客户端应使用独立地址：

```ini
[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = <CLIENT_IP>/32
```

在服务端配置中，`AllowedIPs` 既用于登记对等端地址，也参与路由匹配。因此不同客户端不能重复使用同一个 `/32`，否则可能出现路由归属冲突。

### `PostUp` 和 `PostDown`

接口启动时加入转发与 NAT 规则：

```ini
PostUp = iptables -I DOCKER-USER -i wg0 -j ACCEPT
PostUp = iptables -I DOCKER-USER -o wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o <LAN_IF> -j MASQUERADE
```

接口停止时删除：

```ini
PostDown = iptables -D DOCKER-USER -i wg0 -j ACCEPT 2>/dev/null
PostDown = iptables -D DOCKER-USER -o wg0 -j ACCEPT 2>/dev/null
PostDown = iptables -t nat -D POSTROUTING -o <LAN_IF> -j MASQUERADE 2>/dev/null
```

`wg-quick` 会在接口启动后执行 `PostUp`，停止时执行 `PostDown`。规则最好写得和实际网络边界一致，避免重复添加或放行范围过大。

## 内核转发和 NAT

先在 `/etc/sysctl.d/99-ipforward.conf` 中启用转发：

```text
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

然后应用：

```shell
sysctl --system
```

没有内核转发时，客户端可能能访问宿主机，却到不了宿主机后面的网络。

### 为什么局域网访问常常卡在回包

假设 WireGuard 客户端是 `10.10.99.3`，要访问局域网里的 NAS `192.168.2.20`：

```text
源地址：10.10.99.3
目标地址：192.168.2.20
```

NAS 收到请求后，需要知道 `10.10.99.0/24` 应该经由哪台设备返回。如果家庭路由器没有这条路由，回包就可能走错。最省事的做法是在正确的物理出口接口上做 MASQUERADE：

```shell
iptables -t nat -A POSTROUTING -o <LAN_IF> -j MASQUERADE
```

离开物理网卡时，源地址会变成宿主机的局域网地址：

```text
源地址：192.168.2.99
目标地址：192.168.2.20
```

NAS 会把请求当成来自同一局域网的宿主机，回包先回到宿主机，再由连接跟踪还原并转发给 WireGuard 客户端。

如果能修改家庭主路由，也可以不用 NAT，添加静态路由：

```text
目标网段：10.10.99.0/24
下一跳：Debian 宿主机的局域网地址
```

这样能保留客户端真实地址，但需要管理主路由。家庭网络里，MASQUERADE 通常更容易落地。

## 网段和接口名要先确认

Docker、家庭局域网、VPN、虚拟机和 Kubernetes 都可能使用私有地址。规划 TUN 和 WireGuard 网段前，先检查现有网络：

```shell
ip route
docker network ls
docker network inspect <NETWORK_NAME>
```

`172.31.255.0/30` 这类网段可以降低和常见自动分配网段撞车的概率，但不能保证以后永远不冲突。实际原则是：让 TUN、WireGuard、Docker、局域网和其他虚拟网络各用不重叠的地址空间。

物理接口也不能照抄教程。系统里可能叫 `eth0`、`ens18`、`enp6s18` 或 `eno1`，应当现场确认：

```shell
ip -br link
ip -details link show
ip route get 1.1.1.1
```

`ip route get` 输出里的 `dev` 字段，就是访问目标时内核选择的出口接口。NAT 规则写错接口名，规则看起来存在，实际却不会命中。

## WireGuard 为什么一开始不通

我遇到这类问题时，通常按下面的顺序查：

| 检查项 | 常见问题 | 现象 |
| --- | --- | --- |
| 内核转发 | `net.ipv4.ip_forward` 没开 | 能握手，但到不了其他网络 |
| 防火墙 | `DOCKER-USER` 没放行 `wg0` | 流量进入主机后被丢弃 |
| 回程路径 | 没有 NAT，也没有静态路由 | 请求到了设备，回包回不来 |
| 出口接口 | `<LAN_IF>` 写错 | NAT 规则没有命中 |
| sing-box 规则 | 局域网地址被错误送进代理 | 内网访问异常或绕路 |

修复链条通常就是：

```text
开启 IP 转发
    ↓
放行 wg0 的转发流量
    ↓
在正确出口接口上做 MASQUERADE
    ↓
确认 sing-box 对局域网地址直连或绕过代理
```

握手状态、路由表和防火墙计数器要一起看：

```shell
wg show
ip route
ip rule
iptables -vnL DOCKER-USER
iptables -t nat -vnL POSTROUTING
```

如果还不能确定数据包走到哪里，可以抓三个接口：

```shell
tcpdump -ni wg0
tcpdump -ni <LAN_IF>
tcpdump -ni tun0
```

## 一次开机事故：WireGuard 为什么没能自动起来

这套配置后来遇到过一次很典型的开机问题。机器重启后，WireGuard 没有正常恢复，客户端一度连不上。

日志里最关键的几行是：

```text
wg-quick[794]: [#] iptables -I DOCKER-USER -i wg0 -j ACCEPT
iptables: No chain/target/match by that name.
wg-quick[794]: [#] ip link delete dev wg0
systemd[1]: Failed to start wg-quick@wg0.service
```

问题出在启动顺序。开机时 systemd 会并发拉起多个服务，`wg-quick@wg0` 有时比 Docker 更早启动。此时 Docker 还没有创建 `DOCKER-USER` 链，WireGuard 的 `PostUp` 执行下面这条规则自然会失败：

```shell
iptables -I DOCKER-USER -i wg0 -j ACCEPT
```

`wg-quick` 对 `PostUp` 的错误处理比较严格。规则插入失败后，它会判定接口启动失败，并删除刚刚创建的 `wg0`。结果就是 WireGuard 服务停在 `failed` 状态，端口也不会监听。更麻烦的是，服务没有配置失败自动重启，后面即使 Docker 已经启动，`wg0` 也不会自己回来。

这次修复分成两层。第一层是在 `PostUp` 里先确保链存在，并且用 `-C` 检查规则，避免重复插入：

```ini
PostUp = iptables -N DOCKER-USER 2>/dev/null || true; iptables -C DOCKER-USER -i wg0 -j ACCEPT 2>/dev/null || iptables -I DOCKER-USER -i wg0 -j ACCEPT; iptables -C DOCKER-USER -o wg0 -j ACCEPT 2>/dev/null || iptables -I DOCKER-USER -o wg0 -j ACCEPT; iptables -t nat -C POSTROUTING -o <LAN_IF> -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o <LAN_IF> -j MASQUERADE
```

第二层是给 systemd 加上启动顺序和失败重启：

```ini
[Unit]
After=network-online.target docker.service
Wants=network-online.target

[Service]
Restart=on-failure
RestartSec=5s
```

`After=docker.service` 让 WireGuard 尽量排在 Docker 后面启动；脚本里的“先建链”和 systemd 的自动重试则是两道保险。以后即使启动时序偶尔不理想，服务也有机会在 Docker 就绪后重新拉起。

修复后检查到的状态：

- `wg0` 已恢复运行，并监听配置中的 WireGuard 端口。
- `DOCKER-USER` 的 WireGuard 放行规则和物理接口的 MASQUERADE 规则已生效。
- `net.ipv4.ip_forward = 1` 保持开启。

排查类似问题时，可以先看服务日志和接口状态：

```shell
journalctl -u wg-quick@wg0 --no-pager
systemctl status wg-quick@wg0
ss -lntup
wg show
```

这次事故说明，网络配置不能只验证“手动启动能不能跑”。还要模拟一次重启，确认 Docker、WireGuard、iptables 链和自动恢复机制之间的依赖关系。

## 两条实际流量路径

访问局域网时：

```text
远程客户端
    │
    ▼
WireGuard wg0
    │
    ▼
Linux 转发与 DOCKER-USER
    │
    ▼
物理接口 <LAN_IF> + NAT
    │
    ▼
局域网设备
```

访问互联网时：

```text
远程客户端
    │
    ▼
WireGuard wg0
    │
    ▼
宿主机路由与 sing-box 规则
    │
    ├── 私有地址：直连或绕过代理
    │
    └── 公网地址：进入 tun0，再由代理出口转发
```

最后还是要以实际数据包为准。配置文件写对，只代表“应该这样走”；`wg show`、iptables 计数器和 `tcpdump` 才能告诉你它实际上有没有走到那里。
