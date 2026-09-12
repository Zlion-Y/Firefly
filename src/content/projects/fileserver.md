---
title: "fileserver"
slug: fileserver
published: 2026-09-09
draft: false
description: "一个 Go 编译的单文件自建文件服务器：分片断点续传上传、限时限次分享链接、WebDAV、媒体相册预览、一键自更新，零运行时依赖，8 个平台开箱即用。"
image: "images/fileserver.png"
status: "published"
tags:
  - Go
  - 自托管
  - 文件服务
  - NAS
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/Zlion-Y/fileserver"
  - label: "下载"
    icon: "material-symbols:download"
    value: "https://github.com/Zlion-Y/fileserver/releases"
  - label: "使用教程"
    icon: "material-symbols:menu-book"
    value: "/posts/fileserver-single-binary/"
lang: "zh_CN"
---

::github{repo="Zlion-Y/fileserver"}

一个可执行文件，拷过去就能跑的自建文件服务器。网盘限速、第三方工具要装 Docker 拉一堆镜像、老 NAS 上连 Node 都跑不动——fileserver 就是为这些场景而生：**单个二进制文件，零运行时依赖**。

## 核心特性

- **单文件部署**：Go 编写、零第三方库，前端页面经 `go:embed` 直接编译进二进制；图片缩略图、文本预览、统计图全部在浏览器端渲染，弱机服务器只负责透传字节流
- **8 个平台开箱即用**：Linux（amd64 / arm64 / armv7 / armv5 / mipsle / 386）/ Windows / macOS，从 x86 云服务器到 mipsle 架构的老路由器总有一个包能跑
- **文件管理**：分片断点续传上传、文件夹整体上传、回收站、全盘递归搜索、万级文件目录虚拟滚动
- **分享链接**：有效期（1 小时 ~ 永久）+ 下载次数双限制、自定义别名、Referer 防盗链、PBKDF2 分享密码、访问日志与 30 天下载统计
- **WebDAV 直连**：挂载为本地磁盘，配合播放器、同步盘即插即用
- **在线预览**：图片 / 视频 / 音频浏览器内直接播放，txt / md / 源码 / 配置直接查看，图片文件夹一键切换相册视图
- **一键自更新**：检查新版本并原地升级，无需重新部署

## 快速开始

从 [Releases](https://github.com/Zlion-Y/fileserver/releases) 下载对应架构的二进制，赋予执行权限后运行即可：

```shell
./fileserver -port 8080 -dir /data
```

详细的部署方式（裸机 / Docker / systemd）、反向代理与安全设计说明见[使用教程](/posts/fileserver-single-binary/)。
