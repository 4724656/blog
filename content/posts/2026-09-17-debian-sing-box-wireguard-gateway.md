---
title: "Debian 宿主机上的 sing-box、WireGuard 与网络转发设计"
description: "记录 Debian 宿主机上 sing-box、WireGuard、Linux 转发与 Docker 防火墙的协同设计，并梳理 WireGuard 从握手成功到真正访问内网和互联网所需的关键条件。"
publishDate: "2026-09-17"
tags: ["Debian", "sing-box", "WireGuard", "网络"]
---

> WireGuard 负责把客户端安全地送进来，sing-box 负责流量分流，Linux 转发、防火墙和 NAT 负责让数据包真正完成往返。

## 一、整体架构

这套方案部署在一台 Debian 宿主机上，主要承担三项职责：

- 通过 sing-box 的 TUN 接口接管宿主机及相关来源的外网流量。
- 通过 WireGuard 为远程客户端提供安全接入。
- 通过 Linux 内核转发、iptables 和 NAT，将 WireGuard 客户端连接到家庭局域网及互联网。

整体流量可以理解为：

```text
远程客户端
    │
    │ WireGuard 加密隧道
    ▼
wg0：远程接入层
    │
    │ Linux 内核转发
    ▼
宿主机路由与防火墙
    │
    ├── 局域网流量：经物理网卡访问内网设备
    │
    └── 外网流量：交给 sing-box 的 tun0，再由代理出口转发
```

WireGuard 只负责加密封装和虚拟接口，不会自动完成 IP 转发、Docker 放行或局域网回程路由。这些工作需要宿主机的路由、防火墙和 NAT 配合完成。

## 二、sing-box 配置

主配置文件位于：

```text
/etc/sing-box/config.json
```

核心配置通常包括：

- 创建 `tun0` TUN 接口，并分配独立的 IPv4、IPv6 地址段。
- 启用 `auto_route`，让 sing-box 自动配置相关系统路由。
- 启用 `strict_route`，减少流量绕过 TUN 的可能性。
- 提供 DNS 服务，并按照规则进行 DNS 分流。
- 根据目标地址、域名、规则集和私有地址属性决定直连或代理。
- 通过 Clash API 或 REST API 提供运行状态和节点管理能力。

示例地址可以使用占位符：

```text
IPv4：172.31.255.1/30
IPv6：fdxx:xxxx:xxxx::1/126
```

`auto_route` 并不只是给 `tun0` 添加一条默认路由，在 Linux 上通常还会涉及策略路由、路由表和规则优先级。`strict_route` 用于强化流量接管，但并不代表所有流量在任何情况下都会经过代理，最终效果仍取决于路由规则、直连规则、DNS 路径以及宿主机和上游网络的其他路由。

参考：[sing-box 客户端代理文档](https://sing-box.sagernet.org/manual/proxy/client/)

### Systemd 覆盖配置

如果 sing-box 还需要配合 Docker 转发规则，可以使用 systemd 覆盖配置：

```text
/etc/systemd/system/sing-box.service.d/override.conf
```

示例：

```ini
[Service]
User=root
ExecStartPost=/sbin/iptables -I DOCKER-USER -i <WAN_IF> -j ACCEPT
ExecStopPost=-/sbin/iptables -D DOCKER-USER -i <WAN_IF> -j ACCEPT
```

`<WAN_IF>` 必须替换为当前系统真实参与转发的接口名称。`ExecStopPost` 前的 `-` 表示删除规则失败时，不应阻断服务停止流程。

TUN 接口、系统路由和策略路由通常需要较高权限，但生产环境不应因为“运行不起来”就无条件使用 root。应先确认是否只需要 `CAP_NET_ADMIN`、创建 TUN 的权限，或确实需要完整 root 权限，同时避免把 API 端口暴露到不可信网络。

## 三、Docker 与 `DOCKER-USER`

Docker 会修改主机的转发规则，并可能将 `FORWARD` 链默认策略设为 `DROP`。Docker 官方建议把管理员自定义的转发规则放在 `DOCKER-USER` 链中，因为该链会在 Docker 自身规则之前处理。

```shell
iptables -I DOCKER-USER -i wg0 -j ACCEPT
iptables -I DOCKER-USER -o wg0 -j ACCEPT
```

其中：

- `-i wg0`：允许从 WireGuard 接口进入主机的转发流量。
- `-o wg0`：允许发往 WireGuard 客户端的返回流量。

更严格的规则可以限制来源网段和连接状态：

```shell
iptables -I DOCKER-USER \
  -i wg0 \
  -s <WG_SUBNET> \
  -j ACCEPT
```

参考：[Docker 与 iptables](https://docs.docker.com/engine/network/firewall-iptables/)

## 四、WireGuard 配置

核心配置文件：

```text
/etc/wireguard/wg0.conf
```

由于其中通常包含服务端私钥，权限应设置为：

```shell
chmod 600 /etc/wireguard/wg0.conf
```

发布文章时，不要暴露服务端私钥、客户端私钥、真实公网 IP、动态域名、真实端口、完整局域网拓扑或可识别设备名称。公钥通常不能直接用于冒用隧道，但为了减少环境指纹，仍建议统一替换成占位符。

### 接口与客户端地址

服务端接口示例：

```ini
[Interface]
Address = <WG_SERVER_IP>/<PREFIX>
PrivateKey = <SERVER_PRIVATE_KEY>
ListenPort = <WG_PORT>
```

每个客户端使用独立地址：

```ini
[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = <CLIENT_IP>/32
```

在服务端配置中，`AllowedIPs` 同时承担对等端地址登记和路由匹配的作用。因此不同客户端不能重复使用同一个 `/32` 地址，否则可能产生路由归属冲突。

### `PostUp` 与 `PostDown`

常见的启动规则如下：

```ini
PostUp = iptables -I DOCKER-USER -i wg0 -j ACCEPT
PostUp = iptables -I DOCKER-USER -o wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o <LAN_IF> -j MASQUERADE
```

停止接口时删除对应规则：

```ini
PostDown = iptables -D DOCKER-USER -i wg0 -j ACCEPT 2>/dev/null
PostDown = iptables -D DOCKER-USER -o wg0 -j ACCEPT 2>/dev/null
PostDown = iptables -t nat -D POSTROUTING -o <LAN_IF> -j MASQUERADE 2>/dev/null
```

`wg-quick` 会在接口启动后执行 `PostUp`，停止时执行 `PostDown`。实际规则应结合主机的防火墙策略进行收敛，不建议无条件放行所有转发流量。

## 五、开启内核转发与 NAT

配置文件：

```text
/etc/sysctl.d/99-ipforward.conf
```

内容：

```text
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

应用配置：

```shell
sysctl --system
```

这允许 Linux 在 `wg0`、物理网卡、`tun0` 和 Docker bridge 之间转发数据包。

### NAT 为什么重要

访问局域网时，WireGuard 客户端的请求可能类似这样：

```text
源地址：10.10.99.3
目标地址：192.168.2.20
```

如果家庭主路由或 NAS 不知道 `10.10.99.0/24` 应该经由哪台设备返回，回包就可能走错路径。此时可以在正确的物理出口接口上执行 MASQUERADE：

```shell
iptables -t nat -A POSTROUTING -o <LAN_IF> -j MASQUERADE
```

数据包离开物理接口时，源地址会被转换为宿主机的局域网地址，例如：

```text
源地址：192.168.2.99
目标地址：192.168.2.20
```

NAS 会认为请求来自同一局域网内的宿主机，回包由宿主机根据连接跟踪状态还原并转发给 WireGuard 客户端。

如果能够在家庭主路由上添加静态路由，也可以不使用 NAT：

```text
目标网段：10.10.99.0/24
下一跳：Debian 宿主机的局域网地址
```

这种方式能保留客户端真实地址，但需要控制主路由，配置复杂度也更高。家庭网络中，MASQUERADE 往往更容易部署。

## 六、网段和接口名不要想当然

Docker、家庭局域网、VPN、虚拟机和 Kubernetes 都可能使用私有地址段。规划 TUN 和 WireGuard 网段时，应先检查现有路由和 Docker 网络，避免重叠：

```shell
ip route
docker network ls
docker network inspect <NETWORK_NAME>
```

使用 `172.31.255.0/30` 这类较少被自动分配的地址可以降低冲突概率，但不能保证永久不冲突。更准确的原则是：

> 为 TUN 和 WireGuard 选择经过规划、且与现有 Docker、局域网、VPN 和虚拟化网络不重叠的地址空间。

物理网卡名称也必须以当前系统为准，不要直接套用网上教程中的 `eth0` 或 `ens18`：

```shell
ip -br link
ip -details link show
ip route get 1.1.1.1
```

`ip route get` 输出中的 `dev` 字段，就是内核访问目标时选择的出口接口。

## 七、WireGuard 一开始不通的原因

“能建立 WireGuard 握手”只说明隧道建立，不代表客户端已经具备完整的网络访问路径。常见缺口包括：

| 问题 | 缺少或错误的配置 | 结果 |
| --- | --- | --- |
| 内核未转发 | `net.ipv4.ip_forward = 1` 未启用 | 只能访问宿主机，不能访问其他网络 |
| Docker 拦截 | `DOCKER-USER` 没有放行 `wg0` | 转发流量在 FORWARD 路径被丢弃 |
| 回程路由缺失 | 没有 MASQUERADE，且局域网没有静态路由 | 请求能到达内网设备，但回包找不到客户端 |
| 接口名称错误 | 使用了错误的物理接口名 | NAT 或放行规则没有匹配到实际流量 |

完整的修复链条通常是：

```text
开启 IP 转发
    ↓
放行 wg0 的转发流量
    ↓
在正确的出口接口上执行 MASQUERADE
    ↓
确认 sing-box 对局域网流量执行直连或绕过代理
```

关键排查命令：

```shell
ip route
ip rule
iptables -vnL DOCKER-USER
iptables -t nat -vnL POSTROUTING
wg show
```

如果需要观察数据包经过哪些接口，可以使用：

```shell
tcpdump -ni wg0
tcpdump -ni <LAN_IF>
tcpdump -ni tun0
```

## 八、完整流量路径

### 访问局域网

```text
远程客户端
    │ 10.10.99.3
    ▼
WireGuard wg0
    │
    ▼
Linux 内核转发
    │
    ▼
DOCKER-USER 放行
    │
    ▼
物理接口 <LAN_IF> 与 NAT
    │
    ▼
家庭局域网设备
```

### 访问互联网

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
    └── 公网地址：进入 tun0
                    │
                    ▼
              sing-box 分流
                    │
                    ▼
                代理出口
```

最终是否按预期工作，不能只看配置文件。应结合路由表、防火墙计数器、`wg show` 和 `tcpdump` 观察实际数据包路径。
