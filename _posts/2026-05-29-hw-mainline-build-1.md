---
title: 华为主线内核移植尝试1 Basic ramdisk
date: 2026-05-29 17:00:01 +0800
categories: [ Kernel ]
---

HUAWEI MatePad 11 (2021) 款已经有社区玩家破解了BL锁，为玩家提供了更多的可能。
如今该设备已经停止了系统更新，内核一直停在4.19.157得不到更新。我在尝试chroot容器透过DRM直接显示时
发现该内核的msm-drm驱动有问题，于是决定重新构建厂商内核。从huawei opensource拉到官方内核源码后，
我花费大力气尝试通过编译，可是该源码包缺失的lcdkit等内容太多，不是让社区玩家可以轻松编译的。

于是我上网搜集资料，了解到该设备搭载的骁龙865处理器已经为主线内核所接受，并且有postmarketOS社区研究。
我决定移植一个mainline 6.6内核，并将一系列努力历程记录如下。

**行动请熟读本文思路，小心操作。**

## 解包

下载 magisk，找到符合您架构的libmagiskboot.so，更名为magiskboot并赋予可执行权限。

```shell
./magiskboot unpack ramdisk.img
cpio -iv < ramdisk.cpio
```

## 创建基本 ramdisk cpio

参考https://zhuanlan.zhihu.com/p/716822748，您可以从busybox和原厂ramdisk结构开始

> 我这里采用 aarch64-unknown-linux-musl-clang 工具链，静态编译了一个 busybox
> 您尽可以采用prebuilt的aarch64 static linked busybox，这里不做赘述。

```shell
cd ramdisk
cp ~/busybox system/bin/
cp system/bin/busybox system/bin/sh
touch init
```

这里您可以抄上引用的init内容，我不明白原帖是如何实现拉取lastkmsg.bin，
我的设备很不幸不能透过bootfail_info读取输出信息。于是我做的工作是挂载/data分区到ramdisk的/mnt，
为android设备留言。

```shell
#!/system/bin/sh

echo "INIT: create /kmsg node"
/system/bin/busybox mknod /kmsg c 1 11 -m 0666

echo "INIT: redirect stderr to /kmsg" > /kmsg
exec > /kmsg 2>&1

echo "INIT: before mkdirs" > /kmsg
/system/bin/busybox ls /
[ -d /dev ] || mkdir /dev
[ -d /sys ] || mkdir /sys
[ -d /proc ] || mkdir /proc
[ -d /tmp ] || mkdir /tmp

echo "INIT: after mkdirs"
/system/bin/busybox ls /
/system/bin/busybox mount -t proc proc /proc
/system/bin/busybox mount -t sysfs sysfs /sys
/system/bin/busybox mount -t tmpfs tmpfs /tmp
/system/bin/busybox mount -t devtmpfs devtmpfs /dev

echo "INIT: mount /dev/pts"
/system/bin/busybox mkdir /dev/pts
/system/bin/busybox mount -t devpts devpts /dev/pts
/system/bin/busybox mknod /dev/console c 5 1
/system/bin/busybox mknod /dev/null c 1 3
/system/bin/busybox mdev -s

echo "Starting init process..."
# /system/bin/busybox dpkg
# /system/bin/busybox date
# sleep 5
# /system/bin/busybox
# /system/bin/busybox mount /dev/block/sda40 /mnt
/system/bin/busybox mount /dev/sda40 /mnt
echo "I camed but no one remembers" > /mnt/msg.txt
/system/bin/busybox ls -l /dev >> /mnt/msg.txt
/system/bin/busybox umount /mnt
/system/bin/busybox reboot
```

> 如果您的设备没法在/data目录下留言，可以考虑mdev -s有没有刷新/dev
> 我之前的努力似乎是/dev下根本没有sda40 data设备，刷新后即可

## 组装、刷入并实验

完成修改后可以重新打包

```shell
find . | cpio -H newc -o > ../ramdisk.cpio

export WKDIR_RAMDISK=/home/arch/wkdir/ramdisk
cd $WKDIR_RAMDISK

./magiskboot pack ramdisk.img
fastboot flash ramdisk new-ramdisk.img
```

我观察到的是设备进入第一屏，没多久后重启。刷回原版ramdisk.img以进入android系统查询有没有我们的留言。
```shell
fastboot flash ramdisk ramdisk.img
fastboot reboot

# 启动成功后去查收留言
adb shell
cat /data/msg.txt
```
我观察到的是/data下有留言信息msg.txt，查询内容是留言和/dev的内容。
