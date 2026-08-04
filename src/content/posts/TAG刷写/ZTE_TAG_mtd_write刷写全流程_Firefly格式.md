---
title: 使用 mtd_write_tag 刷写 ZTE TAG 分区
published: 2026-08-04
updated: 2026-08-04
description: 使用自编译的 mtd_write_tag 工具安全刷写并校验 ZTE 路由器 TAG 分区。
tags: [ZTE, TAG分区,Telnet]
category: 路由器
draft: false
pinned: false
slug: zte-tag-mtd-write-flashing-guide
---

# 使用 mtd_write_tag 刷写 ZTE TAG 分区

> 本文档面向已经通过 Telnet 进入 ZTE 路由器 Shell 的用户，介绍如何上传并使用已编译的 `mtd_write_tag` 工具刷写 TAG 分区，以及如何完整读回和校验。
> **适用系统：** Linux 4.1.x / BusyBox 1.17.x
> **适用平台：** ZTE ZXHN E1630 / E2631 / E2633 / E2638 同平台设备

> [!CAUTION] 高风险操作
> 写错 MTD 分区可能导致设备无法启动。只有在 `/proc/mtd` 明确显示 `mtd2` 为 `tag`，且大小为 `0x00100000` 时，才可以执行写入。严禁把本流程中的命令改为 `/dev/mtd0` 或 `/dev/mtd1`。



---

## 1. 操作前准备

- 电脑 TFTP 根目录中放入 [mtd_write_tag_armv7](https://wwbnc.lanzoub.com/iW88t40izsne)。
- 把准备刷入的 TAG 文件命名为 `tag.bin`。
- 确认 `tag.bin` 的大小为 `1,048,576` 字节。
- 电脑与路由器处于同一网络，路由器能够 ping 通电脑。
- 下面示例电脑 IP 为 `192.168.5.7`。如果实际 IP 不同，请替换所有命令中的该地址。

> [!NOTE] 提示
> 本流程不包含编译，也不使用 `cat` 重定向直接写 `/dev/mtd2`。所有擦除和写入均由已经编译好的 `mtd_write_tag` 完成。

---

## 2. 确认 TAG 分区

先执行命令：

```bash
cat /proc/mtd

cat /sys/class/mtd/mtd2/name

cat /sys/class/mtd/mtd2/size

cat /sys/class/mtd/mtd2/erasesize
```

必须看到以下关键结果：

| 检查项 | 期望结果 |
|------|------|
| `/proc/mtd` | `mtd2: 00100000 00020000 "tag"` |
| `name` | `tag` |
| `size` | `1048576` |
| `erasesize` | `131072` |

> [!WARNING] 停止条件
> 任意一项不一致都不要继续写入。先确认当前固件的实际 TAG 分区编号。

---

## 3. 检查网络

```bash
ping -c 3 192.168.5.7
```

应收到 3 个回复且无丢包。若不通，先检查电脑 IP、防火墙、网线和路由器接口地址。

---

## 4. 下载程序和tag分区到路由器

```bash
cd /tmp

tftp -g -l /tmp/mtd_write_tag -r mtd_write_tag_armv7 192.168.5.7

tftp -g -l /tmp/tag.bin -r tag.bin 192.168.5.7
```

> [!NOTE] 提示
> TFTP 命令成功时可能没有任何输出，返回 Shell 提示符即表示传输结束。

---

## 5. 检查下载文件

```bash
ls -l /tmp/mtd_write_tag /tmp/tag.bin

md5sum /tmp/tag.bin
```

- `tag.bin` 必须为 `1,048,576` 字节。
- 记录这里显示的 MD5；写入后必须与读回文件的 MD5 完全一致。
- 如果电脑端也计算了 MD5，此处结果还应与电脑端一致。

---

## 6. 赋予执行权限并确认工具

```bash
chmod +x /tmp/mtd_write_tag

/tmp/mtd_write_tag
```

工具正常时会显示用法：

```text
Usage: /tmp/mtd_write_tag --yes /dev/mtdN file.bin
Example: /tmp/mtd_write_tag --yes /dev/mtd2 /tmp/tag.bin
```

只显示用法是正常现象，说明 ARMv7 程序能够在当前设备运行。

---

## 7. 擦除并写入 TAG

> [!CAUTION] 最后确认
> 再次确认命令中的设备节点是 `/dev/mtd2`，输入文件是 `/tmp/tag.bin`。

执行写入：

```bash
/tmp/mtd_write_tag --yes /dev/mtd2 /tmp/tag.bin
```

本设备上的正常输出形态如下：

```text
mtd=/dev/mtd2 size=1048576 erase=131072 write=2048 input=1048576 erase_len=1048576
MEMUNLOCK warning: Operation not supported
erasing...
writing...
fsync: Invalid argument
```

> [!WARNING] 关于警告
> 该固件的 MTD 驱动可能不支持 `MEMUNLOCK` 和 `fsync`，因此出现上述两条提示不等于写入失败。是否成功只以随后完整读回并校验 MD5 为准。

---

## 8. 同步并完整读回

```bash
sync

cat /dev/mtd2 > /tmp/tag_readback.bin

ls -l /tmp/tag.bin /tmp/tag_readback.bin

md5sum /tmp/tag.bin /tmp/tag_readback.bin
```

> [!IMPORTANT] 成功标准
> 两个文件都必须为 `1,048,576` 字节，并且两行 MD5 必须逐字符完全一致。满足这两个条件，才视为 TAG 刷写成功。

成功示例：

```text
a02af3e738bcb4fbf361aa30b95517f0  /tmp/tag.bin
a02af3e738bcb4fbf361aa30b95517f0  /tmp/tag_readback.bin
```

---

## 9. 把读回文件传回电脑

此步骤可选：

```bash
tftp -p -l /tmp/tag_readback.bin -r tag_readback.bin 192.168.5.7
```

在电脑端再次计算 `tag.bin` 与 `tag_readback.bin` 的 MD5，可形成第二次校验。

---

## 10. 确认型号字符串

```bash
grep -a -o "ZXHN E[0-9]*" /tmp/tag_readback.bin
```

如果写入的是已经修改为 E2631 的 TAG，期望输出：

```text
ZXHN E2631
```

---

## 11. 校验通过后重启

```bash
sync

reboot
```

如果系统没有 `reboot` 命令，可使用设备管理页面重启。不要在 MTD 正在擦写时断电。

---

## 12. 异常处理

| 现象 | 处理方法 |
|------|------|
| `tftp: timeout` | 确认路由器能 ping 通电脑；检查 TFTP 服务是否启动、防火墙是否允许 UDP 69、文件是否位于 TFTP 根目录。 |
| `No such file or directory` | 检查远程文件名是否为 `mtd_write_tag_armv7` 或 `tag.bin`，并确认大小写。 |
| 工具只显示 `Usage` | 这是正常的自检结果，说明 ARMv7 程序可以运行。继续使用带 `--yes` 的完整命令。 |
| `MEMUNLOCK warning` | 本设备驱动不支持解锁 ioctl 时可出现；继续做完整读回校验。 |
| `fsync: Invalid argument` | 不能单独据此判定失败；执行 `sync`、读回和 MD5 对比。 |
| 两个 MD5 不一致 | 不要重启。重新下载已知正确的 `tag.bin`，再次写入并读回，直到 MD5 一致。 |
| `mtd2` 名称或大小不一致 | 立即停止，不执行写入；重新确认当前设备分区表。 |

---

## 13. 禁止操作

- 不要使用 `cat /tmp/tag.bin > /dev/mtd2` 或写入 `/dev/mtdblock2`；本设备已验证这种方式会失败。
- 不要把 TAG 写到 `/dev/mtd0`（整片 Flash）或 `/dev/mtd1`（Bootloader）。
- 不要写入尺寸不是 `1,048,576` 字节的 TAG 文件。
- 不要在 `erasing...` 或 `writing...` 阶段断电或重启。
- 不要在读回 MD5 不一致时重启设备。

---

## 14. 最终检查清单

- [ ] `/dev/mtd2` 的名称为 `tag`。
- [ ] 输入文件和读回文件均为 1 MiB。
- [ ] 输入文件与读回文件的 MD5 完全一致。
- [ ] 读回文件中的目标型号字符串正确。
- [ ] 所有检查通过后才执行重启。

---

> **文档版本：** 2026-08-04
> **适用系统：** Linux 4.1.x / BusyBox 1.17.x
> **适用平台：** ZTE ZXHN E1630 / E2631 / E2633 / E2638 同平台设备
