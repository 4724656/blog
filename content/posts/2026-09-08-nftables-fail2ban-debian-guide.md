---
title: "VPS nftables + Fail2Ban 可靠部署指南（Debian 12+）"
description: "在 Debian VPS 上部署静态 nftables 主机防火墙与 Fail2Ban 动态封禁规则，兼顾 Docker 网络、重启持久化和 SSH 防锁门恢复。"
publishDate: "2026-09-08"
tags: ["Debian", "nftables", "Fail2Ban", "Docker", "VPS安全"]
---

# VPS nftables + Fail2Ban 可靠部署指南（Debian 12+）

> 目标：在新 VPS 上建立**静态 nftables 主机防火墙 + 官方 Fail2Ban nftables action**，避免重启后重复规则。  
> 适用：Debian 12/13，Docker 使用默认 `iptables-nft` 后端。

---

## 一、前置条件与约束

- 操作系统：Debian 12+（nftables 默认安装）
- 软件：`nftables`、`fail2ban`、（可选）`docker.io` 或 `docker-ce`
- 约束：
  - `/etc/nftables.conf` 只保存**静态主机防火墙规则**
  - Docker、Fail2Ban 的规则由各自服务**运行时动态创建**
  - **禁止**执行 `nft list ruleset > /etc/nftables.conf`

> **执行前先防锁门：** 确认当前 SSH 实际端口、云厂商安全组和防火墙已同步放行；保留 VPS Web Console / VNC 等救援入口。不要只保留一个 SSH 会话就直接应用包含 `flush ruleset` 的规则。

---

## 二、部署脚本（推荐方式）

保存为 `deploy-firewall.sh`，在新 VPS 上执行。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# =========================
# 0. 基础检查
# =========================

if [[ $EUID -ne 0 ]]; then
    echo "请以 root 或使用 sudo 执行此脚本"
    exit 1
fi

# =========================
# 1. 安装必要软件
# =========================

apt update
apt install -y nftables fail2ban

# =========================
# 2. 写入静态 nftables 配置
# =========================

cat > /etc/nftables.conf <<'EOF'
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # 回环接口
        iifname "lo" accept

        # 已建立连接及相关连接
        ct state established,related accept

        # 丢弃无效状态包
        ct state invalid drop

        # IPv4 ICMP
        ip protocol icmp icmp type echo-request limit rate 10/second burst 20 packets accept
        ip protocol icmp icmp type {
            destination-unreachable,
            time-exceeded,
            parameter-problem
        } accept

        # IPv6 ICMPv6（邻居发现等）
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

        # IPv6 Ping（可选）
        ip6 nexthdr icmpv6 icmpv6 type echo-request limit rate 10/second burst 20 packets accept

        # 服务端口：若 SSH 使用自定义端口，请将 22 替换为实际端口
        tcp dport 22 accept
        tcp dport 80 accept
        tcp dport 443 accept
    }

    chain forward {
        type filter hook forward priority filter; policy accept;
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
EOF

# 语法检查
nft -c -f /etc/nftables.conf

# =========================
# 3. 配置 Fail2Ban
# =========================

cat > /etc/fail2ban/jail.local <<'EOF'
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
EOF

# =========================
# 4. 启动服务
# =========================

# 开机自启：无论 Docker 是否运行，都应启用静态规则和 Fail2Ban
systemctl enable nftables
systemctl enable fail2ban

# 如果 Docker 正在运行，先停止它，避免 flush ruleset 清空运行时 Docker 规则
if systemctl is-active --quiet docker; then
    systemctl stop fail2ban
    systemctl stop docker.socket docker

    nft -f /etc/nftables.conf

    systemctl start docker
    systemctl start fail2ban
else
    systemctl start nftables
    systemctl restart fail2ban
fi

echo "=== 部署完成 ==="
echo "检查命令："
echo "  systemctl status nftables fail2ban"
echo "  nft list table inet filter"
echo "  fail2ban-client status sshd"
```

### 使用方法

```bash
chmod +x deploy-firewall.sh
sudo ./deploy-firewall.sh
```

---

## 三、部署后验证

### 1. 检查服务状态

```bash
systemctl status nftables
systemctl status fail2ban
```

- `nftables.service` 应为 `active (exited)`，这是 oneshot 服务的正常状态
- `fail2ban.service` 应为 `active (running)`

### 2. 检查 nftables 静态规则

```bash
nft list table inet filter
```

应看到：

```nft
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        iifname "lo" accept
        ct state established,related accept
        ct state invalid drop
        ...
        tcp dport 22 accept
        tcp dport 80 accept
        tcp dport 443 accept
    }
    ...
}
```

### 3. 检查 Fail2Ban 状态

```bash
fail2ban-client status sshd
```

此时可能显示 `Currently banned: 0`，`f2b-table` 尚未创建是正常的。

### 4. 手动测试封禁功能

```bash
# 封禁测试 IP（RFC 5737 文档地址，勿用真实 IP）
fail2ban-client set sshd banip 192.0.2.1

# 在完整规则集中搜索测试 IP
nft list ruleset | grep -A 20 -B 5 '192.0.2.1'

# 查看封禁列表
fail2ban-client get sshd banip

# 解封
fail2ban-client set sshd unbanip 192.0.2.1
```

应能在 `nft list ruleset` 输出和 `fail2ban-client get sshd banip` 中找到测试 IP。不同 Fail2Ban 与 nftables action 版本生成的表名、set 名、链名和拒绝动作可能不同，因此不应把固定的 `f2b-table` 结构当作唯一正确结果。

### 5. 如果已安装 Docker

```bash
docker ps
iptables -S
iptables -t nat -S
nft list ruleset
```

Docker 在 Debian 上通常经由 `iptables-nft` 兼容层维护运行时规则；最终 nftables 表与链名称会随 Docker、iptables 后端和系统版本变化。以容器网络、端口映射和上述规则查询正常为准，不应死盯某个固定 nft 表或 `DOCKER-*` 链名称。

---

## 四、重启验证

```bash
reboot
```

启动后检查：

```bash
systemctl is-enabled nftables
systemctl status nftables
nft list table inet filter
fail2ban-client status sshd
nft list table inet f2b-table
```

- `nftables.service` 应为 `enabled`
- `table inet filter` 应存在
- Docker 容器应正常运行（如已安装）
- Fail2Ban 应正常运行，首次封禁后可通过 `nft list ruleset` 搜索被封禁 IP 确认动态规则已创建

---

## 五、可选加固

### 1. SSH 加固

编辑 `/etc/ssh/sshd_config`：

```conf
# 禁用密码登录
PasswordAuthentication no
PubkeyAuthentication yes

# 禁用 root 登录（如有普通用户）
PermitRootLogin no

# 可选：更改 SSH 端口
# Port 2222
```

如果更改端口，同步修改：

- `/etc/nftables.conf` 中的 `tcp dport 22 accept` → `tcp dport 2222 accept`
- `/etc/fail2ban/jail.local` 中的 `port = ssh` → `port = 2222`

然后：

```bash
sshd -t        # 检查配置语法
systemctl restart sshd
```

### 2. 限制 Fail2Ban 只监控必要服务

确保 `/etc/fail2ban/jail.local` 中只启用需要的 jail，例如：

```ini
[sshd]
enabled = true
port    = ssh
filter  = sshd
action  = nftables[name=sshd, port=ssh, protocol=tcp]
```

不要启用不需要的 jail（如 `apache-auth`、`nginx-http-auth` 等），除非确实运行对应服务。

### 3. 日志监控

```bash
journalctl -u fail2ban -f
journalctl -u sshd -f
```

### 4. 定期查看封禁状态

```bash
fail2ban-client status
fail2ban-client status sshd
fail2ban-client get sshd banip
```

---

## 六、维护与修改

### 修改静态防火墙规则

```bash
nano /etc/nftables.conf
nft -c -f /etc/nftables.conf    # 语法检查
```

如需立即应用（且已安装 Docker）：

```bash
systemctl stop fail2ban
systemctl stop docker.socket docker

nft -f /etc/nftables.conf

systemctl start docker
systemctl start fail2ban
```

### 禁止执行的操作

- **不要**执行 `nft list ruleset > /etc/nftables.conf`
- **不要**手工编辑 Docker 创建的 `DOCKER-*` 链
- **不要**在 `/etc/nftables.conf` 中添加 `f2b-*` 或 `addr-set-*` 相关内容

---

## 七、故障排查

### 1. nftables 服务未启动

```bash
systemctl enable nftables
systemctl start nftables
systemctl status nftables
```

### 2. Fail2Ban 无法创建 f2b-table

```bash
journalctl -u fail2ban -n 50 --no-pager
```

检查是否有 `ERROR` 或 `nft` 相关错误。

### 3. Docker 网络异常

```bash
systemctl restart docker
docker ps
iptables -S
iptables -t nat -S
nft list ruleset
```

### 4. SSH 无法连接

- 确认当前 SSH 端口在 `/etc/nftables.conf` 中已放行
- 确认 `tcp dport 22` 或自定义端口存在
- 使用 VPS 控制台（Web Console / VNC）检查规则

---

## 八、架构说明

| 组件 | 规则类型 | 管理方式 | 配置文件 |
|------|---------|---------|---------|
| 主机防火墙 | 静态 | nftables.service | `/etc/nftables.conf` |
| Fail2Ban SSH 封禁 | 动态 | fail2ban.service | `/etc/fail2ban/jail.local` |
| Docker 网络/NAT | 动态 | docker.service | Docker 自动管理 |

重启后：

1. `nftables.service` 加载 `/etc/nftables.conf`（静态规则）
2. `docker.service` 启动，自动创建 `DOCKER-*` 链和 NAT 规则
3. `fail2ban.service` 启动，首次封禁时自动创建 `f2b-table`

三者职责清晰，互不干扰，不会产生重复规则。

---

## 九、参考

- Debian Handbook - Firewall / nftables
- Fail2Ban 官方文档 - Action Configuration
- Docker 官方文档 - Firewall with nftables / iptables