---
title: "VPS 环境下 gog (Google Workspace CLI) 配置与 OpenClaw 注入完美指南"
description: "本文详细记录了在无头 VPS 服务器上，如何安全上传 client_secret JSON、通过 manual 模式完成 Google OAuth 授权、使用 file keyring 加密并完美注入 OpenClaw Gateway 运行环境的完整步骤。"
publishDate: "2026-08-24"
tags: ["infra", "gog", "openclaw"]
---

> **指南摘要**：本篇教程提供了一套完整的、适合无头 VPS 终端环境的 Google Workspace CLI (`gog`) 的 OAuth 认证与 OpenClaw 注入方案，确保在不依赖本地 GUI 浏览器的前提下，安全、顺畅地打通日程、邮件及云盘的 AI 访问权限。

---

## 1. 在本机安全上传 JSON

从 Windows PowerShell 上传到 VPS，替换用户名和主机：

```shell
scp "F:\下载\client_secret_*.json" root@vps_ip:/root/
```

或在 WSL 中：

```shell
scp "/mnt/f/下载/client_secret_*.json" \
 root@vps_ip:/root/
```

完成后，**建议删除 Windows 下载目录中这份 JSON**，或存入加密密码库。

---

## 2. 登录 VPS，创建私有目录

```shell
ssh root@vps_ip

mkdir -p /root/secure
chmod 700 /root/secure

mv /root/client_secret_*.json /root/secure/gog-client.json
chmod 600 /root/secure/gog-client.json

ls -l /root/secure/gog-client.json
```

预期权限：
```text
-rw------- 1 root root ... /root/secure/gog-client.json
```

---

## 3. 安装 `gog`

如果 VPS 没有安装 Go：

```shell
sudo apt update
sudo apt install -y golang-go
```

安装 `gog` 客户端：

```shell
go install github.com/openclaw/gogcli/cmd/gog@latest
```

将 Go bin 目录加入当前 shell：

```shell
export PATH="$PATH:/root/go/bin"
gog --version
```

为后续 SSH 登录永久追加 PATH 环境变量：

```shell
grep -qxF 'export PATH="$PATH:/root/go/bin"' /root/.bashrc || \
 echo 'export PATH="$PATH:/root/go/bin"' >> /root/.bashrc
```

> **提示**：`gog` 官方支持从下载的 Desktop OAuth JSON 存储 client credentials；安装后可用 `gog --version` 验证。

---

## 4. 设置 VPS 专属 keyring 密码

```shell
export GOG_KEYRING_BACKEND=file
read -rsp 'Set VPS gog keyring passphrase: ' GOG_KEYRING_PASSWORD
echo
export GOG_KEYRING_PASSWORD
```

> **注意**：输入时屏幕不会显示任何字符，这是正常现象。此密码只用来加密 VPS 本地的 `gog` token / keyring；**它不是 Google 账户密码，也不是 OAuth client secret**。

---

## 5. 导入 OAuth Client JSON

```shell
gog auth credentials /root/secure/gog-client.json
```

如果看到导入成功的确认提示，即可继续下一步。

---

## 6. 在 VPS 完成手动 OAuth 授权

```shell
gog auth add xxxxxx@gmail.com \
 --services gmail,calendar,drive \
 --manual \
 --force-consent
```

此时，VPS 终端会打印一个 Google 授权 URL：

1. 将该 URL 复制到本地的 Windows 浏览器中。
2. 登录已被加入 Google Cloud OAuth 测试范围的 Google 账号。
3. 点击允许并授予所需服务权限。
4. 最终浏览器会发生跳转，并提示无法打开网页（这完全是正常的）：`http://127.0.0.1:端口/oauth2/callback?...`。
5. 此时，从浏览器地址栏复制出完整的跳转 URL。
6. **只粘贴回 VPS SSH 终端**，不要发送到任何聊天窗口。
7. 等待 `gog` 显示已绑定邮箱及 `calendar, drive, gmail` 服务。

> **原理说明**：`--manual` 参数正是 headless 环境下复制回调 URL 授权的标准方式；必须在生成 URL 的同一台 VPS、同一个 `gog` home/client 中粘贴完成。

---

## 7. 验证授权

```shell
gog auth list --check
gog auth doctor --check

gog calendar events --today
gog gmail search 'newer_than:7d' --max 5
gog drive ls --max 5
```

前三个服务均正常返回数据即代表完成。
> **提示**：如果返回 `No events` 只是代表当天没有日程，不属于报错。

---

## 8. 注入 OpenClaw 环境

把 **第 4 步为这台 VPS 设置的同一个 keyring 密码** 写入配置文件：

```shell
mkdir -p /root/.openclaw
chmod 700 /root/.openclaw

nano /root/.openclaw/.env
```

写入以下配置：

```shell
GOG_KEYRING_BACKEND=file
GOG_KEYRING_PASSWORD=第4步设置的VPS专属keyring密码
```

保存文件并严格限制其访问权限：

```shell
chmod 600 /root/.openclaw/.env
```

进行安全性检查，确认配置中不会直接在终端历史中泄露密码：

```shell
sed -E 's/^(GOG_KEYRING_PASSWORD)=.*/\1=***REDACTED***/' \
 /root/.openclaw/.env
```

> **权限提示**：OpenClaw Gateway 会读取 `~/.openclaw/.env`；对于 root 用户就是 `/root/.openclaw/.env`。该文件的权限应为 `600`。

---

## 9. 重启 Gateway 并进行最终检查

```shell
openclaw gateway restart
openclaw skills check
```

最后让 OpenClaw 发起一个只读日程查询测试来进行验证：

> “查看我今天的 Google Calendar 日程，不要创建或修改任何事件。”

当看到日程列表正常返回，且没有出现任何 keyring 密码提示或 D-Bus 超时报错时，说明 VPS 侧环境已经完美调通了！
