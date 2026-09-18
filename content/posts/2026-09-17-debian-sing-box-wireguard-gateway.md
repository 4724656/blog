---
title: "Debian 宿主机上的 sing-box、WireGuard 与网络转发设计"
description: "记录在 Debian 宿主机上组合使用 sing-box、WireGuard、Linux 内核转发和 Docker 防火墙的完整实践，重点解析 WireGuard 握手成功后局域网与互联网不通的核心症结与排障链路。"
publishDate: "2026-09-17"
updatedDate: "2026-09-18"
tags: ["Debian", "sing-box", "WireGuard", "网络转发", "防火墙"]
---

> **核心心智模型**：WireGuard 只负责把客户端安全地接进宿主机。数据包能不能继续到达局域网或互联网，取决于 Linux 内核转发、iptables / Docker 防火墙放行、NAT 源地址伪装，以及 sing-box 的路由分流规则。

## 一、这套方案解决什么问题

我把一台 Debian 宿主机作为家庭局域网的**远程安全接入**与**网络流量分流节点**：

- **WireGuard**：为外网远程客户端提供轻量、低延迟且加密的接入入口（`wg0` 接口）；
- **Linux 内核与 iptables / NAT**：负责在各虚拟与物理接口间转发数据包，并实现局域网回包的地址伪装；
- **sing-box**：通过 TUN 虚拟网卡（`tun0`）接管需要科学分流的外网流量。

整套架构的拓扑关系与分流调度链路如下图所示：

![Debian 宿主机 sing-box 与 WireGuard 网络转发拓扑](/diagrams/debian-sing-box-wireguard-gateway.svg)

这几个组件的分工非常清晰：
- **WireGuard** 提供虚拟网卡与加密传输通道；
- **sing-box** 决定出站流量是走直连还是境外代理；
- **Linux 路由、防火墙与 NAT** 负责确保请求与回包能够走完一条端到端的闭环路径。

仅配置好 WireGuard，通常只能说明隧道握手成功，绝不等于整个网络已经打通。

---

## 二、sing-box：用 TUN 接口接管流量

主配置文件通常位于：

```text
/etc/sing-box/config.json
```

在这类转发场景下，建议重点核对以下配置项：

- `tun0` 接口是否成功创建，且分配的 IP 地址段没有与本地其他虚拟网络冲突；
- `auto_route` 是否已启用；
- 是否需要启用 `strict_route` 来降低流量绕过 TUN 的几率；
- DNS 请求是否准确送入 sing-box，内置分流规则是否符合预期；
- 私有网段（RFC 1918）是否配置为 `direct`（直连），公网地址是否走相应的代理出口。

示例地址仅用于说明格式规范：

```text
IPv4：172.31.255.1/30
IPv6：fdxx:xxxx:xxxx::1/126
```

> **原理解析**：`auto_route` 并不只是单纯向系统路由表追加一条默认路由。在 Linux 环境下，它通常会联动创建策略路由（ip rule）、独立路由表及规则优先级。`strict_route` 会进一步强化流量接管力度，但并不意味着所有数据包都能无条件命中代理——直连规则、DNS 解析路径、宿主机原有路由以及上游主路由配置都会对最终走向产生影响。

参考文档：[sing-box 官方客户端代理手册](https://sing-box.sagernet.org/manual/proxy/client/)

### 运行权限的最佳实践

如果 sing-box 同时承担创建 TUN 虚拟接口、调整系统策略路由以及协同防火墙规则的任务，默认的低权限系统用户可能会权限受限。建议先评估是否可以通过授予 `CAP_NET_ADMIN` 能力或 TUN 设备属主权限解决，**切勿一遇到报错就直接盲目换用 root 运行**。

若确需以 `root` 权限托管：
1. 严禁将 sing-box 的 RESTful 管理 API 暴露在不可信的外部网络；
2. 尽量裁剪不必要的辅助进程。

当需要配合 Docker 防火墙规则联动时，可编写 systemd drop-in 覆盖配置来管理启停操作：

```text
/etc/systemd/system/sing-box.service.d/override.conf
```

配置文件示例：

```ini
[Service]
User=root
ExecStartPost=/sbin/iptables -I DOCKER-USER -i <WAN_IF> -j ACCEPT
ExecStopPost=-/sbin/iptables -D DOCKER-USER -i <WAN_IF> -j ACCEPT
```

> ⚠️ 注意：`<WAN_IF>` 必须替换为当前机器实际对外通信的物理接口名。`ExecStopPost` 开头的 `-` 符号表示该命令执行失败时不阻断整体服务的正常停止流程。

---

## 三、Docker 环境下的转发陷阱：DOCKER-USER 链

一旦宿主机部署了 Docker，Docker 会深度介入主机的 iptables 规则链，甚至可能在某些 Linux 发行版或配置下将 `FORWARD` 链的默认策略强制改为 `DROP`。

如果直接在 `FORWARD` 链追加规则，很可能会在 Docker 重启后被冲掉或置于 Docker 自定义规则之后。**管理员的宿主机级转发规则必须写入 `DOCKER-USER` 链**，因为它是 Docker 官方预留给用户自定义规则、并在处理 Docker 自身容器网络之前优先执行的入口。

最基础的双向放行指令如下：

```bash
# 放行来自 WireGuard 接口的入站转发数据包
iptables -I DOCKER-USER -i wg0 -j ACCEPT

# 放行返回给 WireGuard 客户端的出站响应数据包
iptables -I DOCKER-USER -o wg0 -j ACCEPT
```

在正式生产环境中，不建议对整卡无限制放行，建议进一步收紧源网段限制：

```bash
iptables -I DOCKER-USER \
  -i wg0 \
  -s <WG_SUBNET> \
  -j ACCEPT
```

参考文档：[Docker 官方防火墙与 iptables 指南](https://docs.docker.com/engine/network/firewall-iptables/)

---

## 四、WireGuard 配置与生命周期钩子

服务端配置文件默认路径：

```text
/etc/wireguard/wg0.conf
```

由于该配置文件直接包含服务端私钥，必须将其文件权限收敛：

```bash
chmod 600 /etc/wireguard/wg0.conf
```

> 💡 **脱敏建议**：在对外分享或归档配置时，应将私钥、公钥、公网真实 IP、DDNS 动态域名、通信端口、内网网段及机器主机名全面替换为占位符（如 `<SERVER_PRIVATE_KEY>`、`<WG_PORT>`）。

### 服务端与客户端地址分配

```ini
[Interface]
Address = <WG_SERVER_IP>/<PREFIX>
PrivateKey = <SERVER_PRIVATE_KEY>
ListenPort = <WG_PORT>

[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = <CLIENT_IP>/32
```

在 WireGuard 服务端模型中，`AllowedIPs` 既充当合法对等端公钥的登记表，又作为内核数据包的路由匹配键。因此**不同客户端之间严禁共用同一个 `/32` IP 地址**，否则会产生严重的路由归属冲突。

### PostUp 与 PostDown 自动化钩子

利用 `wg-quick` 提供的生命周期钩子，可以在网卡拉起与关闭时自动注入和清理 iptables 规则：

```ini
# 接口拉起时自动生效
PostUp = iptables -I DOCKER-USER -i wg0 -j ACCEPT
PostUp = iptables -I DOCKER-USER -o wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o <LAN_IF> -j MASQUERADE

# 接口销毁时清理规则
PostDown = iptables -D DOCKER-USER -i wg0 -j ACCEPT 2>/dev/null
PostDown = iptables -D DOCKER-USER -o wg0 -j ACCEPT 2>/dev/null
PostDown = iptables -t nat -D POSTROUTING -o <LAN_IF> -j MASQUERADE 2>/dev/null
```

规则定义应与网络物理拓扑精准对应，严禁泛化放行或遗漏清理逻辑。

---

## 五、内核转发与局域网回包困境（MASQUERADE）

首先必须确保操作系统层面开启了 IPv4/IPv6 转发机制：

编辑 `/etc/sysctl.d/99-ipforward.conf`：

```text
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

立即加载使之生效：

```bash
sysctl --system
```

如果未开启内核转发，客户端即使与宿主机成功握手，流量也无法跨网卡传递到宿主机身后的局域网设备。

### 核心疑难：为什么访问局域网总是卡在回包阶段？

这是最容易让人困惑的网络现象。假设场景如下：
- WireGuard 客户端分配的虚拟 IP 为 `10.10.99.3`；
- 目标局域网中运行着一台 NAS，IP 为 `192.168.2.20`；
- Debian 宿主机局域网物理网卡 IP 为 `192.168.2.99`。

当请求发出时：

```text
数据包原始状态：
源 IP：10.10.99.3  --->  目标 IP：192.168.2.20
```

数据包经由 `wg0` 转发并通过物理网卡成功抵达 NAS。**但当 NAS 尝试组织响应数据包发回时，问题出现了**：
NAS 发现目标是 `10.10.99.3`，检查本地路由表，找不到该网段的专门下一跳，只能把回包扔给家庭主路由（如 `192.168.2.1`）。而家庭主路由器同样不知道 `10.10.99.0/24` 在哪里，直接把回包丢弃或转发至公网，造成“握手正常、发包成功，但永远收不到回包”。

#### 解决方案一：在物理网卡启用 MASQUERADE（推荐，无需改动上游）

最直接、最通用的工程方案是在宿主机物理网卡上做源地址转换（SNAT/MASQUERADE）：

```bash
iptables -t nat -A POSTROUTING -o <LAN_IF> -j MASQUERADE
```

当数据包离开宿主机物理网卡发往局域网时，内核 conntrack 会将源地址偷换成宿主机自身的局域网 IP：

```text
经 MASQUERADE 后的数据包：
源 IP：192.168.2.99  --->  目标 IP：192.168.2.20
```

此时 NAS 看到的是来自同二层局域网邻居 `192.168.2.99` 的请求，直接将响应数据送回宿主机。宿主机的连接跟踪模块识别出此连接，自动将目标 IP 还原为 `10.10.99.3`，顺畅推回 `wg0` 接口。

#### 解决方案二：在主路由添加静态路由条目

如果你拥有家庭主路由的完整管理权限，且希望在局域网审计日志中保留客户端原始 IP，也可以在主路由上增加一条静态路由：

```text
目标网络：10.10.99.0/24
下一跳（Gateway）：Debian 宿主机局域网 IP (192.168.2.99)
```

这样局域网设备的回包即可经由主路由正确重定向给 Debian 宿主机。但在大多数网络拓扑下，MASQUERADE 依然是最解耦、抗干扰性最好的方案。

---

## 六、网段规划与接口名称现场确认

在混合了 Docker、WireGuard、家庭 LAN 以及 sing-box 的宿主机上，最忌讳不同接口的私有网段发生重叠碰撞。建议在初始化网段前执行系统盘查：

```bash
ip route
docker network ls
docker network inspect <NETWORK_NAME>
```

规划原则：**让 TUN 虚拟网卡、WireGuard 隧道、Docker Bridge、局域网子网彼此分配完全隔离的地址段**。例如使用 `172.31.255.0/30` 能有效减少与常规 `172.17.x.x` 或 `192.168.x.x` 发生碰撞。

此外，**切勿死板照抄网络教程中的网卡名**（如 `eth0`）。现代 systemd 可预测网卡命名机制下，物理网卡常命名为 `ens18`、`enp6s18` 或 `eno1`，必须现场通过命令行确定真实出口：

```bash
ip -br link
ip route get 1.1.1.1
```

`ip route get` 输出中的 `dev` 字段，即为内核路由选择的默认物理出口。如果把 NAT 规则里的网卡名写错，防火墙规则看似存在，却永远不会被数据包命中。

---

## 七、排错全景表：WireGuard 为什么连上却不通？

当你遇到“客户端显示已连接且有少量握手包，但网页无法打开、内网 Ping 不通”时，按以下矩阵逐一排查：

| 检查项 | 常见根因 | 典型排查现象 | 解决手段 |
| :--- | :--- | :--- | :--- |
| **内核转发** | `net.ipv4.ip_forward` 值为 `0` | 能与宿主机握手，但完全无法跨越网卡进入其他网络 | 检查 sysctl 并在 `/etc/sysctl.d/` 中持久化开启 |
| **防火墙拦截** | Docker 环境下 `DOCKER-USER` 链未放行 `wg0` | 数据包进入主机后直接被内核丢弃，计数器无增长 | 执行 `iptables -I DOCKER-USER -i wg0 -j ACCEPT` |
| **回程路由缺失** | 未配置物理接口 MASQUERADE，上游亦无静态路由 | 请求已到达局域网设备，但设备回包走丢 | 在正确物理网卡追加 MASQUERADE 规则 |
| **出口网卡写错** | `<LAN_IF>` 填写与真实硬件名称不一致 | `iptables -t nat -vnL` 中规则命中计数为 `0` | 使用 `ip route get` 纠正真实网卡接口名称 |
| **sing-box 分流配置** | 局域网私有网段误被 route 规则归入代理出站 | 访问内网设备异常卡顿、报错或重定向至境外节点 | 在 sing-box 规则中确保私网网段走 `direct` |

推荐的标准排错闭环链路：

```text
开启内核 IP 转发 (sysctl)
    ↓
在 DOCKER-USER 链双向放行 wg0 流量
    ↓
在正确物理出口网卡配置 MASQUERADE
    ↓
核对 sing-box 分流规则，确保内网直连不绕路
```

日常调试时，应将握手信息、策略路由表与防火墙计数器结合分析：

```bash
wg show
ip route
ip rule
iptables -vnL DOCKER-USER
iptables -t nat -vnL POSTROUTING
```

如果仍无法确定丢包点，在三个关键接口同时抓包观察：

```bash
tcpdump -ni wg0
tcpdump -ni <LAN_IF>
tcpdump -ni tun0
```

---

## 八、开机时序事故：WireGuard 启动失败根因与双重防御

这套配置曾经历过一次极为典型的开机启动故障：主机计划性重启后，WireGuard 接口未能恢复，导致外部远程链路全部中断。

提取关键的 systemd 日志：

```text
wg-quick[794]: [#] iptables -I DOCKER-USER -i wg0 -j ACCEPT
iptables: No chain/target/match by that name.
wg-quick[794]: [#] ip link delete dev wg0
systemd[1]: Failed to start wg-quick@wg0.service
```

### 事故根因剖析

开机启动时，systemd 会并发启动依赖树上的各个单元。`wg-quick@wg0.service` 启动时机偶发性早于 `docker.service`。此时 Docker 尚未启动，内建的 `DOCKER-USER` 自定义链尚不存在。当 WireGuard 触发 `PostUp` 尝试向 `DOCKER-USER` 链插入规则时，立即抛出 `No chain/target/match by that name` 异常。

`wg-quick` 对 `PostUp` 失败执行强一致性回滚：**判定启动失败，并立刻删除刚创建的 `wg0` 网卡**。由于原生服务单元没有配置失败重试策略，即便数秒后 Docker 顺利就绪并创建了该链，WireGuard 也不会再被自动拉起。

### 双重防御解决策略

为彻底杜绝开机并发竞争冒险，实施双层加固：

#### 第一层：在 PostUp 脚本中实现防御性幂等操作

在网卡执行规则插入前，先行尝试建立该链（已存在时忽略报错），并通过 `-C` 参数检查规则是否已存在，防止重复插入：

```ini
PostUp = iptables -N DOCKER-USER 2>/dev/null || true; iptables -C DOCKER-USER -i wg0 -j ACCEPT 2>/dev/null || iptables -I DOCKER-USER -i wg0 -j ACCEPT; iptables -C DOCKER-USER -o wg0 -j ACCEPT 2>/dev/null || iptables -I DOCKER-USER -o wg0 -j ACCEPT; iptables -t nat -C POSTROUTING -o <LAN_IF> -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o <LAN_IF> -j MASQUERADE
```

#### 第二层：通过 systemd override 明确启动依赖与失败自动恢复

创建 `/etc/systemd/system/wg-quick@wg0.service.d/override.conf`：

```ini
[Unit]
After=network-online.target docker.service
Wants=network-online.target

[Service]
Restart=on-failure
RestartSec=5s
```

`After=docker.service` 明确约束时序关系，确保 WireGuard 尽可能晚于 Docker 启动；而 `Restart=on-failure` 则赋予了服务在遭遇非预期环境异常时自动拉起自愈的能力。

修复完成后，系统状态恢复健康：
- `wg0` 稳定保持 UP 状态，持续监听 WireGuard 端口；
- `DOCKER-USER` 链与 `POSTROUTING` NAT 计数器正常递增；
- 模拟执行多次整机 `reboot` 压测，WireGuard 均 100% 自动就绪。

---

## 九、总结：两条核心流量路径与验证原则

在完成上述全部配置后，Debian 宿主机上实际形成了两条互不干扰、权责明确的数据路径：

### 1. 访问局域网内部（NAS / 私有服务）

```text
远程移动客户端
    │  (WireGuard UDP 隧道加密)
    ▼
虚拟网卡 wg0
    │  (内核解密与 IP 转发)
    ▼
DOCKER-USER 防火墙放行
    │  (路由匹配为 LAN 私有地址段)
    ▼
物理出口网卡 <LAN_IF> + MASQUERADE (源 IP 替换为宿主机局域网 IP)
    │  (二层直达)
    ▼
局域网目标设备 (回包直接返回宿主机，conntrack 自动还原推回 wg0)
```

### 2. 访问公网 / 互联网流量

```text
远程移动客户端
    │  (WireGuard UDP 隧道加密)
    ▼
虚拟网卡 wg0
    │  (内核解密与 IP 转发)
    ▼
宿主机路由与 sing-box 策略规则
    │  (DNS 拦截解析与分流匹配)
    ▼
虚拟网卡 tun0 (公网流量入栈)
    │  (sing-box 核心代理引擎)
    ▼
指定 Proxy 出口节点 ---> 目标公网网站
```

网络工程调试始终遵循一个铁律：**配置文件写得再完美，也只代表预期设计；只有 `wg show` 的握手时间戳、iptables 的实时计数器增长，以及 `tcpdump` 在关键网卡上抓到的真实报文，才是验证链路畅通的唯一标准。**
