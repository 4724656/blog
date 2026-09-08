---
title: "把 nftables 用清楚：静态规则、Docker 与 Fail2Ban 的职责边界"
description: "厘清 Debian VPS 中 nftables 静态规则、Docker 容器网络与 Fail2Ban 动态封禁的所有权和生命周期，避免保存运行时快照造成重启后规则混乱。"
publishDate: "2026-09-08"
tags: ["Debian", "nftables", "Fail2Ban", "Docker", "VPS安全"]
---

在 VPS 上配置防火墙时，最容易犯的错误通常不是规则语法，而是把**运行时规则**误认为需要永久保存的配置。

我曾经执行过：

```bash
nft list ruleset > /etc/nftables.conf
```

表面上看，这是把现有规则“保存起来”。实际上，它会把 Docker 自动创建的 NAT/转发规则、Fail2Ban 的动态封禁集合和当前被封禁 IP，一并写入系统开机加载的静态配置。重启后，系统先加载旧快照，Docker 与 Fail2Ban 再创建自己的规则，最终可能出现重复、残留或难以预测的防火墙状态。

> 正确思路不是“保存所有当前规则”，而是按**规则所有者**与**生命周期**拆开管理。

## 一、核心结论：只维护 `/etc/nftables.conf`

一台同时使用 Docker 与 Fail2Ban 的 Debian VPS，可以把规则分成三层：

| 层级 | 内容 | 谁维护 | 是否写入 `/etc/nftables.conf` |
| --- | --- | --- | --- |
| 静态主机防火墙 | SSH、HTTP/HTTPS、Ping、回环、默认拒绝策略 | 管理员 | 是 |
| Docker 网络规则 | 容器端口映射、DNAT、NAT、bridge 转发 | Docker | 否 |
| Fail2Ban 封禁规则 | 当前攻击 IP、封禁集合、jail 规则 | Fail2Ban | 否 |

因此，日常人工维护的唯一静态防火墙文件是：

```text
/etc/nftables.conf
```

它只放希望每次开机都稳定存在的规则，例如：

- SSH 管理端口；
- TCP 80、443 等公开 Web 服务端口；
- HTTP/3 / QUIC 所需的 UDP 443；
- loopback 回环通信；
- 已建立连接的回包；
- 必要 ICMP / ICMPv6；
- 默认 `drop` 的入站策略。

Docker 与 Fail2Ban 的规则同样重要，只是它们应由各自服务在运行时创建和清理，不应被静态配置文件接管。

## 二、推荐的静态规则文件

以下适用于常见 Debian VPS。示例假设 SSH 使用 TCP 22，Web 服务使用 TCP 80/443，且 Caddy 等服务需要 UDP 443 提供 HTTP/3。

> **加载前先防锁门：** 使用包含 `flush ruleset` 的配置前，确认 SSH 实际端口和云厂商安全组一致；保留至少两个 SSH 会话，并打开 VPS Web Console、VNC 或 Serial Console 等救援入口。

```nft
#!/usr/sbin/nft -f

# 仅维护静态主机防火墙规则。
# Docker 与 Fail2Ban 的运行时规则不应写进此文件。
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # 本机回环
        iifname "lo" accept

        # 已建立连接及关联流量
        ct state established,related accept

        # 尽早丢弃无效状态包
        ct state invalid drop

        # IPv4 ICMP：允许 Ping，并限速
        ip protocol icmp icmp type echo-request limit rate 10/second burst 20 packets accept
        ip protocol icmp icmp type {
            destination-unreachable,
            time-exceeded,
            parameter-problem
        } accept

        # IPv6 ICMPv6：邻居发现、路径 MTU 等基础网络功能需要
        ip6 nexthdr icmpv6 icmpv6 type {
            destination-unreachable,
            packet-too-big,
            time-exceeded,
            parameter-problem,
            nd-router-solicit,
            nd-router-advert,
            nd-neighbor-solicit,
            nd-neighbor-advert
        } accept
        ip6 nexthdr icmpv6 icmpv6 type echo-request limit rate 10/second burst 20 packets accept

        # 若 SSH 使用自定义端口，请将 22 替换为实际端口。
        tcp dport 22 accept
        tcp dport 80 accept
        tcp dport 443 accept
        udp dport 443 accept
    }

    # Docker 通常需要转发；不要贸然改为 drop。
    chain forward {
        type filter hook forward priority filter; policy accept;
    }

    # 普通服务器通常允许所有出站流量。
    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

`table inet filter` 可以在同一张表中处理 IPv4 和 IPv6。`input` 链的默认策略为 `drop`，即未明确允许的入站流量会被拒绝。

## 三、为什么不能保存完整 ruleset

下面的命令适合查看当前内核中的全部规则：

```bash
nft list ruleset
```

也适合做排障快照备份：

```bash
nft list ruleset > /root/nft-ruleset-backup-$(date +%F-%H%M%S).nft
```

但它不应当覆盖静态配置：

```bash
# 不要这样做
nft list ruleset > /etc/nftables.conf
```

完整 ruleset 可能包含：

- Docker 通过 `iptables-nft` 兼容层建立的 `DOCKER-*`、NAT 和转发规则；
- Fail2Ban 当前已封禁 IP 及动态 set/chain；
- 其他软件创建的临时规则。

这些规则应由相关服务启动时自行恢复。把它们导出到静态文件，会让系统先加载旧快照，再由服务创建新规则。

> `nft list ruleset` 用于查看和备份；`/etc/nftables.conf` 用于手工维护静态规则。

## 四、Docker 与 Fail2Ban 如何协作

## Docker：动态管理容器网络

在 Debian 上，Docker 常通过 `iptables-nft` 兼容层创建规则。规则最终会出现在 `nft list ruleset` 输出中，但 Docker 仍是这些规则的唯一管理者。

常用检查：

```bash
docker ps
iptables -S
iptables -t nat -S
nft list ruleset | grep -iE 'docker|DOCKER'
```

不要手工编辑或删除 `DOCKER-*` 链，也不要用 `iptables-save`、`netfilter-persistent save` 保存 Docker 的动态规则。

如果 Caddy 等容器发布了：

```text
80/tcp 443/tcp 443/udp
```

静态 nftables 的 `input` 链也应允许相应 TCP/UDP 端口。Docker 负责将流量转发至容器；静态主机防火墙负责决定公网流量能否进入主机。

Docker 的最终 nftables 表、链名称和位置会随 Docker、iptables 后端与系统版本变化。应以容器网络、端口映射和上述规则查询正常为准，而不是死盯某个固定表名。

## Fail2Ban：动态管理攻击者封禁

Fail2Ban 应使用官方 nftables action，不要手工创建 `f2b-sshd` 链，也不要每次封禁 IP 时添加一条静态规则。

一个基础 `/etc/fail2ban/jail.local` 示例：

```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
backend  = systemd
banaction = nftables

[sshd]
enabled = true
port    = ssh
filter  = sshd
action  = nftables[name=sshd, port=ssh, protocol=tcp]
```

采用 systemd journal 后端时，确保系统具备 Python systemd 绑定：

```bash
apt install -y python3-systemd
```

不同 Fail2Ban 与 nftables action 版本生成的表名、链名、集合名和 reject/drop 动作可能不同，因此不应以固定名称作为唯一验证标准。

推荐用 RFC 5737 文档测试地址验证：

```bash
fail2ban-client set sshd banip 192.0.2.1
fail2ban-client get sshd banip
nft list ruleset | grep -A 20 -B 5 '192.0.2.1'
fail2ban-client set sshd unbanip 192.0.2.1
```

`192.0.2.1` 是文档保留测试地址，不应分配给公网真实主机。

## 五、服务启动与持久化

静态规则、Docker 和 Fail2Ban 都应启用开机自启：

```bash
systemctl enable nftables.service
systemctl enable docker.service
systemctl enable fail2ban.service
```

检查：

```bash
systemctl is-enabled nftables.service docker.service fail2ban.service
```

正常应显示：

```text
enabled
enabled
enabled
```

系统启动后的职责顺序是：

1. `nftables.service` 加载 `/etc/nftables.conf` 中的静态主机规则；
2. `docker.service` 创建容器 NAT、端口映射和转发规则；
3. `fail2ban.service` 启动 jail，并在需要封禁时创建动态 nftables 规则。

避免同时启用两套静态规则恢复器。如果采用 `nftables.service`，通常应停用旧的 `netfilter-persistent.service`：

```bash
# 未安装该服务时安全跳过
systemctl disable --now netfilter-persistent.service 2>/dev/null || true
```

## 六、后期维护规则

## 新增或删除主机端口

以新增 WireGuard UDP 51820 为例：

1. 编辑 `/etc/nftables.conf`；
2. 在 `input` 链中加入：

```nft
udp dport 51820 accept
```

3. 检查语法：

```bash
nft -c -f /etc/nftables.conf
```

4. 同步确认云安全组/云防火墙已放行 UDP 51820；
5. 按下节安全顺序立即加载，或留待下次重启生效。

不要通过 `nft delete rule ... handle ...` 维护长期策略；rule handle 属于运行时标识，不适合作为可复用配置。

## 立即应用静态规则修改

示例配置包含 `flush ruleset`，直接执行：

```bash
nft -f /etc/nftables.conf
```

会删除 Docker 与 Fail2Ban 当前动态规则。应按所有者重新建立：

```bash
# 保持已有 SSH 会话不断开，并准备 VPS 控制台。
systemctl stop fail2ban 2>/dev/null || true
systemctl stop docker.socket docker

nft -c -f /etc/nftables.conf
nft -f /etc/nftables.conf

systemctl start docker
systemctl start fail2ban
```

若服务器没有 Docker 或 Fail2Ban，可省略相应 stop/start 步骤。加载后验证：

```bash
nft list table inet filter
systemctl --no-pager --full status docker fail2ban
iptables -S
iptables -t nat -S
```

## 仅修改文件、下次重启再生效

如果无需立即变更，只编辑并检查：

```bash
nano /etc/nftables.conf
nft -c -f /etc/nftables.conf
```

保存后不要加载；规则会在下次启动时由 `nftables.service` 自动读取。

## 修改 SSH 端口的安全流程

SSH 改端口最容易把自己锁在门外，应遵循以下顺序：

1. 保留两个已连接 SSH 会话，并打开 VPS Web Console/VNC；
2. 先在 sshd 配置中加入新端口，而不是立刻删除旧端口；
3. 在 `/etc/nftables.conf` 同时允许旧端口和新端口；
4. 确认云安全组/云防火墙已放行新端口；
5. 运行 `sshd -t` 检查配置；
6. 重启 sshd，并从第三个终端测试新端口登录；
7. 确认稳定后，再删除旧 SSH 端口和对应放行规则；
8. 修改 Fail2Ban 的 `[sshd] port`，使其与最终 SSH 监听端口一致。

## 七、部署或修改前检查清单

任何会加载 `flush ruleset` 的操作前，逐项确认：

- 当前 SSH 实际端口与 `/etc/nftables.conf` 中的放行规则一致；
- 云厂商安全组/云防火墙已放行 SSH 和必要的公开服务端口；
- 保留至少两个 SSH 会话；
- VPS Web Console、VNC 或 Serial Console 可用；
- 已备份静态配置与当前运行时快照到 `/root`，而不是覆盖 `/etc/nftables.conf`；
- `nft -c -f /etc/nftables.conf` 无输出且返回成功；
- Docker 正运行时，已计划好 stop → load → start 的恢复顺序。

建议备份：

```bash
install -d -m 700 /root/firewall-backup
cp -a /etc/nftables.conf /root/firewall-backup/nftables.conf.$(date +%F-%H%M%S)
nft list ruleset > /root/firewall-backup/ruleset.$(date +%F-%H%M%S).nft
```

## 八、重启验收清单

完成首次部署或重大修改后，应实际重启一次：

```bash
reboot
```

重启后使用新 SSH 连接检查：

```bash
systemctl is-enabled nftables.service docker.service fail2ban.service
systemctl --no-pager --full status nftables.service docker.service fail2ban.service
nft list table inet filter
docker ps
fail2ban-client ping
fail2ban-client status sshd
```

合格标准：

- `nftables.service`、`docker.service`、`fail2ban.service` 均为 `enabled`；
- `nftables.service` 显示 `active (exited)`，这是 oneshot 服务的正常状态；
- `docker.service`、`fail2ban.service` 显示 `active (running)`；
- `table inet filter` 存在，`input` 链为 `policy drop`；
- SSH 新连接可建立；
- Docker 容器和对外端口映射恢复；
- Fail2Ban 返回 `Server replied: pong`，且 sshd jail 正常。

## 九、总结

维护这套架构时，只需牢记四条原则：

1. **静态规则只写 `/etc/nftables.conf`。**
2. **Docker 规则交给 Docker，Fail2Ban 规则交给 Fail2Ban。**
3. **不要把 `nft list ruleset` 的完整输出写回 `/etc/nftables.conf`。**
4. **每次涉及 `flush ruleset` 的变更，都先检查语法、保留 SSH 救援路径，并在加载后重启 Docker 与 Fail2Ban。**

这样做的价值不只是“规则能够重启后保留”，而是让规则的来源、所有权、加载时机和故障边界都清晰可控。
