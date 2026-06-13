---
title: 华为主线内核移植尝试6 移植Alpine文件系统
date: 2026-06-01 20:00:02 +0800
categories: [ Kernel ]
---

选择alpine是因为它小小的也很可爱

https://mirrors.nju.edu.cn/alpine/latest-stable/releases/aarch64/alpine-minirootfs-3.23.4-aarch64.tar.gz

## chroot

```shell
export CMLFS=/home/arch/alpine

touch $CMLFS/etc/resolv.conf

sudo arch-chroot "$CMLFS" /usr/bin/env -i \
    HOME=/root \
    TERM="$TERM" \
    PS1='(chroot) [\u@\h \W]\$ ' \
    PATH="/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin" \
    /bin/sh
```

## 安装基本软件包

替换软件源镜像，修改/etc/apk/repositories
```
https://mirror.nju.edu.cn/alpine/latest-stable/main
https://mirror.nju.edu.cn/alpine/latest-stable/community
```

```shell
apk add bash coreutils kmod openrc
```

/system/bin/busybox mount /dev/sda40 /mnt

## 静态编译文件系统mount工具

mount userdata分区时dmesg报告
```
F2FS-fs (sda40): using deprecated layout of large_nat_bitmap, please run fsck v1.13.0 or higher to
F2FS-fs (sda40): failed to get valid F2FS checkpoint
mount: mounting /dev/sda40 on /mnt failed: Invalid argument
```

```
# e2fsprogs
LDFLAGS=“-static” ./configure --prefix=/usr

# f2fs-tools
LDFLAGS="-static" ./configure --prefix=/usr
```

其实压根不用
```shell
fastboot format:f2fs userdata
fastboot format:ext4 userdata
```

## 让朴素的initrd进化->带领我们进入真正的Rootfs

参见https://dylanunicorn.github.io/linux-notes/%E9%98%B6%E6%AE%B5%E4%B8%83%EF%BC%9Av1.0-switch_root-%E5%94%A4%E9%86%92%E7%9C%9F%E5%AE%9E%E7%B3%BB%E7%BB%9F

```shell
#!/bin/sh

echo "Mount initrd filesystem"
/bin/busybox mount -t proc proc /proc
/bin/busybox mount -t sysfs sysfs /sys
/bin/busybox mount -t devtmpfs devtmpfs /dev

echo "Trigger a hotplug"
echo '/sbin/mdev' > /proc/sys/kernel/hotplug
/bin/busybox mdev -s

# (REMOVE ME) 这一步很重要 mdev 探测 ufs/scsi 设备需要一段时间 避免还没有等到scsi设备出现就挂载
# (REMOVE ME) 注意 while 内的空格
echo "Wait for rootfs to present"
while [ ! -b /dev/sda40 ]; do
    sleep 1
done

echo "Mount rootfs"
/bin/busybox mount /dev/sda40 /mnt

echo "Move mountpoints"
/bin/busybox mount --move /sys /mnt/sys
/bin/busybox mount --move /proc /mnt/proc
/bin/busybox mount --move /dev /mnt/dev

echo "Black Magic: switch_root now!"
exec switch_root /mnt /sbin/init

# (REMOVE ME) 原帖说这样可以保障 switch_root 失败还能有 Emergency Shell 用
# (REMOVE ME) 可是我之前跳过等待设备节点出现就挂载时的失败情形中没有起作用
echo "Seems you dropped out"
exec /bin/sh
```

开启Enable loadable module support，可以开启Module compress
```shell
make \
    CC="sccache clang" \
    ARCH=arm64 LLVM=1 O=build \
    INSTALL_MOD_STRIP=1 \
    INSTALL_MOD_PATH="$(pwd)/_install" \
    modules_install

cp -r $(pwd)/_install/lib $ALPINE_ROOTFS

# chroot to ALPINE_ROOTFS and execute depmod 6.18.34-dirty

sudo tar -cf - * | gzip > ../alpine.tar.gz
```

刷入emergency boot&ramdisk，格式化 /dev/sda40为ext4，busybox tar -zxf $EXTERNAL_STORAGE/alpine.tar.gz