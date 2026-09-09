---
title: fileserver：一个单文件就够用的自建文件服务器（Go）
published: 2026-09-09
updated: 2026-09-09
description: 用一个 Go 编译的二进制文件在服务器 / NAS / 路由器上自建文件服务：上传下载、限时限次分享链接、一键自更新，零运行时依赖。
image: ./fileserver-cover.png
tags: [Go, 自托管, 文件服务, NAS, 开源]
category: 运维教程
draft: false
pinned: false
slug: fileserver-single-binary
author: 世琰
---

> 本文档面向想在服务器 / NAS / 路由器上自建文件分享服务的用户，涵盖功能介绍、三种部署方式与安全设计。
> **适用系统：** Linux（amd64 / arm64 / armv7 / armv5 / mipsle / 386）/ Windows / macOS
> **适用平台：** 云服务器 / 本地虚拟机 / NAS / 软路由 / Docker

---

## 一、为什么要再造一个文件服务器

网盘限速、第三方工具要装 Docker 拉一堆镜像、老 NAS 上连 Node 都跑不动——fileserver 就是为了解决这些场景：**一个可执行文件，拷过去就能跑**。

它的定位很简单：

- **单个二进制文件**：Go 编写、零第三方库，前端页面经 `go:embed` 直接编译进二进制，无任何运行时依赖
- **8 个平台开箱即用**：从 x86 云服务器到 mipsle 架构的老路由器，总有一个包能跑
- **核心功能齐全**：目录管理、流式上传、带 token 的限时限次分享链接、一键自更新

::github{repo="Zlion-Y/fileserver"}

---

## 二、功能总览

| 功能 | 说明 |
|------|------|
| 文件管理 | 上传（拖拽 / 多文件 / 大文件流式直写）、删除、重命名 / 移动、新建文件夹 |
| 搜索排序 | 全盘递归文件名搜索、按名称 / 大小 / 时间排序、按类型筛选 |
| 分享链接 | 有效期（1 小时 ~ 30 天 / 永久 / 自定义）+ 下载次数（1/5/20/不限）双限制 |
| 分享密码 | PBKDF2 哈希校验，密码 AES-GCM 加密存储，管理端可查看 |
| 直链 / 确认页 | 直链打开即下载；确认页模式先展示文件信息再计数 |
| 分享管理 | 筛选、批量撤销、行内快速编辑有效期 / 密码 / 访问方式 |
| 访问日志 | 每条分享记录最近 50 次访问（IP / UA / 成败原因） |
| 一键自更新 | 自动下载对应架构二进制 → 校验 → 原子替换 → 自动重启 |
| 安全与会话 | 查看活跃会话、远程踢出、网页改密立即生效并踢出其他会话 |
| 断点续传 | HTTP Range 支持，且续传须持授权 ticket，伪造 Range 无法绕过限次 |
| 防爆破 | 同一 IP 5 分钟内认证失败 10 次自动封禁 15 分钟 |
| 定时清理 | 过期分享记录自动清理，保留时长可视化配置 |

---

## 三、快速开始

### 3.1 直接运行

从 [Releases](https://github.com/Zlion-Y/fileserver/releases) 下载对应架构的二进制，赋予执行权限后运行：

```bash
chmod +x fileserver

./fileserver --port 16666 --dir /data/files --user admin --pass 你的密码
```

打开 `http://服务器IP:16666` 即可使用。不设 `--user/--pass` 时无认证，仅建议纯内网使用。

### 3.2 常用启动参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--port` | `8080` | 监听端口 |
| `--dir` | 无 | 文件根目录（也可启动后在网页里选） |
| `--data-dir` | `./data` | 配置与分享记录存放目录，**必须持久化** |
| `--user` / `--pass` | 无 | 管理端账号密码 |
| `--max-upload-mb` | `0` | 单文件上传上限（MB），0 不限制 |
| `--trust-proxy` | `0` | 反向代理场景置 1 |

同名环境变量（`PORT`、`ROOT_DIR`、`AUTH_PASS` 等）同样生效，命令行优先。

> [!TIP] 建议
> `--data-dir` 务必放在持久目录，分享记录存在里面的 `shares.json`，目录丢了链接全部失效。

---

## 四、一键部署到 Linux（推荐）

部署脚本自动识别 CPU 架构、从 Release 下载二进制、放行防火墙、注册 systemd 开机自启：

```bash
curl -fsSL https://raw.githubusercontent.com/Zlion-Y/fileserver/main/install-remote.sh -o install-remote.sh

sudo bash install-remote.sh --dry-run

sudo FS_USER=admin FS_PASS=你的密码 bash install-remote.sh
```

> [!NOTE] 提示
> `--dry-run` 只预览将要执行的操作，不改系统，建议先跑一次确认。

装好后的常用运维操作：

| 需求 | 命令 |
|------|------|
| 换目录 / 端口 | `sudo DIR=/mnt/disk/share PORT=9000 bash install-remote.sh` |
| 指定版本安装 | `sudo VERSION=v1.0.16 bash install-remote.sh` |
| 升级到最新版 | 重跑安装即可，自动继承已有配置与密码 |
| 查看日志 | `journalctl -u fileserver -f` |
| 卸载 | `sudo bash install-remote.sh --uninstall` |

配置文件在 `/etc/fileserver.env`，改完 `systemctl restart fileserver` 生效。

---

## 五、Docker 部署

```bash
cp .env.example .env

docker compose up -d
```

`docker-compose.yml` 默认只监听 `127.0.0.1:16666`，由 Nginx / Caddy 反代后对外。数据落在 `./files` 与 `./data`，容器重建不丢。

> [!WARNING] 警告
> Docker 场景升级请重新构建镜像，不要在容器内替换二进制；未注入 `--build-arg VERSION` 时版本为 `dev`，网页会禁用一键更新。

---

## 六、一键自更新是怎么工作的

个人服务器最烦的就是升级：登服务器、找架构、下载、替换、重启。fileserver 把这整套做成了一个按钮：

```mermaid
graph TD
    A[网页点击一键更新] --> B[服务端查询 GitHub 最新 Release]
    B --> C[按当前架构下载对应二进制]
    C --> D{校验二进制魔数}
    D -->|通过| E[原子替换旧文件]
    D -->|失败| F[中止并报错]
    E --> G[自动重启新版本]
    G --> H[网页展示版本更新日志]
```

服务器连不上 GitHub 也没关系：本地下载好二进制，从网页直接上传更新包（上限 256MB），校验通过后同样替换重启。

---

## 七、安全设计

- **口令哈希**：管理端与分享密码均用 PBKDF2-HMAC-SHA256（管理端 10 万轮 / 分享端 5 万轮），旧版单次 SHA-256 记录校验通过后自动就地升级，无需用户操作
- **密码加密存储**：分享密码以 AES-GCM 加密落盘，主密钥独立文件存放（0600），`shares.json` 泄露不再连带交出密码
- **改密踢会话**：修改管理员密码后立即踢出其他所有登录会话，被盗会话不会永久有效
- **续传授权**：Range 断点续传必须持有真实下载后签发的 ticket Cookie，伪造 `Range: bytes=1-` 无法绕过一次性链接的次数限制
- **路径越权**：所有路径参数做根目录校验，`../` 逃逸无效
- **上传限制**：`MAX_UPLOAD_MB` 对 chunked 上传同样生效，超限立即中断并删除半成品

---

## 八、注意事项与故障排查

> [!IMPORTANT] 重要
> 公网部署务必设置 `AUTH_USER` / `AUTH_PASS`，否则任何人都能管理你的文件。

> [!NOTE] 反代注意
> 反向代理场景需设置 `TRUST_PROXY=1`，否则生成的分享链接会是 `http://127.0.0.1:...`；Nginx 需放开 `client_max_body_size 0` 并关闭 `proxy_buffering`。

常见问题排查：

| 现象 | 排查方向 |
|------|----------|
| 分享链接打开 404 | `shares.json` 是否随 `--data-dir` 丢失；文件是否被移动或删除 |
| 链接提示下载次数用完 | 限次已耗尽，可在分享管理里快速编辑重新放开 |
| 反代后链接是 127.0.0.1 | 未设 `TRUST_PROXY=1`，或反代来源不在环回 / 内网信任范围 |
| 上传大文件失败 413 | 触发了 `MAX_UPLOAD_MB` 限制，或 Nginx 未放开 `client_max_body_size` |
| 一键更新报错 | 服务器连不上 GitHub，改用网页上传更新包方式 |
| 忘记管理密码 | 在配置中改 `AUTH_PASS` 后重启；网页改过密码的话删除 `config.json` 中的 `authPassHash` 字段即可回退到启动参数密码 |

---

> **文档版本：** 2026-09-09
> **软件版本：** fileserver v1.0.16
> **适用平台：** 云服务器 / 本地虚拟机 / NAS / 软路由 / Docker
