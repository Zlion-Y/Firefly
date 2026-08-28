---
title: Z500/Z503/Z507升级小方糖公版系统
published: 2026-08-28
updated: 2026-08-28
description: 记录 Z503 通过 Telnet 扩容分区，升级为公版的过程.
image: ./cover.png
tags: [Z503, Telnet]
category: 路由器
draft: false
slug: z503-to-public
author: zlion
---

## 说明

该教程由[@cnjn](https://www.right.com.cn/forum/?639610)首发于恩山论坛：[中兴小方糖 Z503 刷 Z506 固件](https://www.right.com.cn/forum/thread-8480346-1-1.html)，这里进行优化，适用于小白操作。

## 准备工作

- [Z506官方固件](https://wwanu.lanzoue.com/ijPLK3vbfl1i)
- [一键开启Telnet脚本](https://wwbnc.lanzoub.com/iuDNi451dula)
- [一键修改Tag脚本](https://wwbnc.lanzoub.com/iP5mZ451e9hg)
- MobaXterm终端工具

## 操作步骤

### 1. 开启Telnet

从路由器网页管理后台导出配置文件`config.bin`，与`一键开启Telnet脚本`放在一起，双击脚本运行得到`config-telnet-enable.bin`，将其再从管理后台还原回去，路由器会自动重启，Telnet就打开了，默认的账密为admin。

### 2. 扩容分区

使用MobaXterm新建Telnet会话连接路由器，执行下面的命令，扩容 /var 目录。

```shell
mount -o remount,size=32M /var
```

### 3. 修改设备型号

将设备型号Z500/Z503/Z507通通改为中兴的公版型号Z506。

设置MobaXterm的TFTP服务
![TFTP 目录配置参考](./TFTP.png)

命令中的 `192.168.5.7`请替换为电脑当前的局域网 IP。

#### 3.1 提取Tag分区

```shell
cat /dev/mtdblock2 > /tmp/tag.bin
tftp -p -l /tmp/tag.bin -r tag.bin 192.168.5.7
```

#### 3.2 修改Tag分区

把提取出来的tag.bin与`一键修改Tag脚本`放在一起双击脚本，得到`tag-Z506.bin`文件，放到TFTP目录中。

#### 3.3 写入Tag分区

```shell
tftp -g -l /tmp/tag.bin -r tag-E2631.bin 192.168.5.7
cat /tmp/tag.bin > /dev/mtdblock2
```

### 4. 上传公版固件

进入路由器管理后台，找到系统更新固件上传`zxhnze50x_hv100_fv1002b28000_firmware-20230615093954.bin` ,确认升级即可。重启完毕后就成功刷为公版系统了。

### 5. 重置

升级完成后可以恢复出厂设置一次。