---
title: E2633转正官方版分析（Telnet篇）
published: 2026-08-04
updated: 2026-08-04
description: E2633刷入官方公版后的config.bin解密、本机信息优化、本地Web上传固件和官方系统升级研究部分。
tags: [E2631,  Telnet, fw-flashing]
category: 路由器
draft: false
pinned: false
slug: e2631-research-telnet
image: ./cover.ping
author: zlion
---

> [!CAUTION] 免责声明与使用约定
> 1. 本教程及随附的全部工具仅供技术研究、学习交流和维护本人拥有或已获明确授权的设备使用，不构成任何形式的商业服务、质量保证或操作承诺。
> 2. **禁止以任何形式出售、打包收费、付费分发、引流获利，或将本教程及随附工具用于其他商业获利行为。**
> 3. 对教程、文档或工具进行二次修改、转载和再分发时，必须保留原作者 `zlion`、原始版权说明及本免责声明，并明确标注原始来源；不得删除署名、冒充原创或以修改版本替代原始来源。
> 4. 固件刷写、分区读写、Telnet和 MTD 操作均可能造成数据丢失、设备变砖、网络中断、保修失效或其他不可预期后果。**操作者应对自己的设备和全部操作结果负责并自行承担风险。**
> 5. 教程作者不保证本文内容和工具适用于所有机型、硬件批次或固件版本。
> 6. 使用者必须遵守所在地适用的法律法规、网络安全规定、电信管理要求、知识产权规则及设备服务协议，不得用于未经授权的访问、破坏、规避安全机制或侵犯第三方权益。
> 7. 教程中涉及的固件、商标、产品名称及第三方程序，其权利归各自权利人所有。如本文内容存在侵权、错误引用或其他权益问题，请联系作者核实；确认后将及时删除或修正相关内容。


---

## 1. Config.bin的加解密与Telnet开启

### 1.1 Config.bin的加解密

`config.bin` 不是明文 XML，而是 ZTE 的 Type-4 外层容器。E2631 中使用的是 AES-256-CBC。

| 参数 | 值 |
|------|------|
| `keySeed` | `E2631Key13515211` |
| `ivSeed` | `E2631Iv13515211` |
| `key` | `SHA256(keySeed)` |
| `iv` | `SHA256(ivSeed)` 取前 16 字节 |
| 填充 | 关闭，尾部补零到 AES 块边界 |

解包流程如下：

```mermaid
graph TD
    A[config.bin Type-4 外层容器] --> B[AES-256-CBC 解密]
    B --> C[Type-0 内层容器]
    C --> D[zlib 解压]
    D --> E[最终 XML]
```

回写流程如下：

```mermaid
graph TD
    A[修改后的 XML] --> B[zlib 压缩]
    B --> C[封装 Type-0 内层容器]
    C --> D[AES-256-CBC 加密]
    D --> E[写回 config.bin]
```

### 1.2 修改Config.bin开启Telnet

这里提供一个一键开启Telnet脚本[config_telnet_tool.zip](https://wwbnc.lanzoub.com/iQRhI40ks1ze)
其中也包含解、加密脚本，可以自行研究，解包得到可修改的XML文件，在打包回去使用。
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

> [!CAUTION] 注意
> 本工具由 zlion 原创，首发于 zlion.top
> 禁止二次修改、倒卖、打包收费或用于任何商业盈利行为，转载须保留完整出处及作者署名。

> [!WARNING] 使用说明
> 1. 将 config-telnet-enable.bat、zte_config_telnet_enable.js 和 config.bin 放在同一个目录。
> 2. 双击 config-telnet-enable.bat。
> 3. 成功后，同目录会生成 config-telnet-enable.bin。
> 4. Telnet 端口为 23，账号和密码均为 admin。
> 
> 运行环境：
> 1. 完整工具包已经附带 node.exe，无需安装 Node.js 或 Python。
> 2. 请勿单独移动 BAT；config-telnet-enable.bat、zte_config_telnet_enable.js 和 node.exe 必须放在一起。
> 3. 不支持的配置会显示“config文件暂不适配”，且不会保留输出文件



## 2. 本机信息优化

> [!NOTE] 本节提醒
> 适用于保留自己设备的 Tag、Wi-Fi 等身份和无线数据分区
> 适用于刷了三方固件无法绑定APP（未验证APP端是否只检测设备型号，仅猜测）


### 2.1 Tag 、Wifi 分区读取

在路由器 Telnet 终端执行：

```sh
cat /proc/mtd
```
可以看到：
```text
mtd2: 00100000 00020000 "tag"
mtd3: 00100000 00020000 "wifi"
```
两者大小均为 `0x100000`，即 `1048576` 字节。

读取 Tag 分区：

```sh
cat /dev/mtd2 > /tmp/tag-backup.bin
```
读取 Wi-Fi 分区：

```sh
cat /dev/mtd3 > /tmp/wifi-backup.bin
```


确认电脑的 TFTP 服务已启动，根目录允许写入。

上传 Tag 备份：

```sh
tftp -p -l /tmp/tag-backup.bin -r tag-backup.bin 192.168.5.7
```

上传 Wi-Fi 备份：

```sh
tftp -p -l /tmp/wifi-backup.bin -r wifi-backup.bin 192.168.5.7
```

 `192.168.5.7`请替换为电脑当前的局域网 IP。

### 2.2 修改Tag分区信息

这里只需要修改型号就可以了，使用脚本[改型号为E2631](https://wwbnc.lanzoub.com/iLgJ840kyibi)
把`tag-backup.bin`重命名为`tag.bin`，跟脚本放在同一级目录下，双击脚本即可得到`tag-E2631.bin`。这样其他信息都还是自己贴纸上的
提醒一下tag分区是有校验的，Tag 中同时存在主 MAC、第二 MAC、MAC 前缀、D-SN、S/N、型号和 Wi-Fi 信息。`F87***` 只是 MAC 前三字节，不是完整 MAC；

![Tag分区信息](./tag.png)

Tag结构：

```text
0x00  33 33 33 33
0x04  payload size，小端
0x08  CRC32，小端
0x0c  记录区
尾部  ff 填充
```

CRC32 计算范围从 `0x0c` 开始，长度由 payload size 决定。

### 2.3 使用 TFTP 将 Tag 和 Wi-Fi 分区传输到路由器中去

本节先把待写入的 `tag-E2631.bin`、`wifi.bin` 以及方案二使用的 `mtd_write_tag_armv7` 传入路由器。以下示例中 `192.168.5.7` 是电脑的 TFTP 地址，请按实际网络修改。

电脑端 TFTP 根目录至少放置：

```text
tag-E2631.bin
wifi.bin
mtd_write_tag_armv7
```

路由器 Telnet 端执行：

```sh
ping -c 3 192.168.5.7

rm -f /tmp/tag.bin /tmp/wifi.bin /tmp/mtd_write_tag
tftp -g -l /tmp/tag.bin -r tag-E2631.bin 192.168.5.7
tftp -g -l /tmp/wifi.bin -r wifi.bin 192.168.5.7
tftp -g -l /tmp/mtd_write_tag -r mtd_write_tag_armv7 192.168.5.7

```


### 2.3.1 写入分区：方案一，使用 cat 命令

方案一只适用于已经确认驱动允许通过字符设备写入的设备。先写入 Tag，再写入 Wi-Fi：

```sh
cat /tmp/tag.bin > /dev/mtd2
sync

cat /tmp/wifi.bin > /dev/mtd3
sync
```

写入后立即在路由器端读回并校验：

```sh
rm -f /tmp/tag-readback.bin /tmp/wifi-readback.bin
cat /dev/mtd2 > /tmp/tag-readback.bin
cat /dev/mtd3 > /tmp/wifi-readback.bin

md5sum /tmp/tag.bin /tmp/tag-readback.bin
md5sum /tmp/wifi.bin /tmp/wifi-readback.bin
```

两个输入文件与读回文件的 MD5 必须分别一致。Tag 分区如果出现以下任一情况，应停止操作并改用方案二：

- 出现 `can not create /dev/mtdblock2: Success` 或类似写入错误；
- 命令返回但读回 MD5 不一致；

`cat` 对 NAND 的写入可能绕过必要的擦除、坏块和 ECC 处理；因此命令“没有明显报错”不等于写入成功。

### 2.3.2 写入分区：方案二，使用 MTD 写入工具

方案二使用已经我自己编译好的 ARMv7 程序 `mtd_write_tag_armv7`。先确认程序架构和用法：

```sh
chmod +x /tmp/mtd_write_tag
/tmp/mtd_write_tag
```

应能看到类似用法：

```text
Usage: /tmp/mtd_write_tag --yes /dev/mtdN file.bin
Example: /tmp/mtd_write_tag --yes /dev/mtd2 /tmp/tag.bin
```

确认输入文件为目标 Tag 分区大小后执行写入：

```sh
ls -l /tmp/tag.bin
cat /sys/class/mtd/mtd2/name 2>/dev/null
cat /sys/class/mtd/mtd2/size 2>/dev/null

/tmp/mtd_write_tag --yes /dev/mtd2 /tmp/tag.bin
sync
```

程序会按 MTD 的擦除块、写入页和坏块规则执行。出现 `MEMUNLOCK warning: Operation not supported` 或 `fsync: Invalid argument` 时，不要只根据警告判断成功或失败，必须继续做读回校验。

这里只在路由器端比较写入前后的 MD5：

```sh
md5sum /tmp/tag.bin

rm -f /tmp/tag-readback.bin
cat /dev/mtd2 > /tmp/tag-readback.bin
md5sum /tmp/tag-readback.bin
```

两次 MD5 一致，且程序没有报告擦除、写入或读回错误，才可认为 Tag 写入成功。若不一致，保留终端输出，不要重启设备，也不要继续写 Bootloader 或整片 NAND。该工具只针对 Tag 的 MTD 写入流程，不能未经验证地用于 Wi-Fi、Bootloader 或其他分区。

---

## 3. Web 本地上传分析研究(失败)

### 3.1 接口和表单

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

### 3.2 SessionTimeout 的定位

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

### 3.3 升级管理器调试

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

### 3.4 请求体上限实验

| 测试文件 | 现象 | 判断 |
|---|---|---|
| 约 300 KiB | 100% 后 HTTP 200，`SessionTimeout` | 能传完整请求，会话仍有问题，不进入后续校验升级流程 |
| 约 320 KiB | 100% 后连接重置 | 接近/超过请求体限制 |
| 约 330 KiB | 约 97% 后重置 | multipart 边界也占空间 |
| 约 512 KiB | 约 62% 后重置 | 设备端提前关闭连接 |
| 22,413,844 bytes 官方包 | 无法稳定上传 | 不适合该页面 |

![300Kib测试](./300.png)
![320Kib测试](./330.png)


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

### 3.5 `/tmp` 空间与 Web 限制

当时读取到：

```text
/var/tmp 约 40960 KiB，可用约 40736 KiB
/tmp     约 20480 KiB，可用约 19456 KiB
```

官方包约 21.38 MiB，在 `/var/tmp` 理论上放得下，但仍被 HTTP 处理路径重置。磁盘空间够，不等于请求体限制够，也不等于升级进程会接受该文件。

### 3.6 Wireshark 过滤器

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

### 3.7 Web 结论

- 进度 100% 只表示浏览器发完请求，不表示设备已经校验或写入。
- `SessionTimeout` 和 `ERR_CONNECTION_RESET` 必须分层排查。
- 大于几百 KiB 的正式升级包应走 Telnet + TFTP + 本地 `fw_flashing`，不应继续反复提交 Web 请求。
- 隐藏页面是真实刷写路径，不是安全测试页面。

---

## 4. fw-flashing 与双槽启动

### 4.1 参数和最小失败测试

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

### 4.2 官方包成功刷写

电脑端确认：

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

### 4.3 确认当前槽

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

### 4.4 已确认的检查方向

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

## 5. 试错部分

### 5.1 Tag 直接写失败

尝试 `cat`、`mknod`、`chmod` 后仍 MD5 不一致。设备没有 `mtd_debug`、`flash_erase`、`nandwrite`，只有 `boot_flashing`、`fw_flashing`、`upgradetest` 等程序；这些程序没有暴露 Tag 写入接口。

最终只能自己编译使用 MTD ioctl 工具（`mtd_write_tag`）成功。

### 5.2 Web 反复失败

```text
升级准备失败：SessionTimeout
上传连接失败
net::ERR_CONNECTION_RESET
```

进度 31%、62%、97%、100% 后都见过失败。用小文件先拆分问题，确认大文件主要被 HTTP 路径限制后，停止继续提交 22 MiB 包，转为本地 `fw_flashing`。

### 5.3 `boot_flashing` 误用风险

```sh
/bin/boot_flashing

/bin/boot_flashing -h
```

它要求特定带头的 Boot 文件，并直接涉及 `/dev/mtd1`。从 `/dev/mtd1` 读出的裸 1 MiB 备份不能直接作为其输入。

### 5.4 未验证自动回退

已确认 `currentverphyaddr`、`backverphyaddr`、两个 `IsBad` 字段，但没有故意刷坏包测试回退。不要为了得到一个“确定答案”而破坏当前可启动槽。

---

## 6. 通用脚本（自行研究使用）

### 6.1 目录

```text
common-tools/
  README.md
  zte_type4_config_tool.js
  enable_telnet_admin.bat
  zte_firmware_analyzer.js
  fw_slot_status.sh
  mtd_write_tag_armv7
```
[点此下载](https://wwbnc.lanzoub.com/i5JFl40o809g)

### 6.2 Type-4 配置工具

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

### 6.3 固件分析工具

```bash
node common-tools/zte_firmware_analyzer.js firmware.bin

node common-tools/zte_firmware_analyzer.js firmware.bin --signature-offset 0x37fdcc --signature-size 564
```

输出哈希、可打印头部、型号标记、uImage CRC 和指定签名区零字节数。签名区非零不等于签名有效。


### 6.4 写 Tag 最小清单

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

只有工具报告成功且输入/读回 MD5 一致才继续。任何不一致都应停止并保存日志，不要马上重启；Tag 写入优先使用 2.3.2 的 MTD 工具方案。

---
