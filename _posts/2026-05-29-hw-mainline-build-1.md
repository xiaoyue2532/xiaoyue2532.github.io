---
title: 华为主线内核移植尝试1 基本 ramdisk
date: 2026-05-29 17:00:01 +0800
categories: [ Kernel ]
---

社区已提供 MatePad 11 的 Bootloader 解锁方案，为玩家提供了更多的可能。
可华为官方开源的内核源码是不完整的、缺少大量专有驱动，因此无法完全编译。
搜集资料了解到，主线已经合并了对骁龙 865 的驱动支持，并且有 postmarketOS 社区的一些研究工作。

本文以移植主线 6.18 内核为例，将工作过程记录如下。

**在行动之前请熟读本文思路，并小心操作。**

# 概念辨析：boot.img 还是 ramdisk.img？

- `boot.img` 内含 kernel, dtb。该文件可供 Apatch 修补。
- `ramdisk.img` 内含 ramdisk。该文件可供 Magisk 修补。

# 获取工具：工欲善其事，必先利其器。

```shell
# 获取 Linux 稳定版本源码树
git clone https://mirrors.bfsu.edu.cn/git/linux-stable.git

# AOSP 提供的 mkbootimg 系列解包工具，无论是分解 boot.img 还是 ramdisk.img 都有需要
git clone https://mirrors.bfsu.edu.cn/git/AOSP/platform/system/tools/mkbootimg.git

# AOSP 提供的预编译 LLVM 工具链，当然可以选择其他非 AOSP 工具链
git clone https://mirrors.bfsu.edu.cn/git/AOSP/platform/prebuilts/clang/host/linux-x86.git

# （可选）AOSP 提供的 AvbTool，可以为 Android 镜像添加 Avb 签名、解析 vbmeta.img 等
git clone https://mirrors.bfsu.edu.cn/git/AOSP/platform/external/avb.git
```

# 解包 ramdisk.img

## 提取本机 ramdisk.img

```shell
dd if=/dev/block/by-name/ramdisk of=/storage/emulated/0/ramdisk.img
```

## 传输到电脑上备份并解包

在电脑上执行以下命令进行解包操作，将输出的 `mkbootimg` 格式信息保存、后面重新封包会用到：

```shell
./unpack_bootimg.py --boot_img ramdisk.img --format mkbootimg
```

未指定时默认输出目录为 `out/`，检查输出、可以看到 `ramdisk.img` 中包含：

```shell
file kernel
# kernel: empty
file ramdisk
# ramdisk: gzip compressed data, from Unix, original size modulo 2^32 2880000
```

解包 `ramdisk(.cpio.gz)`：

```shell
mkdir ramdisk_dir && cd ramdisk_dir
gzip -dc ../ramdisk | cpio -idv
```

检查 cpio 内容：

```shell
tree -a
# .
# ├── cache
# ├── debug_ramdisk
# ├── dev
# ├── eng
# ├── init
# ├── metadata
# ├── mnt
# ├── patch_avb_key
# ├── patch_hw
# ├── proc
# ├── second_stage_resources
# ├── sys
# └── system
#     └── etc
#         └── ramdisk
#             └── build.prop
# 
# 14 directories, 3 files
```

# 创建基本 ramdisk cpio

参考[让wayland接管android——systemd wifi 以及 gsi](https://zhuanlan.zhihu.com/p/716822748)，
本节聚焦于利用 `busybox-static` 打包自定义的 `ramdisk`。

## 拷贝 busybox

本文采用 `aarch64-unknown-linux-musl-clang` 目标静态编译了 `busybox`，具体步骤不做赘述。

```shell
make CROSS_COMPILE=aarch64-unknown-linux-musl- defconfig
make CROSS_COMPILE=aarch64-unknown-linux-musl- -j`nproc --all`
```

最好使用符号连接而非复制 busybox，`ramdisk.img` 空间小得可怜。

```shell
tree -a
# .
# ├── bin
# │   ├── busybox
# │   ├── ls -> busybox
# │   ├── mdev -> busybox
# │   ├── mkdir -> busybox
# │   ├── mknod -> busybox
# │   ├── mount -> busybox
# │   ├── reboot -> busybox
# │   ├── sh -> busybox
# │   └── umount -> busybox
# ├── dev
# ├── init
# ├── mnt
# ├── proc
# ├── sys
# └── tmp
# 
# 7 directories, 10 files
```

## 编写 init 脚本

这里您可以抄上引用的init内容，我不明白原帖是如何实现拉取lastkmsg.bin，
我的设备很不幸不能透过bootfail_info读取输出信息。于是我做的工作是挂载/data分区到ramdisk的/mnt，
为android设备留言。

```shell
#!/bin/sh

echo "init: mount filesystems"
/bin/mount -t proc proc /proc
/bin/mount -t sysfs sysfs /sys
/bin/mount -t tmpfs tmpfs /tmp

echo "init: mount devtmpfs"
/bin/mount -t devtmpfs devtmpfs /dev
/bin/mknod /dev/console c 5 1
/bin/mknod /dev/null c 1 3

echo "init: mount devpts"
/bin/mkdir /dev/pts
/bin/mount -t devpts devpts /dev/pts

echo "init: run mdev"
/bin/mdev -s

echo "init: starting processes..."
/bin/mount /dev/sda40 /mnt
echo "I camed but no one remembers" > /mnt/msg.txt
/bin/umount /mnt

echo "init: see you again..."
/bin/reboot
```

> 若无法在/data目录下留言，可以考虑 `mdev -s` 有没有刷新 /dev
> 我之前的努力似乎是/dev下根本没有sda40 data设备，刷新后即可

# 组装 ramdisk.img 并测试

完成修改后可以重新打包

```shell
find . | cpio -H newc -o | gzip > ../ramdisk

./mkbootimg.py \
    --header_version 0 \
    --os_version 12.0.0 \
    --os_patch_level 2021-12 \
    --kernel out/kernel \
    --ramdisk out/ramdisk \
    --pagesize 0x00000800 \
    --base 0x00000000 \
    --kernel_offset 0x10008000 \
    --ramdisk_offset 0x12000000 \
    --second_offset 0x00000000 \
    --tags_offset 0x10000100 \
    --board '' \
    --cmdline '' \
    --output ramdisk_new.img
```

我观察到的是设备进入第一屏，没多久后重启。刷回原版ramdisk.img以进入android系统查询有没有我们的留言。

```shell
fastboot flash ramdisk ramdisk_new.img
fastboot reboot

# 启动成功后去查收留言
adb shell
cat /data/msg.txt
```
我观察到的是/data下有留言信息msg.txt，查询内容是留言和/dev的内容。
