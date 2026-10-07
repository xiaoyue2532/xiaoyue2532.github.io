---
title: 华为主线内核移植尝试5 完善设备树与内核配置
date: 2026-05-30 23:00:02 +0800
categories: [ Kernel ]
---

图省事可以直接使用postmarketos的kernel config，他的配置完全可以识别ufs上的存储。
不过很多驱动被构建成了模块，比如启用USB后发现外接键盘无法输入。而install_module同时strip安装，会让ramdisk.img大小超限。
可以关闭Enable loadable module support。

```shell
curl -O https://gitlab.postmarketos.org/postmarketOS/pmaports/-/raw/main/device/testing/linux-postmarketos-qcom-sm8250/config-postmarketos-qcom-sm8250.aarch64
```

<写新的ramdisk.img、init，测试dtb功能，为下一步启动alpine做准备>

## fsck 自愈

静态编译 e2fs

```shell
CC=aarch64-unknown-linux-musl-clang \
LD=aarch64-unknown-linux-musl-ld.lld \
./configure \
    --prefix=/usr \
    --host=aarch64-unknown-linux-musl \
    LDFLAGS="-static"
```

## 完善 init 脚本

```shell
#!/bin/sh

PATH=/bin
export PATH

echo "init: mount filesystems"
mount -t devtmpfs devtmpfs /dev
mount -t proc proc /proc
mount -t sysfs sysfs /sys

echo "init: run mdev"
echo /bin/mdev > /proc/sys/kernel/hotplug
mdev -s

echo "init: wait for userdata to appear"
while [ ! -b /dev/sda40 ]; do
    sleep 1
done

echo "init: mount userdata and move mountpoints"
e2fsck /dev/sda40
mount /dev/sda40 /mnt
mount --move /dev /mnt/dev
mount --move /proc /mnt/proc
mount --move /sys /mnt/sys

echo "init: black magic! switch_root now"
exec switch_root /mnt /usr/sbin/init

echo "init: seems you dropped out"
exec /bin/sh
```

# 主线 kernel 接受多种 initrd 压缩格式

```shell
# 新版本 lz4 initrd 似乎内核不认
# ramdisk: LZ4 compressed data (v1.4+)
# 注意使用 -l legacy 参数，
# ramdisk: LZ4 compressed data (v0.1-v0.9)
find . | cpio -H newc -o | lz4 -l > ../ramdisk

./mkbootimg.py \
    --header_version 0 \
    --os_version 12.0.0 \
    --os_patch_level 2021-12 \
    --kernel out-ramdisk/kernel \
    --ramdisk out-ramdisk/ramdisk \
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
