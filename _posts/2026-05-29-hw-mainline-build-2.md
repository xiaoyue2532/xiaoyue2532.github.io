---
title: 华为主线内核移植尝试2 Basic Kernel Image
date: 2026-05-29 17:00:02 +0800
categories: [ Kernel ]
---

HUAWEI MatePad 11 (2021) 款已经有社区玩家破解了BL锁，为玩家提供了更多的可能。
如今该设备已经停止了系统更新，内核一直停在4.19.157得不到更新。我在尝试chroot容器透过DRM直接显示时
发现该内核的msm-drm驱动有问题，于是决定重新构建厂商内核。从huawei opensource拉到官方内核源码后，
我花费大力气尝试通过编译，可是该源码包缺失的lcdkit等内容太多，不是让社区玩家可以轻松编译的。

于是我上网搜集资料，了解到该设备搭载的骁龙865处理器已经为主线内核所接受，并且有postmarketOS社区研究。
我决定移植一个mainline 6.18内核，并将一系列努力历程记录如下。

**行动请熟读本文思路，小心操作。**

## 获取 LLVM 工具链

华为 HarmonyOS kernel 构建采用了一个Qualcomm提供的10.0.7的某LLVM工具链，上网搜素了很多资料。
也只有蛛丝马迹，并没有可以直接下载的途径。

我决定从 AOSP 下载一个预编译工具链。对于读者您可以自由选择 LLVM 编译器套件，不用AOSP编译器也行。

```shell
git clone https://mirrors.bfsu.edu.cn/git/AOSP/platform/prebuilts/clang/host/linux-x86.git

# 截至目前，最新ndk版本为 ndk-r27d
git checkout ndk-r27d

clang -v
# Android (11039501, +pgo, +bolt, +lto, +mlgo, based on r498229b) clang version 17.0.4 (https://android.googlesource.com/toolchain/llvm-project e34ed7d63863b45858e74126edaa738e75887800)
# Target: x86_64-unknown-linux-gnu
# Thread model: posix
# InstalledDir: /home/arch/linux-x86/clang-r498229b/bin
# Found candidate GCC installation: /usr/lib/gcc/x86_64-pc-linux-gnu/16.1.1
# Found candidate GCC installation: /usr/lib64/gcc/x86_64-pc-linux-gnu/16.1.1
# Selected GCC installation: /usr/lib64/gcc/x86_64-pc-linux-gnu/16.1.1
# Candidate multilib: .;@m64
# Candidate multilib: 32;@m32
# Selected multilib: .;@m64
```

可以看到该LLVM工具链也没有很新，也只能编译6.6内核。可以查询你的kernel源码树中`scripts/min-tool-version.sh`以了解所需最低版本工具链。

| LTS kernel | 最低LLVM版本 |
| ---------- | ------------ |
| linux-6.18 | 15.0.0       |
| linux-6.12 | 13.0.1       |
| linux-6.6  | 11.0.0       |
| linux-6.1  | 11.0.0       |

## 配置 kernel config

这里我是以厂商config为参考，可以透过`zcat /proc/config.gz`来获得。配置过程太过繁琐，不做赘述。

```shell
# 我这里从 tinyconfig 出发，图简单可以从defconfig出发。
make \
    ARCH=arm64 LLVM=1 \
    tinyconfig

make \
    ARCH=arm64 LLVM=1 \
    menuconfig

make \
    ARCH=arm64 LLVM=1 \
    -j16
```

## 组装

```shell
export WKDIR_BOOT=/home/arch/wkdir/boot
cd $WKDIR_BOOT

./magiskboot unpack boot.img
mv kernel kernel.bak

export KERNEL_SRCTREE=/home/arch/sources/linux-6.6
cp $KERNEL_SRCTREE/arch/arm64/boot/Image kernel

./magiskboot repack boot.img

fastboot flash boot new-boot.img
fastboot reboot
```

我这里看到的是开机卡第一屏，我的情况是ramdisk里面是上一节刷入的busybox留言initramfs，看来没有顺利调起init。
