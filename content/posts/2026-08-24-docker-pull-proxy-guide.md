---
title: "Docker Pull 代理配置指南：QNAP、iStoreOS 与 Debian/Ubuntu"
description: "在网络受限或 Docker Hub 访问不稳定的环境中，仅在当前终端设置代理通常是不够的。真正负责拉取镜像的是 Docker 守护进程 dockerd。本文整理 QNAP、iStoreOS/OpenWrt 和 Debian/Ubuntu 三类常见环境中的 Docker 镜像拉取代理方案与排障流程。"
publishDate: "2026-08-24"
tags: ["docker", "network", "infra"]
---

> **指南摘要**：在网络受限或 Docker Hub 访问不稳定的环境中，仅在当前终端设置代理通常是不够的。真正负责拉取镜像的是 Docker 守护进程 `dockerd`，因此必须让 `dockerd` 进程本身获得代理配置。本文整理了 QNAP Container Station、iStoreOS / OpenWrt 与 Debian/Ubuntu 等系统中的 Docker 拉取镜像代理配置方案。
> 
> *注：文中的代理地址统一使用 `http://192.168.2.2:7890`，请根据实际网络环境替换为你的代理服务器地址和端口。本文只配置 Docker 守护进程拉取镜像时的代理，不会自动让容器内的应用通过代理访问网络。*

---

## 1. 先理解 Docker 代理的作用范围

| 网络请求 | 负责进程 | 本文配置是否影响 |
| :--- | :--- | :--- |
| `docker pull` 拉取镜像 | `dockerd` | **是** |
| 构建镜像时下载依赖 | `dockerd` / BuildKit | 部分影响 |
| 容器内应用访问外网 | 容器内应用 | 否 |
| Docker CLI 访问 Docker API | Docker CLI | 不一定 |

通过 systemd、UCI 或 QNAP 启动脚本设置的代理，主要作用对象是 `dockerd`。即使执行 `export HTTP_PROXY=http://192.168.2.2:7890`，也不代表 Docker 守护进程已经使用代理，必须把配置放到 Docker 服务实际读取的位置。

---

## 2. 准备代理服务

本文以 Mihomo、Clash 或其他兼容 HTTP CONNECT 的代理服务为例。假设代理服务运行在 `192.168.2.2:7890`。

### 检查代理端口
```shell
ss -lntp | grep 7890
# 没有 ss 时：
netstat -lntp | grep 7890
```

在 OpenWrt / iStoreOS 上，监听地址通常应允许局域网设备访问，例如 `0.0.0.0:7890` 或 `192.168.2.2:7890`。如果只监听 `127.0.0.1:7890`，其他设备无法通过局域网访问。

### 测试代理出站连接
```shell
curl -v -x http://192.168.2.2:7890 https://www.google.com --max-time 20
curl -v -x http://192.168.2.2:7890 https://registry-1.docker.io/v2/ --max-time 20
```

访问 Registry 返回 `401 Unauthorized` 通常不表示代理失败，而是说明已经成功到达 Docker Registry，只是请求没有携带认证信息。

> **提示**：如果 Mihomo 同时提供纯 HTTP 端口和 mixed-port，建议优先使用纯 HTTP 代理端口，例如 `http://192.168.2.2:8080`。Docker 通常通过 HTTP 代理建立 HTTPS CONNECT 连接，代理地址仍应写成 `http://代理地址:端口`，不要因为目标是 HTTPS 就写成 `https://192.168.2.2:7890`，除非代理明确提供 HTTPS 代理协议。

---

## 3. QNAP Container Station

QNAP 的 Container Station 不一定使用标准的 systemd Docker 服务。不同 QTS、QuTS hero 和 Container Station 版本的启动链路可能不同。部分版本可以通过 `/lib/container-station/ld-wrapper.sh` 向 Container Station 启动的 Docker 进程注入代理环境变量，但必须先确认文件存在且实际 `dockerd` 会经过该包装脚本。

### 备份并检查文件
```shell
cp -a /lib/container-station/ld-wrapper.sh \
 /lib/container-station/ld-wrapper.sh.bak.$(date +%Y%m%d-%H%M%S)
ls -l /lib/container-station/ld-wrapper.sh
grep -nE 'dockerd|docker.sock' /lib/container-station/ld-wrapper.sh
```

### 编辑文件
在 `#!/bin/sh` 下一行加入：
```shell
export HTTP_PROXY="http://192.168.2.2:7890"
export HTTPS_PROXY="http://192.168.2.2:7890"
export NO_PROXY="localhost,127.0.0.1,172.16.0.0/12,192.168.0.0/16,.local"
```

### 重启 Container Station
```shell
/etc/init.d/container-station.sh restart
```
如果命令不存在，应通过 Container Station 管理界面重启，或先确认实际启动脚本位置。

### 验证 dockerd 是否获得代理
```shell
for pid in $(pidof dockerd); do
 echo "--- PID: $pid ---"
 tr '\0' '\n' < "/proc/$pid/environ" \
 | grep -iE '^(HTTP|HTTPS|NO)_PROXY='
done
```
没有 `pidof` 时可先用 `ps w | grep '[d]ockerd'` 查找进程。确认环境变量后测试：
```shell
docker pull hello-world
docker pull busybox
```

> **注意**：这种方法不是厂商正式的永久配置接口，Container Station 或 QNAP 升级可能覆盖修改，启动脚本结构也可能变化。修改前应备份，升级后重新检查；如果无效，应检查实际 `dockerd` 的启动命令和父进程。

---

## 4. iStoreOS / OpenWrt

iStoreOS 上的 Docker 通常通过 OpenWrt 的 UCI 配置和初始化脚本管理，常见流程是 `/etc/config/dockerd` -> `/etc/init.d/dockerd` -> `/tmp/dockerd/daemon.json` -> `dockerd`。不应默认照搬 Debian 的 systemd 配置，也不应仅修改当前 Shell 的环境变量。

### 检查当前配置和脚本
```shell
uci show dockerd
cat /etc/config/dockerd
sed -n '1,260p' /etc/init.d/dockerd
```
重点确认初始化脚本是否读取 `proxies` 配置段，以及是否生成 `/tmp/dockerd/daemon.json`。

### 写入代理设置
如果系统已有 `dockerd.proxies` 配置段，可设置：
```shell
uci set dockerd.proxies.http_proxy='http://192.168.2.2:7890'
uci set dockerd.proxies.https_proxy='http://192.168.2.2:7890'
uci set dockerd.proxies.no_proxy='localhost,127.0.0.1,172.16.0.0/12,192.168.0.0/16,.local'
uci commit dockerd
uci show dockerd.proxies
```

只有在当前版本初始化脚本明确支持 `proxies` 段时，才创建它：
```shell
uci set dockerd.proxies=proxies
uci set dockerd.proxies.http_proxy='http://192.168.2.2:7890'
uci set dockerd.proxies.https_proxy='http://192.168.2.2:7890'
uci set dockerd.proxies.no_proxy='localhost,127.0.0.1,172.16.0.0/12,192.168.0.0/16,.local'
uci commit dockerd
```
如果初始化脚本不识别这些字段，配置不会生效，应以当前系统脚本支持的格式为准。

不要无条件删除已有 `daemon.json`。建议先备份：
```shell
cp -a /etc/docker/daemon.json \
 /etc/docker/daemon.json.bak.$(date +%Y%m%d-%H%M%S) \
 2>/dev/null || true
```

### 重启并验证
```shell
/etc/init.d/dockerd restart
/etc/init.d/dockerd status
cat /tmp/dockerd/daemon.json
```

预期生成的配置文件 `daemon.json` 中应包含代理配置：
```json
{
 "proxies": {
 "http-proxy": "http://192.168.2.2:7890",
 "https-proxy": "http://192.168.2.2:7890",
 "no-proxy": "localhost,127.0.0.1,172.16.0.0/12,192.168.0.0/16,.local"
 }
}
```
*注：实际文件可能还有其他 Docker 配置，不要为了匹配示例而直接覆盖整个文件。*

### 测试拉取与日志查看
```shell
docker pull hello-world
docker pull busybox
```
如果代理测试成功但拉取失败，依次确认生成的 `daemon.json`、`dockerd` 是否已重启，并查看日志：
```shell
logread | grep -iE 'docker|dockerd'
```
同时可更换 Mihomo 节点，测试其他镜像，确认代理能稳定处理 Registry 的 HTTPS、认证和大文件分层下载，并检查代理服务是否允许来自 iStoreOS 的局域网连接。

---

## 5. Debian、Ubuntu 等 systemd 系统

推荐使用 systemd drop-in，而不是直接修改发行版自带的 unit 文件：

```shell
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/proxy.conf
```

### 写入配置
```ini
[Service]
Environment="HTTP_PROXY=http://192.168.2.2:7890"
Environment="HTTPS_PROXY=http://192.168.2.2:7890"
Environment="NO_PROXY=localhost,127.0.0.1,::1,172.16.0.0/12,192.168.2.0/24,.local"
```

> **注意**：`HTTP_PROXY` 和 `HTTPS_PROXY` 用于 Docker 守护进程访问 HTTP/HTTPS 目标，`NO_PROXY` 中应填写主机名、域名、IP、网段或端口，不要写带协议头的 URL。例如错误写法是 `https://registry.cn-hangzhou.aliyuncs.com`，正确写法是 `registry.cn-hangzhou.aliyuncs.com`；只有确实希望绕过代理时才加入该域名。

### 重新加载并重启
```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 验证配置与进程环境
```shell
systemctl show --property=Environment docker
systemctl cat docker

for pid in $(pidof dockerd); do
 echo "--- PID: $pid ---"
 tr '\0' '\n' < "/proc/$pid/environ" \
 | grep -iE '^(HTTP|HTTPS|NO)_PROXY='
done
```

### 测试拉取并查看日志
```shell
docker pull hello-world
docker pull busybox
sudo journalctl -u docker -n 100 --no-pager
```

---

## 6. 镜像加速器与代理

| 配置项 | 作用 |
| :--- | :--- |
| `HTTP_PROXY` / `HTTPS_PROXY` | 让 `dockerd` 通过代理访问 Registry |
| `registry-mirrors` | 将部分镜像拉取请求转发到镜像加速服务 |
| `dns` | 指定容器使用的 DNS 服务器 |

镜像加速器不能随便填写普通 Registry 地址。云厂商专用的 Docker Hub pull-through cache 地址才适合作为 `registry-mirrors`。没有确认可用的专用地址时，可以不配置，只使用代理。

标准 Docker 主机可在 `/etc/docker/daemon.json` 中合并配置：
```json
{
 "registry-mirrors": [
 "https://你的专用镜像加速地址"
 ]
}
```
已有其他配置时应合并 JSON，而不是覆盖原文件。修改后重启并查看：
```shell
sudo systemctl restart docker
docker info | grep -A5 -i "Registry"
```
建议先只配置代理并验证 `docker pull`，再单独添加镜像加速器，避免增加排障复杂度。iStoreOS 应优先使用 `/etc/config/dockerd` 和 UCI。

---

## 7. 为容器设置代理

前面的配置只影响 Docker 守护进程，通常用于 `docker pull`、`docker push`、`docker build`，不会自动让容器内应用使用代理。

### Docker Compose 示例
```yaml
services:
 app:
 image: alpine:latest
 environment:
 HTTP_PROXY: http://192.168.2.2:7890
 HTTPS_PROXY: http://192.168.2.2:7890
 NO_PROXY: localhost,127.0.0.1,172.16.0.0/12,192.168.0.0/16,.local
```

### 单个容器运行示例
```shell
docker run --rm \
 -e HTTP_PROXY=http://192.168.2.2:7890 \
 -e HTTPS_PROXY=http://192.168.2.2:7890 \
 -e NO_PROXY=localhost,127.0.0.1,172.16.0.0/12,192.168.0.0/16,.local \
 alpine:latest
```
> **要求**：容器必须能够访问代理地址，代理服务必须监听局域网地址，Docker 网络和防火墙不能阻断访问，`NO_PROXY` 也应按容器实际访问的内网地址配置。

---

## 8. DNS 配置与验证

DNS 问题有时会被误认为代理问题。Docker 宿主机能访问代理，不代表容器内 DNS 一定正常。

标准 Docker 主机可在 `daemon.json` 中配置：
```json
{
 "dns": [
 "223.5.5.5",
 "1.1.1.1"
 ]
}
```
如果同时配置镜像加速器，应与已有 JSON 合并。修改后重启 Docker。更可靠的验证方式是直接检查容器内部的解析状态：
```shell
docker run --rm busybox cat /etc/resolv.conf
docker run --rm busybox nslookup registry-1.docker.io
# Alpine 环境测试：
docker run --rm alpine getent hosts registry-1.docker.io
```

---

## 9. 统一排障流程

当 `docker pull` 失败时，建议依次检查：

1. **代理端口状态**：确认代理服务端口是否正常监听，以及是否允许 Docker 主机所在的网段访问。
2. **连接可行性测试**：确认宿主机是否能通过代理正常建立连接：
   ```shell
   curl -I -x http://192.168.2.2:7890 https://registry-1.docker.io/v2/
   ```
3. **守护进程环境变量加载**：
   - QNAP：检查 `/proc/<pid>/environ`。
   - iStoreOS：检查 UCI 配置和生成的 `/tmp/dockerd/daemon.json`。
   - systemd：运行 `systemctl show --property=Environment docker`。
4. **服务运行日志**：查看 Docker 服务状态和相关日志。
5. **极简测试**：使用最基础的镜像来进行连接验证：
   ```shell
   docker pull hello-world
   docker pull busybox
   ```

如果简单镜像可以拉取而特定镜像失败，可能是镜像名称错误、仓库需要登录、镜像层较大、代理节点对长连接或大文件不稳定，或者镜像来源是 GHCR、Quay 或私有 Registry。

如果 curl 通过代理成功但 docker pull 仍超时，可更换 Mihomo 节点或代理协议，优先使用纯 HTTP 代理端口，检查代理日志中的 Registry 请求，以及 TLS、认证和大文件下载情况。

---

## 10. 三种系统的配置速查表

| 系统 | 配置位置 | 重启命令 | 验证方式 |
| :--- | :--- | :--- | :--- |
| QNAP Container Station | `/lib/container-station/ld-wrapper.sh`（版本相关） | `/etc/init.d/container-station.sh restart` | 检查 `/proc/<pid>/environ` |
| iStoreOS / OpenWrt | UCI：`/etc/config/dockerd` | `/etc/init.d/dockerd restart` | `uci show`、`/tmp/dockerd/daemon.json` |
| Debian / Ubuntu | `/etc/systemd/system/docker.service.d/proxy.conf` | `systemctl daemon-reload && systemctl restart docker` | `systemctl show`、`/proc/<pid>/environ` |

---

## 结语

Docker 代理配置的核心不是修改 Docker CLI，而是确认 dockerd 的启动方式和实际配置来源：QNAP 通过 Container Station 启动包装脚本注入环境变量，iStoreOS 通过 UCI 和 `/etc/init.d/dockerd` 管理配置，Debian、Ubuntu 等系统通过 systemd drop-in 配置 Docker 服务。

如果代理配置后仍然无法拉取镜像，应按照“代理端口 -> 宿主机代理访问 -> dockerd 实际配置 -> Docker 日志 -> 镜像仓库”的顺序排查。

最重要的原则是：**不要只看配置文件是否写入，而要验证实际运行中的 dockerd 是否已经加载配置。**
