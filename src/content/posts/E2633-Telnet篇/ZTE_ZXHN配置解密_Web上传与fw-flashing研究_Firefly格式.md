---
title: 从U-Boot到Telnet研究中兴巡天AX3000（Telnet篇）
published: 2026-08-04
updated: 2026-08-04
description: 中兴巡天AX3000 从配置文件config.bin的 Type-4 解密、Telnet 开启、Tag 处理、Web 固件上传到双槽 fw-flashing 的完整分析、命令、失败尝试和通用工具。
tags: [ZTE, ZXHN, E1630, E2631, 固件, Type-4, Telnet, NAND, fw-flashing]
category: 路由器
draft: true
pinned: false
slug: e2631-research-telnet
author: zlion
---

> 本文面向拥有自有 ZTE ZXHN 设备、能够通过 TTL 或 Telnet 维护设备的用户，记录一次完整的固件研究过程。
> **适用平台：** ZXHN E1630（256MB）/E2633/E2638/E2631.
> **重要边界：** 本文不把推断写成结论，不推荐直接写 Bootloader 或整片 NAND。涉及配置、序列号、MAC、账号密码和固件的内容，只应在自己的设备上使用。

> [!WARNING] 免责声明与使用约定
> 1. 本教程及随附的全部工具仅供技术研究、学习交流和维护本人拥有或已获明确授权的设备使用，不构成任何形式的商业服务、质量保证或操作承诺。
> 2. **禁止以任何形式出售、打包收费、付费分发、引流获利，或将本教程及随附工具用于其他商业获利行为。**
> 3. 对教程、文档或工具进行二次修改、转载和再分发时，必须保留原作者 `zlion`、原始版权说明及本免责声明，并明确标注原始来源；不得删除署名、冒充原创或以修改版本替代原始来源。
> 4. 固件刷写、分区读写、Telnet、TTL、U-Boot 和 MTD 操作均可能造成数据丢失、设备变砖、网络中断、保修失效或其他不可预期后果。操作者应自行核对机型、硬件版本、分区布局、文件哈希和恢复条件，**对自己的设备和全部操作结果负责并自行承担风险。**
> 5. 教程作者不保证本文内容和工具适用于所有机型、硬件批次或固件版本；在法律允许的范围内，作者不对因使用、误用或依赖本文内容造成的直接或间接损失承担责任。
> 6. 使用者必须遵守所在地适用的法律法规、网络安全规定、电信管理要求、知识产权规则及设备服务协议，不得用于未经授权的访问、破坏、规避安全机制或侵犯第三方权益。
> 7. 教程中涉及的固件、商标、产品名称及第三方程序，其权利归各自权利人所有。如本文内容存在侵权、错误引用或其他权益问题，请联系作者核实；确认后将及时删除或修正相关内容。

## 目录

1. [最终结论](#1-最终结论)
2. [设备与工具环境](#2-设备与工具环境)
3. [第一部分：config 解密与 Telnet](#3-第一部分config-解密与-telnet)
4. [第二部分：Tag、OOB 与 MTD](#4-第二部分tagoob-与-mtd)
5. [第三部分：Web 上传研究](#5-第三部分web-上传研究)
6. [第四部分：fw-flashing 与双槽启动](#6-第四部分fw-flashing-与双槽启动)
7. [第三方固件对比](#7-第三方固件对比)
8. [完整试错记录](#8-完整试错记录)
9. [通用脚本与操作说明](#9-通用脚本与操作说明)
10. [故障排查](#10-故障排查)
11. [风险边界与推荐流程](#11-风险边界与推荐流程)

---

## 1. 最终结论

### 1.1 config

- `config (*.bin)` 是 ZTE Type-4 外层容器，不是普通明文 XML。
- 本次 E1630/E2631 样本使用 AES-256-CBC，参数如下：

```text
keySeed = E2631Key13515211
ivSeed  = E2631Iv13515211
key     = SHA256(keySeed)
iv      = SHA256(ivSeed)[0:16]
padding = 关闭，尾部零填充到 AES 块边界
```

- 解密后是 Type-0 内层容器，chunk 使用 zlib，最终内容为 XML。
- 开 Telnet 要同时修改 `PortControl` 和 `TelnetCfg`。
- 输出必须重新解密回读，并与修改后的 XML 做字节级比较。

### 1.2 Tag 与 NAND

- Tag 是私有记录表，不是文件系统，不能 `mount`。
- NAND 页面为 `2048` 字节数据加 `64` 字节 OOB，擦除块为 `131072` 字节。
- `cat > /dev/mtd2` 或 `cat > /dev/mtdblock2` 未写入成功，最终以读回 MD5 不一致确认。
- 成功方法是 ARMv7 静态 MTD ioctl 工具：擦除、写入、逐字节回读。
- `fsync: Invalid argument` 可能只是驱动不支持该同步调用；只能以工具读回验证和独立 MD5 为准。

### 1.3 Web 上传

- 隐藏 Web 页面调用真实升级接口，但 `httpd` 对请求体存在较小上限。
- 约 300 KiB 文件可发完；约 320 KiB 以上开始连接重置。官方 22,413,844 字节包不适合走该页面。
- `SessionTimeout` 是业务会话问题，`ERR_CONNECTION_RESET` 是连接层问题，不能混为一谈。

### 1.4 fw-flashing

- `/bin/fw_flashing -f FILE` 使用带升级头的小包，不是 128 MiB 全片镜像。
- 实验中它把官方包写入备用高槽 `0x02300000`，之后设备从高槽成功启动。
- 平台至少有低槽 `0x00700000` 和高槽 `0x02300000`，高槽边界为 `0x03f00000`。
- 已确认有 Magic、版本、板型、VID、uImage/CRC、范围、写后回读等检查；RSA 验证的准确位置仍未确认。
- 双槽和 `IsBad` 字段已确认，但自动回退没有故意破坏固件验证，不能承诺一定会发生。

---

## 2. 设备与工具环境

### 2.1 已确认信息

```text
Linux 4.1.25 / armv7l / Buildroot 2015.08.1
CPU ZX279128R
RAM 256M
SPI NAND 128MiB
Page 2048 / OOB 64 / Erase 131072
路由器 192.168.5.1
电脑/TFTP 192.168.5.7
```

过程中还使用过 `192.168.5.6` 和 `192.168.10.4`，命令中的 IP 必须按当前网卡修改。

### 2.2 典型分区表

曾从目标设备读到：

```text
dev:    size      erasesize  name
mtd0:   08000000  00020000   Whole flash
mtd1:   00100000  00020000   Bootloader
mtd2:   00100000  00020000   tag
mtd3:   00100000  00020000   wifi
mtd4:   00200000  00020000   usercfg
mtd5:   00200000  00020000   defcfg
mtd6:   00340000  00020000   kernel1
mtd7:   00340000  00020000   kernel2
mtd8:   01660000  00020000   rootfs
mtd9:   00500000  00020000   UUPlugin
mtd10:  000a0000  00020000   owdptsbin
mtd11:  00500000  00020000   status
mtd12:  00300000  00020000   DPIPlugin
```

官方包刷写后，rootfs 大小和后续插件分区可能变化，曾见到 `mtd8=0x01220000`、`UNIFIED` 和 `JDXBPLUGIN`。因此不能把旧分区表硬编码到新设备；每次操作前重新读取 `/proc/mtd`。

### 2.3 路由器只读采集

```sh
uname -a

cat /proc/version

cat /proc/cmdline

cat /proc/mtd

cat /proc/capability/boardtype

df -k /tmp /var/tmp
```

BusyBox TFTP：

```sh
# 电脑到路由器
tftp -g -l /tmp/file.bin -r file.bin 192.168.5.7

# 路由器到电脑
tftp -p -l /tmp/file.bin -r file.bin 192.168.5.7
```

研究 Web/config 时曾提取目标机自己的二进制和配置材料：

```sh
tftp -p -l /bin/cspd -r E1630-cspd 192.168.5.7

tftp -p -l /bin/httpd -r E1630-httpd 192.168.5.7

tftp -p -l /bin/fw_flashing -r E1630-fw_flashing 192.168.5.7

tftp -p -l /etc/enhardcodefile -r enhardcodefile 192.168.5.7

tftp -p -l /etc/enwebdhardcodefile -r enwebdhardcodefile 192.168.5.7

tftp -p -l /etc/hardcode -r hardcode 192.168.5.7

tftp -p -l /lib/libhardcode.so -r libhardcode.so 192.168.5.7

tftp -p -l /lib/libsha256.so -r libsha256.so 192.168.5.7

tftp -p -l /lib/libtagparam.so -r libtagparam.so 192.168.5.7
```

实际路径可能位于 `/kmodule/bin` 或其他目录。先用 `ls -l /proc/PID/exe` 确认正在运行的二进制，再比较 MD5，避免分析错副本。

### 2.4 哈希

Windows CMD：

```bat
certutil -hashfile fw.bin MD5

certutil -hashfile fw.bin SHA256
```

Linux/Armbian：

```sh
md5sum fw.bin

sha256sum fw.bin
```

---

## 3. 第一部分：config 解密与 Telnet

### 3.1 只使用目标机文件

早期参考过 H3600P、E2631/E2638 文章和 `ztetool.py`。这些内容只能提供格式线索，不能证明密钥、Tag、分区和升级策略相同。后续研究停止使用参考机文件，改为目标设备自己的 `config`、`cspd`、`httpd`、`fw_flashing` 和硬编码文件。

备份后先记录哈希：

```bat
certutil -hashfile config.bin SHA256
```

### 3.2 识别和解密 Type-4

Type-4 开头：

```text
magic = 0x01020304
type  = 4
```

只读检查：

```bash
node common-tools/zte_type4_config_tool.js inspect config.bin
```

解密：

```bash
node common-tools/zte_type4_config_tool.js decrypt config.bin config.xml --profile e2631
```

内部流程：

1. 读取 Type-4 外层 chunk。
2. 用 SHA256 派生 AES key/IV，对每个 payload 做 AES-256-CBC 解密。
3. 检查内层 `0x01020304 / type 0`。
4. 校验内层 header CRC32 和压缩 payload CRC32。
5. 对 zlib chunk 解压，检查每块长度和 XML 总长度。

任何 Magic、CRC、长度或 zlib 失败都必须停止。

### 3.3 Telnet 的两个修改点

`PortControl` 中 TELNET 行：

```xml
<DM name="ServName" val="TELNET"/>
<DM name="PortEnable" val="1"/>
```

`TelnetCfg` 中启用 LAN、关闭 WAN：

```xml
<Tbl name="TelnetCfg" RowCount="1">
  <Row No="0">
    <DM name="TS_Enable" val="1"/>
    <DM name="Wan_Enable" val="0"/>
    <DM name="Lan_Enable" val="1"/>
    <DM name="TS_Port" val="23"/>
    <DM name="TS_UName" val="admin"/>
    <DM name="TS_UPwd" val="admin"/>
  </Row>
</Tbl>
```

自动修改、加密、回读：

```bash
node common-tools/zte_type4_config_tool.js enable-telnet config.bin config-telnet-enable.bin --profile e2631 --user admin --password admin --xml-output config-telnet-enable.xml
```

工具会确认两类表都存在，并在输出后重新解密。如果回读 XML 不一致，不会报告成功。

### 3.4 Windows 双击脚本

将下列文件放在同一目录：

```text
config.bin
zte_type4_config_tool.js
enable_telnet_admin.bat
```

双击 `enable_telnet_admin.bat`。它检查输入、工具、`node.exe`、退出码和输出文件。成功生成：

```text
config-telnet-enable.bin
账号 admin
密码 admin
端口 23，仅开启 LAN
```

没有安装 Node.js 时，可以把可信来源的 `node.exe` 放在脚本目录。BAT 本身不能原生完成 AES、SHA256、zlib 和 CRC；它只是稳定启动处理程序并保留错误窗口。

### 3.5 本次配置样本产物

期间处理过多个目标设备备份，工作区中保留了类似以下文件：

```text
outputs/config15-telnet-root.bin
outputs/config16-telnet-root.bin
outputs/config17-telnet-root.bin
outputs/config15-telnet-root-verify.xml
outputs/config16-telnet-root-verify.xml
outputs/config17-telnet-root-verify.xml
```

早期曾测试 `root/root`，后续通用 BAT 默认使用 `admin/admin`。两套输出不能混淆，导入前必须解密 XML 检查实际账号。`config16-telnet-root.bin` 和 `config17-telnet-root.bin` 的历史输出 SHA256 分别为：

```text
config16: 56492433c67efc3157d5ffe42a3e8d90a2f6a68764c1983f1fbeb53cc85a84d0
config17: caed8d8b75f5e90d999a84a2ed1097ff8d530fb8af90eb67b2260775af51bdf2
```

这些哈希只用于识别本次历史产物，不是通用校验值。任何新配置都应重新计算。

### 3.6 config 试错

#### `sendcmd cspd hcget` 没有取到 key

```sh
sendcmd cspd hcget /etc/enhardcodefile E2631.Cfg.AESKey

sendcmd cspd hcget /etc/enhardcodefile E2631.Cfg.AESIv
```

返回的是 `cspd.dbd_task.DB` 帮助，不是 key/IV。原因可能是命令路由、进程名或参数不匹配，不能据此认定硬编码项不存在。

#### 只改一张表无效

只改 `TelnetCfg` 或只改 `PortControl` 都可能导致 Telnet 不启动。最终工具把两处作为强制条件。

#### BAT 闪退

原因包括工作目录错误、Node 不存在、错误输出被隐藏和编码问题。修正为 `%~dp0` 定位目录、保留错误输出、失败 `pause`、删除不完整输出。

---

## 4. 第二部分：Tag、OOB 与 MTD

### 4.1 Tag 读取与结构

Tag 不是文件系统。U-Boot 中分区起始 `0x100000`、大小 `0x100000`：

```text
nand read 0x48000000 0x100000 0x100000

md.b 0x48000000 0x220
```

样本结构：

```text
0x00  33 33 33 33
0x04  payload size，小端
0x08  CRC32，小端
0x0c  记录区
尾部  ff 填充
```

CRC32 计算范围从 `0x0c` 开始，长度由 payload size 决定。

本次 Tag 中同时存在主 MAC、第二 MAC、MAC 前缀、D-SN、S/N、型号和 Wi-Fi 信息。`F8731A` 只是 MAC 前三字节，不是完整 MAC；修改时不能只做一次字符串替换。文档不公开真实设备标识。

### 4.2 去 OOB 和切分区

含 OOB 镜像步长：

```text
2048 + 64 = 2112 bytes
```

去 OOB 并切 Tag/Wi-Fi：

```bash
node common-tools/strip_oob_and_slice.js e2638.bin e2638-no-oob.bin --page 2048 --oob 64 --slice tag:0x100000:0x100000:tag.bin --slice wifi:0x200000:0x100000:wifi.bin
```

偏移是去 OOB 后的逻辑偏移。脚本流式处理，不修改源镜像，并输出 SHA256。

### 4.3 直接写设备节点失败

```sh
cat /tmp/tag.bin > /dev/mtd2

cat /tmp/tag.bin > /dev/mtdblock2

sync

cat /dev/mtd2 > /tmp/tag_readback.bin

md5sum /tmp/tag.bin /tmp/tag_readback.bin
```

设备返回：

```text
/bin/sh: can't create /dev/mtdblock2: Success
```

且 MD5 不一致。重新 `mknod` 到 `/tmp`、修改节点权限也无效，因为根因不是路径权限，而是 MTD 擦除/写入接口。

### 4.4 编译 ARMv7 MTD 工具

默认 `gcc` 得到 AArch64，目标路由器是 armv7l，不能使用。安装交叉工具链：

```sh
apt update

apt install -y gcc-arm-linux-gnueabi binutils-arm-linux-gnueabi libc6-dev-armel-cross linux-libc-dev-armel-cross
```

清除宿主头文件变量后编译：

```sh
unset CPATH

unset C_INCLUDE_PATH

unset CPLUS_INCLUDE_PATH

unset LIBRARY_PATH

arm-linux-gnueabi-gcc -Os -static -s -o mtd_write_tag_armv7 mtd_write_tag.c

file mtd_write_tag_armv7
```

正确架构：

```text
ELF 32-bit LSB executable, ARM, EABI5, statically linked
```

第一次报 `bits/wordsize.h: No such file or directory`，正是缺少 `libc6-dev-armel-cross`/`linux-libc-dev-armel-cross`。

### 4.5 写入和三重校验

```sh
tftp -g -l /tmp/mtd_write_tag -r mtd_write_tag_armv7 192.168.5.7

tftp -g -l /tmp/tag.bin -r tag.bin 192.168.5.7

chmod +x /tmp/mtd_write_tag

/tmp/mtd_write_tag --yes /dev/mtd2 /tmp/tag.bin

cat /dev/mtd2 > /tmp/tag_readback.bin

md5sum /tmp/tag.bin /tmp/tag_readback.bin

tftp -p -l /tmp/tag_readback.bin -r tag_readback.bin 192.168.5.7
```

成功条件：工具内部逐字节 `OK`、路由器 MD5 一致、电脑端原文件与读回文件 MD5 一致。

`MEMUNLOCK warning: Operation not supported` 可在后续擦除/写入/校验全部成功时视为非致命。`fsync: Invalid argument` 也是驱动不支持同步调用的可能表现，但绝不能跳过读回校验。

---

## 5. 第三部分：Web 上传研究

### 5.1 接口和表单

抓到的 Web 流程：

```text
GET  /?_type=vueData&_tag=vue_nowversion_lua
POST /?_type=vueData&_tag=prepare_firmware_upgrade
POST /?_type=vueData&_tag=do_firmware_upgrade
```

multipart 字段名：

```text
VersionUpload
```

核心表单头：

```text
Content-Type: multipart/form-data; boundary=...
Content-Disposition: form-data; name="VersionUpload"; filename="firmware.bin"
Content-Type: application/octet-stream
```

### 5.2 SessionTimeout 的定位

页面返回过 `logintoken`，响应头也返回过 `X_XSRF_TOKEN`。曾出现：

```text
HTTP 200
IF_ERRORSTR = SessionTimeout
```

以及多种 token 组合均失败：

```text
prepare same token  => SessionTimeout
prepare sessOnly    => SessionTimeout
prepare sessionOnly => SessionTimeout
```

后续在同一登录 Session 内获取页面 token、Cookie 和 XSRF token，并先调用 `vue_loadprevent_data`，曾得到：

```text
prepare same:<page token> => SUCC
```

这只说明准备阶段通过，不说明上传和刷写一定成功。

### 5.3 升级管理器调试

Telnet 下可以打开升级管理器日志并查看状态：

```sh
sendcmd cspd.cspd.upgrade_mgr setDebug 1

sendcmd cspd.cspd.upgrade_mgr show

sendcmd cspd.cspd.upgrade_mgr -p

sendcmd cspd.cspd.upgrade_mgr -l
```

`show` 中的 `State`、`Location`、`TempDirName`、`FlashingPid` 和 `MediaPid` 可帮助判断升级是否真正进入后端。研究中 Web 失败后常见：

```text
State       :0
Location    :
TempDirName :
MediaPid    :
```

同时没有 `/var/tmp/fw.bin`，这更像请求未进入后端刷写，而不是固件已经写入。

### 5.4 请求体上限实验

| 测试文件 | 现象 | 判断 |
|---|---|---|
| 约 300 KiB | 100% 后 HTTP 200，`SessionTimeout` | 能传完整请求，会话仍有问题 |
| 约 320 KiB | 100% 后连接重置 | 接近/超过请求体限制 |
| 约 330 KiB | 约 97% 后重置 | multipart 边界也占空间 |
| 约 512 KiB | 约 62% 后重置 | 设备端提前关闭连接 |
| 22,413,844 bytes 官方包 | 无法稳定上传 | 不适合该页面 |

`/bin/httpd` 中发现：

```text
content length is beyond limit.
Http request body read fail.
Upload file fail for not enough space, g_nContentLength(%d) >= g_nAllowedSpace(%d)
fwrite fail when upload.
VersionUpload
/var/tmp/fw.bin
do_firmware_upgrade
my_upload_file
```

这证明有 body length、临时空间和写文件分支；字符串本身不能证明唯一的上限数值。

### 5.5 `/tmp` 空间与 Web 限制

当时读取到：

```text
/var/tmp 约 40960 KiB，可用约 40736 KiB
/tmp     约 20480 KiB，可用约 19456 KiB
```

官方包约 21.38 MiB，在 `/var/tmp` 理论上放得下，但仍被 HTTP 处理路径重置。磁盘空间够，不等于请求体限制够，也不等于升级进程会接受该文件。

### 5.6 Wireshark 过滤器

只看 HTTP 上传：

```text
ip.addr == 192.168.5.1 && ip.addr == 192.168.5.7 && tcp.port == 80
```

只看电脑到路由器：

```text
ip.src == 192.168.5.7 && ip.dst == 192.168.5.1 && tcp.dstport == 80
```

只看重置/重传：

```text
tcp.flags.reset == 1 || tcp.analysis.retransmission
```

早期过滤 `udp` 只能看到 DNS、多播或调试包，不能证明 HTTP 升级。抓包中由 `192.168.5.1:80` 返回 `RST, ACK`，与设备端连接关闭相符。

### 5.7 Web 结论

- 进度 100% 只表示浏览器发完请求，不表示设备已经校验或写入。
- `SessionTimeout` 和 `ERR_CONNECTION_RESET` 必须分层排查。
- 大于几百 KiB 的正式升级包应走 Telnet + TFTP + 本地 `fw_flashing`，不应继续反复提交 Web 请求。
- 隐藏页面是真实刷写路径，不是安全测试页面。

---

## 6. 第四部分：fw-flashing 与双槽启动

### 6.1 参数和最小失败测试

```sh
/bin/fw_flashing

/bin/fw_flashing -h
```

输出包含：

```text
fw_flashing:no parameter
d:r:p:f:m:v:
```

确认 `-f`：

```sh
echo test > /var/tmp/fw-test.bin

/bin/fw_flashing -f /var/tmp/fw-test.bin

echo $?
```

输出：

```text
fw_flashing name=/var/tmp/fw-test.bin,states.st_size =5
read magic failed!
GetVersionInfo error==========fw_flashing error==========
```

说明 `-f` 确实接受路径，并要求带固件 Magic/版本头。

### 6.2 小升级包和 128 MiB 全片的区别

本次官方升级包：

```text
文件大小     22,413,844 = 0x1560214
文件头       0x214 bytes
实际固件体   0x1560000 bytes
```

全片镜像：

```text
Whole flash  0x08000000
```

| 文件 | 用途 |
|---|---|
| 128 MiB 全片镜像 | 备份、离线分区分析、专用救援流程 |
| 约 22 MiB `fw.bin` | `fw_flashing -f` 带头升级包 |
| 约 25/26 MiB 第三方包 | 仍是带头包，但版本、签名和校验必须单独确认 |

不要把 `e2638_no_oob_full_128MiB.bin` 直接传给 `fw_flashing -f`。

### 6.3 官方包成功刷写

电脑确认：

```bat
dir fw.bin

certutil -hashfile fw.bin MD5
```

路由器获取并校验：

```sh
cd /var/tmp

rm -f /var/tmp/fw.bin

tftp -g -l /var/tmp/fw.bin -r fw.bin 192.168.5.7

ls -l /var/tmp/fw.bin

md5sum /var/tmp/fw.bin
```

本次包 MD5：

```text
de682a3dce2b128cf7edeb8ac5f958a1
```

执行：

```sh
/bin/fw_flashing -f /var/tmp/fw.bin
```

关键输出：

```text
RunningVerStartAddr:700000
RunningHeadHighAddr:2300000
g_ver_towrite=2
g_upgrade_startaddr=0x2300000
g_upgrade_endaddr=0x3f00000
firmware_size=22413312
firmware_offset=0x214
sect_size=0x20000
WriteVersion head success!
==========fw_flashing success==================
```

说明程序选择备用高槽、擦除并写入了包体，同时更新了版本头。

### 6.4 确认当前槽

```sh
cat /proc/cmdline

cat /proc/zte/verinfo/versionstates

cat /proc/zte/verinfo/softVersion

cat /proc/zte/verinfo/othersoftVersion
```

槽位关键字段：

| 字段 | 含义 |
|---|---|
| `currentverphyaddr` | 当前运行物理起始地址 |
| `backverphyaddr` | 备用槽物理起始地址 |
| `curverIsBad` | 当前槽是否标坏 |
| `backverIsBad` | 备用槽是否标坏 |
| `curverSerialNumber` | 当前记录序号 |
| `othersoftVersion` | 备用槽版本记录 |

本次低槽运行时确认：

```text
currentverphyaddr: 0x00700000
backverphyaddr:    0x02300000
```

另一次从高槽启动时，`/proc/cmdline` 尾部出现 `0x2300000`。两者一致时，才是较强的槽位证据；不能只看 `root=/dev/mtdblock8`。

### 6.5 已确认的检查方向

从 `/bin/fw_flashing` 字符串、符号和运行输出来看，存在：

```text
read magic failed!
GetVersionInfo
check_ver_board
invalid vid version!
CheckBootFile
CheckFile
Checkheader
CspUserMtdRead / Write / Erase
flash addr overflow
uImage
CRC
CspSwitchVersion
```

合理的校验链：

1. 文件大小和可读性。
2. Magic、版本和升级头。
3. 板型、VID、版本关系。
4. 长度和目标槽范围。
5. uImage 头部/数据 CRC，具体分支由版本决定。
6. 擦除、写入和写后读回。
7. 写版本头、切换或等待下一次启动选择。

未找到明确 RSA 公钥验证调用。RSA 可能在 `cspd`、Web、BootROM 或其他库中，不能仅凭 `fw_flashing` 字符串判断。

### 6.6 低版本回官方最新包

先采集：

```sh
cat /proc/capability/boardtype

cat /proc/zte/verinfo/softVersion

cat /proc/zte/verinfo/versionstates
```

离线分析：

```bash
node common-tools/zte_firmware_analyzer.js official-fw.bin --signature-offset 0x37fdcc --signature-size 564
```

确认型号、硬件版本、VID、包头格式和槽容量后，仍应保留当前可启动槽、Tag、配置和 Bootloader 备份。`fw_flashing` 没有可靠的只检查不写入模式，正式执行随时可能擦写备用槽。

---

## 7. 第三方固件对比

### 7.1 官方包

```text
大小：22,413,844 bytes
MD5：de682a3dce2b128cf7edeb8ac5f958a1
SHA256：ad05c7afade1bc2e0c900cf05e19a1b05b6df92d8d33bbb598436ab1cba77326
型号：ZXHN E2631 V1.0.0
版本：V1.0.0.7B5.8000
```

### 7.2 `E2633toE3630`/`AX5400Pro` 包

不同文件名的两个包字节完全一致：

```text
大小：25,428,500 bytes
MD5：1f3109ec3e282c93ae6964d88c6f5485
SHA256：d0218b01739fa6fc110735f9020f38f1c0b9d3e1b6d026088568e798f091f7a8
包头型号：ZXHN E2631 V1.0.0
版本：V1.0.0.3B1.8000
日期：20230327020150
```

固定签名区 `0x37fdcc`、长度 564 字节全零。文件名中的 E2633/E3630/AX5400Pro 不能替代包头实际型号。

### 7.3 `V1.0.0.6B3.8000` 包

```text
大小：26,870,292 bytes
MD5：525045e36cc27dd9ef42c21f99e54921
SHA256：0617c17a70972cdc8e9b0c7beffa32e3b8bd86faad5f888d94e0ec62d492ce6c
包头型号：ZXHN E2631 V1.0.0
版本：V1.0.0.6B3.8000
日期：20240611215845
```

该包签名区非全零，且长度仍落在单槽容量内；但从 `7B5` 降到 `6B3` 是降级场景，不能只看型号匹配。

### 7.4 第三方升级程序

| 文件 | 官方 MD5 | 第三方 MD5 | 判断 |
|---|---|---|---|
| `fw_flashing` | `b543771bf99ba80cdd12e1f327563039` | `d3c4d2c36e5b24a5348dee7711b625b2` | 二进制不同 |
| `boot_flashing` | `5b651691df4d333fc94cf07cb8afbe84` | `5152556ea2d0a0b788ebfe3cfdb5bc6d` | 二进制不同 |
| `cspd` | `c9fb945c7789335f2307e876d0e6aa72` | `d1dd580d45819fb92e6978d495112a2f` | 升级策略可能不同 |

第三方 `fw_flashing` 仍保留 Magic、板型、VID、uImage、CRC、范围、写后回读和版本切换相关字符串。因此准确描述是“升级栈被修改，部分策略或签名处理发生变化”，不是“所有校验都被删除”。

---

## 8. 完整试错记录

### 8.1 U-Boot 命令差异

缺失或不可用：

```text
mtdparts
crc32
tftpput
fatwrite
usb
mmc
loadb
loads
```

可用关键命令：

```text
nand read / write / read.raw / erase / bad
tftp
xmodem
mtddebug
cspboot
cspstart
```

标准 U-Boot 教程不能直接套用。

### 8.2 U-Boot 中断问题

从 `/dev/mtd1` 和 `/dev/mtd0` 仍能 grep 到：

```text
bootdelay=3
bootcmd=setenv
BootImageNum=0x00000001
```

但实际启动日志没有稳定倒计时。此前 TTL 曾正常显示倒计时，硬件问题已低优先级排除。前置提示为：

```text
Hit 1 to upgrade software version
Hit any key to stop autoboot: 0
```

更像 `cspboot` 阶段输入窗口极短或串口时机问题，而不是 Linux `bootargs`。继续修改 Linux 参数不能解决尚未启动 Linux 的问题。

### 8.3 Tag 直接写失败

尝试 `cat`、`mknod`、`chmod` 后仍 MD5 不一致。设备没有 `mtd_debug`、`flash_erase`、`nandwrite`，只有 `boot_flashing`、`fw_flashing`、`upgradetest` 等程序；这些程序没有暴露 Tag 写入接口。

最终使用 MTD ioctl 工具成功。

### 8.4 Web 反复失败

```text
升级准备失败：SessionTimeout
上传连接失败
net::ERR_CONNECTION_RESET
```

进度 31%、62%、97%、100% 后都见过失败。用小文件先拆分问题，确认大文件主要被 HTTP 路径限制后，停止继续提交 22 MiB 包，转为本地 `fw_flashing`。

### 8.5 `boot_flashing` 误用风险

```sh
/bin/boot_flashing

/bin/boot_flashing -h
```

它要求特定带头的 Boot 文件，并直接涉及 `/dev/mtd1`。从 `/dev/mtd1` 读出的裸 1 MiB 备份不能直接作为其输入。

### 8.6 未验证自动回退

已确认 `currentverphyaddr`、`backverphyaddr`、两个 `IsBad` 字段，但没有故意刷坏包测试回退。不要为了得到一个“确定答案”而破坏当前可启动槽。

---

## 9. 通用脚本与操作说明

### 9.1 目录

```text
common-tools/
  README.md
  zte_type4_config_tool.js
  enable_telnet_admin.bat
  zte_firmware_analyzer.js
  strip_oob_and_slice.js
  fw_slot_status.sh
```

### 9.2 Type-4 工具

```bash
node common-tools/zte_type4_config_tool.js inspect config.bin

node common-tools/zte_type4_config_tool.js decrypt config.bin config.xml --profile e2631

node common-tools/zte_type4_config_tool.js encrypt config.xml config-new.bin --profile e2631

node common-tools/zte_type4_config_tool.js enable-telnet config.bin config-telnet.bin --profile e2631 --user admin --password admin
```

其他机型必须显式提供已验证种子：

```bash
node common-tools/zte_type4_config_tool.js decrypt config.bin config.xml --key-seed MODELKeyXXXXXXXX --iv-seed MODELIvXXXXXXXX
```

### 9.3 固件分析工具

```bash
node common-tools/zte_firmware_analyzer.js firmware.bin

node common-tools/zte_firmware_analyzer.js firmware.bin --signature-offset 0x37fdcc --signature-size 564
```

输出哈希、可打印头部、型号标记、uImage CRC 和指定签名区零字节数。签名区非零不等于签名有效。

### 9.4 OOB 和切片工具

```bash
node common-tools/strip_oob_and_slice.js raw.bin no-oob.bin --page 2048 --oob 64 --slice tag:0x100000:0x100000:tag.bin

node common-tools/strip_oob_and_slice.js raw.bin no-oob.bin --page 2048 --oob 64 --slice tag:0x100000:0x100000:tag.bin --slice wifi:0x200000:0x100000:wifi.bin
```

offset 是去 OOB 后的逻辑偏移，原始镜像不会被修改。

### 9.5 双槽只读采集

```sh
tftp -g -l /tmp/fw_slot_status.sh -r fw_slot_status.sh 192.168.5.7

/bin/sh /tmp/fw_slot_status.sh > /tmp/fw-slot-status.txt

tftp -p -l /tmp/fw-slot-status.txt -r fw-slot-status.txt 192.168.5.7
```

重点查看 `versionstates` 的当前地址、备用地址和 `IsBad`，不要把 `othersoftVersion` 当作自动回退证明。

### 9.6 写 Tag 最小清单

```sh
cat /sys/class/mtd/mtd2/name 2>/dev/null

cat /sys/class/mtd/mtd2/size 2>/dev/null

tftp -g -l /tmp/mtd_write_tag -r mtd_write_tag_armv7 192.168.5.7

tftp -g -l /tmp/tag.bin -r tag.bin 192.168.5.7

chmod +x /tmp/mtd_write_tag

/tmp/mtd_write_tag --yes /dev/mtd2 /tmp/tag.bin

cat /dev/mtd2 > /tmp/tag_readback.bin

md5sum /tmp/tag.bin /tmp/tag_readback.bin
```

只有工具 `OK` 且输入/读回 MD5 一致才继续。任何不一致都应停止并保存日志，不要马上重启。

---

## 10. 故障排查

### 10.1 TFTP 超时或 `(512)`

```sh
ping -c 3 192.168.5.7

ls -l /tmp/file.bin

tftp -g -l /tmp/file.bin -r file.bin 192.168.5.7
```

电脑侧检查当前 IP、TFTP 根目录、Windows 防火墙、UDP 69/数据端口和服务端覆盖权限。`(512)` 通常是服务端拒绝写入、目录权限或文件策略，不是固件格式错误。

### 10.2 架构错误

```sh
file mtd_write_tag_armv7
```

路由器为 armv7l 时必须是 `ELF 32-bit ARM EABI5`，不能是 AArch64。

### 10.3 config 解密失败

确认原文件来自目标机、前 8 字节 Type-4、种子来自目标机、ASCII 种子经过 SHA256 派生、AES 关闭自动 padding、CRC 和 zlib 均通过。失败后不要继续生成配置。

### 10.4 Web 失败

小文件返回 `SessionTimeout`，先重新登录并使用同一 Session 的 Cookie/XSRF token；小文件成功而大文件 RST，则排查请求体限制。进度 100% 不代表刷写。

### 10.5 RST 与升级状态

```sh
ls -l /var/tmp/fw.bin /tmp/fw.bin 2>/dev/null

sendcmd cspd.cspd.upgrade_mgr show

ps | grep -E "fw_flashing|httpd|cspd"
```

如果没有临时文件、升级状态为 0、由路由器发 RST，优先判定为上传链路/请求体/进程状态，而不是已经开始刷写。

### 10.6 槽位不明确

```sh
cat /proc/cmdline

cat /proc/zte/verinfo/versionstates

cat /proc/zte/verinfo/softVersion

cat /proc/zte/verinfo/othersoftVersion
```

当前地址和命令行中的物理地址一致时，证据最强。两个槽通常共用 Linux 分区编号。

---

## 11. 风险边界与推荐流程

### 11.1 推荐顺序

```text
1. 记录 /proc/mtd、boardtype、cmdline、versionstates。
2. 备份当前 config、Tag、Wi-Fi、Bootloader，并保存哈希。
3. 所有输入先离线分析，失败就停。
4. config 修改后做 XML 回读；Tag 修改后做 CRC 和结构检查。
5. 只有确认架构和分区后才使用 ARMv7 MTD 工具。
6. 固件升级使用带头小包，不把全片镜像当升级包。
7. 写入后采集当前槽、备用槽和版本状态。
8. 不覆盖唯一可启动备份槽，不主动测试坏包回退。
```

### 11.2 禁止直接做

```text
不要把参考机 config、Tag、rootfs 或 bootloader 写到目标机。

不要把 128 MiB 全片 dump 传给 fw_flashing -f。

不要用 cat > /dev/mtd2 的“Success”判断写入成功。

不要把裸 /dev/mtd1 备份直接交给 boot_flashing。

不要在没有救援手段时覆盖当前唯一正常槽。

不要故意写入损坏固件来测试自动回退。
```

### 11.3 仍未确认的事项

- 当前版本完整的 Bootloader 中断窗口和输入时序。
- `versionopenorclose=close` 的全部状态转换。
- 启动失败后的自动回退是否真的启用。
- RSA 签名在具体升级链中的准确调用位置。
- 第三方 `cspd` 修改了哪个完整性分支。
- 不同硬件批次的配置种子是否完全相同。

这些问题应优先通过只读二进制、日志、符号、包头和配置研究，不能用高风险刷写猜测。

---

> **文档版本：** 2026-08-04
> **适用系统：** Windows + Node.js；路由器 BusyBox/Linux 4.1.x；Armbian ARM 交叉编译环境
> **适用平台：** 已确认 ZXHN E1630/E2631 同平台样本；其他机型必须重新验证
