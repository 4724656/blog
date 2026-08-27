---
title: "Watchtower的通知配置"
description: "Watchtower 自动更新很好用，但默认通知太吵。本文记录如何通过 Telegram HTML 模板与静默参数，实现“仅在有容器更新时”才推送精简聚合消息。"
publishDate: "2026-08-27"
tags: ["Docker", "Watchtower", "Telegram", "运维"]
---

Watchtower 能自动更新容器，但默认通知体验极差：容器重启发一条、定时扫描没更新也发一条。

折腾了一圈，最终把它调校成理想的**“极简静默模式”**：

- 启动不通知、没更新不通知。
- 只有实际更新了镜像，才往 Telegram 推送**一条**排版清晰的聚合卡片。
- 更新后自动清理旧镜像，不占磁盘。

## 预期通知效果

实际推送效果类似以下文本框：

```text
🚀 Docker 更新完成
🖥 服务器: 你的服务器名称
📊 概览: 8 扫描 | 2 更新
📦 容器列表:
• watchtower (nickfedor/watchtower:latest)
• vaultwarden (vaultwarden/server:latest)
```

---

## 完整 Docker Compose 配置

```yaml
services:
  watchtower:
    image: nickfedor/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TZ: Asia/Shanghai

      # Telegram 推送配置（注意替换 API 和 chats ID）
      WATCHTOWER_NOTIFICATION_URL: "telegram://BOT_TOKEN@telegram?chats=CHAT_ID&parsemode=HTML"
      WATCHTOWER_NOTIFICATION_REPORT: "true"

      # 静默优化：关闭启动提示、隐藏默认标题、仅告警级别
      WATCHTOWER_NOTIFICATION_SKIP_TITLE: "true"
      WATCHTOWER_NO_STARTUP_MESSAGE: "true"
      WATCHTOWER_NOTIFICATIONS_LEVEL: "warn"

      # 聚合通知模板（仅在有 .Updated 时输出正文）
      WATCHTOWER_NOTIFICATION_TEMPLATE: |
        {{- if .Report -}}
        {{- with .Report -}}
        {{- if .Updated -}}
        🚀 <b>Docker 更新完成</b>
        🖥 服务器: 你的服务器名称
        📊 概览: {{len .Scanned}} 扫描 | {{len .Updated}} 更新
        📦 容器列表:
        {{- range .Updated }}
        • <code>{{ .Name }}</code> ({{ .ImageName }})
        {{- end }}
        {{- end -}}
        {{- end -}}
        {{- end -}}

    # 每小时整点检查一次，并自动清理旧镜像
    command: --schedule "0 0 * * * *" --cleanup
```
