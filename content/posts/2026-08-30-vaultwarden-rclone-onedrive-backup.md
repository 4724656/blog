---
title: 用 rclone 加密备份 Docker Vaultwarden 到 OneDrive
description: 在 Debian VPS 上为 Docker Vaultwarden 建立自动备份：生成一致性 SQLite 快照、打包关键数据、使用 rclone crypt 客户端加密并上传到 OneDrive。
publishDate: 2026-08-30
tags:
  - rclone
  - vaultwarden
  - onedrive
  - docker
  - backup
draft: false
---

Vaultwarden 保存的是密码、TOTP 秘钥、附件和服务配置，不能只依赖 VPS 磁盘或单一 Docker 卷。本文记录如何在 Debian VPS 上建立一条自动化备份链路：

```text
Vaultwarden Docker 容器
→ 内置 backup 生成一致性 SQLite 快照
→ 打包数据库、RSA 密钥及可选附件
→ rclone crypt 在 VPS 本地加密
→ 直接上传到 OneDrive
→ cron 每天自动执行
```

本文的关键原则：

- 不直接复制正在运行的 `db.sqlite3`
- 不把密码库归档以明文上传到 OneDrive
- 不使用 `rclone sync` 做备份上传，避免误删远端历史
- 上传成功后才清理过期的本地文件
- 配置完成后，Windows 不参与日常备份；数据由 VPS 直接上传到 OneDrive

## 1. 确认 Vaultwarden 数据目录

如果 Vaultwarden 通过 Docker 运行，先从容器实际挂载中确认宿主机数据目录，不要凭印象猜路径：

```bash
docker inspect vaultwarden \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

示例：

```text
/opt/apps/vaultwarden/data -> /data
```

查看数据目录：

```bash
ls -lah /opt/apps/vaultwarden/data
```

常见关键内容包括：

```text
db.sqlite3
db.sqlite3-wal
db.sqlite3-shm
rsa_key.pem
attachments/
sends/
config.json
```

实际存在什么就备份什么。`attachments/`、`sends/` 和 `config.json` 在不同部署中可能不存在。

## 2. 使用 Vaultwarden 内置命令生成数据库快照

运行中的 SQLite 数据库如果处于 WAL 模式，会存在：

```text
db.sqlite3-wal
db.sqlite3-shm
```

因此不应把在线使用中的 `db.sqlite3` 当作唯一备份来源。Vaultwarden 内置命令会创建一致性的数据库快照：

```bash
docker exec vaultwarden /vaultwarden backup
```

成功时会得到类似输出：

```text
Backup to 'data/db_YYYYMMDD_HHMMSS.sqlite3' was successful
```

快照会出现在 Vaultwarden 的宿主机数据目录中，例如：

```text
/opt/apps/vaultwarden/data/db_YYYYMMDD_HHMMSS.sqlite3
```

查找最新快照：

```bash
LATEST_DB="$(
  find /opt/apps/vaultwarden/data \
    -maxdepth 1 \
    -type f \
    -name 'db_*.sqlite3' \
    -printf '%T@ %p\n' |
  sort -nr |
  head -n 1 |
  cut -d' ' -f2-
)"

printf '%s\n' "$LATEST_DB"
```

## 3. 配置 rclone 与 OneDrive

安装较新的 rclone 版本后，先配置普通 OneDrive remote：

```bash
rclone config
```

建议建立：

```text
onedrive:
```

对于无图形界面的 VPS，OneDrive 的首次 OAuth 授权需要借用一台有浏览器的设备完成一次登录；授权 token 最终保存在 VPS 的 rclone 配置中。之后的上传和定时备份均由 VPS 独立执行，不需要 Windows 开机或参与文件传输。

验证 OneDrive remote：

```bash
rclone listremotes
rclone lsd onedrive:
```

## 4. 建立 rclone crypt 加密层

普通 `onedrive:` 不应直接作为 Vaultwarden 备份目标。应在它上面创建一个 `crypt` remote，例如：

```text
vaultcrypt:
```

其底层路径设为：

```text
onedrive:EncryptedBackups
```

后续脚本只使用：

```text
vaultcrypt:Vaultwarden
```

不要直接使用：

```text
onedrive:EncryptedBackups
```

rclone 会在 VPS 本地完成文件内容加密，默认还会加密文件名和子目录名。OneDrive 网页中看到乱码文件名属于正常现象；通过 `vaultcrypt:` 才能看到解密后的真实名称。

测试链路：

```bash
printf 'rclone crypt upload test %s\n' "$(date -u +'%FT%TZ')" \
  > /tmp/rclone-upload-test.txt

rclone copyto --progress \
  /tmp/rclone-upload-test.txt \
  vaultcrypt:ConnectionTest/rclone-upload-test.txt

rclone lsl vaultcrypt:ConnectionTest

rm -f /tmp/rclone-upload-test.txt
```

## 5. 自动备份脚本

创建脚本目录、备份目录和日志目录：

```bash
install -d -m 700 /opt/scripts
install -d -m 700 /opt/backups/vaultwarden
install -d -m 700 /var/log/vaultwarden-backup
```

创建 `/opt/scripts/backup-vaultwarden.sh`：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
umask 077

CONTAINER="vaultwarden"
VW_DATA="/opt/apps/vaultwarden/data"
LOCAL_DIR="/opt/backups/vaultwarden"
REMOTE="vaultcrypt:Vaultwarden"
KEEP_DAYS=14

STAMP="$(date -u +'%Y-%m-%dT%H-%M-%SZ')"
STAGE="${LOCAL_DIR}/stage-${STAMP}"
ARCHIVE="${LOCAL_DIR}/vaultwarden-${STAMP}.tar.gz"

log() {
  printf '%s %s\n' "$(date -u +'%FT%TZ')" "$*"
}

cleanup_stage() {
  rm -rf "$STAGE"
}
trap cleanup_stage EXIT

require_file() {
  if [ ! -f "$1" ]; then
    log "ERROR: required file is missing: $1"
    exit 1
  fi
}

log "Starting Vaultwarden backup"

require_file "$VW_DATA/rsa_key.pem"

mkdir -p "$STAGE"

docker exec "$CONTAINER" /vaultwarden backup

LATEST_DB="$(
  find "$VW_DATA" -maxdepth 1 -type f -name 'db_*.sqlite3' -printf '%T@ %p\n' |
    sort -nr |
    head -n 1 |
    cut -d' ' -f2-
)"

if [ -z "${LATEST_DB:-}" ] || [ ! -f "$LATEST_DB" ]; then
  log "ERROR: Vaultwarden did not create a database snapshot"
  exit 1
fi

cp -a "$LATEST_DB" "$STAGE/db.sqlite3"
cp -a "$VW_DATA"/rsa_key* "$STAGE/"

if [ -d "$VW_DATA/attachments" ]; then
  cp -a "$VW_DATA/attachments" "$STAGE/"
fi

if [ -d "$VW_DATA/sends" ]; then
  cp -a "$VW_DATA/sends" "$STAGE/"
fi

if [ -f "$VW_DATA/config.json" ]; then
  cp -a "$VW_DATA/config.json" "$STAGE/"
fi

tar -C "$LOCAL_DIR" -czf "$ARCHIVE" "$(basename "$STAGE")"

if ! tar -tzf "$ARCHIVE" >/dev/null; then
  log "ERROR: archive integrity check failed: $ARCHIVE"
  exit 1
fi

log "Uploading $(basename "$ARCHIVE")"
rclone copyto --retries 3 --low-level-retries 10 \
  "$ARCHIVE" \
  "$REMOTE/$(basename "$ARCHIVE")"

if ! rclone lsf "$REMOTE" | grep -Fqx "$(basename "$ARCHIVE")"; then
  log "ERROR: uploaded archive was not found through crypt remote"
  exit 1
fi

# 删除远端超过 KEEP_DAYS 天的 vaultwarden-*.tar.gz
rclone delete "$REMOTE" \
  --include 'vaultwarden-*.tar.gz' \
  --min-age "${KEEP_DAYS}d"

find "$LOCAL_DIR" -maxdepth 1 -type f -name 'vaultwarden-*.tar.gz' \
  -mtime +"$KEEP_DAYS" -print -delete

find "$VW_DATA" -maxdepth 1 -type f -name 'db_*.sqlite3' \
  -mtime +"$KEEP_DAYS" -print -delete

log "SUCCESS: $(basename "$ARCHIVE")"
```

赋予权限：

```bash
chmod 700 /opt/scripts/backup-vaultwarden.sh
```

先检查语法：

```bash
bash -n /opt/scripts/backup-vaultwarden.sh
```

再手动运行一次并查看日志：

```bash
/opt/scripts/backup-vaultwarden.sh 2>&1 | tee -a /var/log/vaultwarden-backup/backup.log
```

成功时应出现：

```text
Starting Vaultwarden backup
Uploading vaultwarden-...
SUCCESS: vaultwarden-...
```

验证远端备份：

```bash
rclone lsl vaultcrypt:Vaultwarden
```

## 6. 设置 cron 自动运行

确认手动执行成功后，编辑 root 的 crontab：

```bash
crontab -e
```

每天凌晨 03:20 执行：

```cron
20 3 * * * /opt/scripts/backup-vaultwarden.sh >> /var/log/vaultwarden-backup/backup.log 2>&1
```

确认已生效：

```bash
crontab -l
```

查看日志：

```bash
tail -n 50 /var/log/vaultwarden-backup/backup.log
```

检查 cron 服务：

```bash
systemctl status cron --no-pager
```

需要注意：cron 使用哪个用户运行，就要能读取该用户的 rclone 配置。本文以 root 配置 rclone，并使用 root 的 crontab，因此配置文件位于：

```text
/root/.config/rclone/rclone.conf
```

## 7. 恢复验证

上传成功不等于备份一定可恢复。恢复测试应从加密 remote 下载，而不是直接从 OneDrive 网页下载密文文件。

先列出远端归档：

```bash
rclone lsl vaultcrypt:Vaultwarden
```

下载指定备份到隔离目录：

```bash
install -d -m 700 /opt/restore/vaultwarden

rclone copyto \
  vaultcrypt:Vaultwarden/vaultwarden-YYYY-MM-DDTHH-MM-SSZ.tar.gz \
  /opt/restore/vaultwarden/vaultwarden-YYYY-MM-DDTHH-MM-SSZ.tar.gz
```

检查归档是否可读取：

```bash
tar -tzf /opt/restore/vaultwarden/vaultwarden-YYYY-MM-DDTHH-MM-SSZ.tar.gz
```

至少应看到：

```text
.../db.sqlite3
.../rsa_key.pem
```

如果 Vaultwarden 使用过附件，还应看到：

```text
.../attachments/
```

不要仅因为 OneDrive 上显示文件存在就删除本地副本。应定期进行独立的恢复演练。

## 8. 必须保护的恢复材料

完成自动化后，还必须保护以下内容：

- OneDrive 账号访问权
- `vaultcrypt:` 的加密密码
- `vaultcrypt:` 的 salt
- `/root/.config/rclone/rclone.conf`
- Vaultwarden 的 Docker Compose / Dockge stack 配置
- `.env` 中的部署配置与密钥

rclone 配置文件含有 OneDrive token 和 crypt 参数，应限制权限：

```bash
chmod 600 /root/.config/rclone/rclone.conf
```

不要把 crypt 密码与 salt 只存放在 Vaultwarden 中。因为 Vaultwarden 恰好是正在备份的对象，恢复材料还应在独立、安全的位置留存一份。

## 总结

完成后，日常流程不再需要人工下载、压缩或上传：

```text
cron
→ Vaultwarden backup
→ tar.gz
→ rclone crypt
→ OneDrive
```

关键点不是“文件上传成功”，而是：

1. 数据库快照来自 Vaultwarden 内置 backup。
2. 数据离开 VPS 前已完成客户端加密。
3. 数据库、RSA 密钥和未来的附件会一并保存。
4. 不使用危险的 `rclone sync` 覆盖远端历史。
5. 定期验证可以下载、解密和读取归档。
