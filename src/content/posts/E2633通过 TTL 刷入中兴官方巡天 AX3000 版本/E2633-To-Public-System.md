---
title: E2633通过 TTL 刷入中兴官方巡天 AX3000 版本（实操可行）
published: 2026-07-20
updated: 2026-08-01
description: 记录 E2633通过 TTL 进入 u-boot 后，使用 TFTP 写入官方巡天 AX3000 版本固件的流程，并附保留 tag 与 wifi 分区的可选做法。
image: ./cover.png
tags: [E2633, TTL]
category: 路由器
draft: false
slug: e2633-to-public-system
---

> [!WARNING] 风险提示
> 本教程涉及刷写 NAND 闪存，存在变砖风险。请自行评估风险后操作。内容仅适用于 256 MB 内存、WSON8 封装版本。

## 说明

1. 一开始采用编程器整片写入，后来在刷 WiFi 分区时发现也可以直接通过 u-boot 刷入。
2. 由于是整片写入，刷完后默认 WiFi 名称会变为 `ZTE-Q2QHKu`，密码为 `bhdh3954`。
3. 无线校准数据 WiFi 分区、TAG 分区（SN 和 MAC 信息）都会被覆盖。但测试下来影响不大；如果想尽量完美保留这两个分区，可以先开telnet将这两个分区提取出来，后面在通过telnet刷入。
4. 仅适用于 256 MB 内存、WSON8 封装的版本。

## 准备工作

- USB 转 TTL 模块 CH340G 一个，最好带杜邦线，方便连接。
- 由编程器固件去掉 OOB 后生成的 [128 MB 镜像](https://wwbnc.lanzoub.com/idNc33xkhc4f)。编程器固件来自 [hzw521](https://www.right.com.cn/forum/thread-8472425-1-1.html)。
- MobaXterm 等串口连接工具。
- TFTP 服务工具（MobaXterm自带）。
- 网线一根。

## 操作步骤

### 1. 拆机

快拆可参考[拆机视频](https://www.bilibili.com/video/BV1ic7y6wE3E/)。

### 2. 连接 TTL 和网线

使用网线连接电脑和路由器，参考下图连接 TTL：

![TTL 接线参考](https://www.right.com.cn/forum/data/attachment/forum/202607/22/111254flissfe9fz4f9efe.jpeg)

### 3. 进入 u-boot

连接好后，使用 MobaXterm 软件进行串口连接，波特率设置为 `115200`：

![设置配置参考](https://www.right.com.cn/forum/data/attachment/forum/202607/20/221326v8t5l45tv4k66mlj.png)

路由器上电后马上按 `1` 中断系统启动，输入密码进入 u-boot（密码来源见[论坛](https://voz.vn/t/unlock-mesh-zte-e1600-2603-2615-thanh-phien-ban-quoc-te.903294/)）：

```text
5cE080@fyBD
```

### 4. 配置电脑网络和 TFTP

关闭防火墙，将电脑 IPv4 改为手动配置：

- IP 地址：`192.168.10.1`
- 子网掩码：`255.255.255.0`

设置固件存放目录。此时先不要开启 TFTP 服务。


![TFTP 目录配置参考](https://www.right.com.cn/forum/data/attachment/forum/202607/20/221333x8qwa6o5aytx6tki.png)

### 5. 在串口终端设置 IP

在 u-boot 串口终端中输入：

```shell
setenv ipaddr 192.168.10.2
setenv serverip 192.168.10.1
ping 192.168.10.1
```

看到类似下面的提示即表示通信正常：

```text
host 192.168.10.1 is alive
```

此时再启动前面配置好的 TFTP 服务。

### 6. 通过 TFTP 传输并刷写固件

在串口终端中通过 TFTP 将固件传到 RAM：

```shell
tftp 0x43000000 e2631_no_oob_full_128MiB.bin
```

传输完成后，确认下载大小必须为：

```text
Bytes transferred = 134217728 (8000000 hex)
```

![传输文件参考](https://www.right.com.cn/forum/data/attachment/forum/202607/22/111042clhollaa7jdk4xlo.png)

确认无误后，擦除整片 NAND：

```shell
nand erase 0x0 0x8000000
```

然后一次性写入整片：

```shell
nand write 0x43000000 0x0 0x8000000
```


### 7. 重启

刷写完成后执行：

```shell
reset
```

## 可选：保留原机 tag 和 wifi 分区

如果想保留原机的 tag 与 wifi 分区，可以在一开始就通过telnet提取这两个分区，然后再使用cat命令写入WiFi分区，用mtd_write_tag_armv7程序写入tag分区。程序可以看我这一篇帖子：
[使用 mtd_write_tag 刷写 ZTE TAG 分区](https://zlion.top/posts/zte-tag-mtd-write-flashing-guide/)


```
