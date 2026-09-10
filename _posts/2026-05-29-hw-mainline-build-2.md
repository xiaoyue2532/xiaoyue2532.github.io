---
title: 华为主线内核移植尝试2 基本内核映像
date: 2026-05-29 17:00:02 +0800
categories: [ Kernel ]
---

# 查询当前内核所需工具链的最低版本

可以查询 kernel 源码树中 `scripts/min-tool-version.sh` 以了解所需最低版本工具链。

| LTS kernel | 最低LLVM版本 |
| ---------- | ------------ |
| linux-6.18 | 15.0.0       |
| linux-6.12 | 13.0.1       |
| linux-6.6  | 11.0.0       |
| linux-6.1  | 11.0.0       |

# 切换 LLVM 工具链

本文选择了 AOSP 提供的预编译工具链。截至目前，最新 tag 版本为 `ndk-r29`，
阅读 `README.md` 得知 `Android Platform Currently clang-r530567`。

```shell
cd linux-x86
git checkout ndk-r29

./clang-r530567/bin/clang -v
# Android (12328485, +pgo, +bolt, +lto, +mlgo, based on r530567) clang version 19.0.0 (https://android.googlesource.com/toolchain/llvm-project 97a699bf4812a18fb657c2779f5296a4ab2694d2)
# Target: x86_64-unknown-linux-gnu
# Thread model: posix
# InstalledDir: /home/ubuntu/linux-x86/clang-r530567/bin
```

# 配置内核

获取 [config-postmarketos-qcom-sm8250.aarch64](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/raw/c66aa62e36a61ca9fb62da2e894ed87d3b5a32c6/device/testing/linux-postmarketos-qcom-sm8250/config-postmarketos-qcom-sm8250.aarch64) 现成的 config (6.17.0)。

可以使用 `ccache` 缓存以加速重新编译(recompile)。

```shell
KBUILD_BUILD_TIMESTAMP='' \
make \
    ARCH=arm64 LLVM=1 \
    CC="ccache clang" \
    menuconfig

KBUILD_BUILD_TIMESTAMP='' \
make \
    ARCH=arm64 LLVM=1 \
    CC="ccache clang" \
    -j`nproc --all`
```

检查产出内核映像

```shell
file arch/arm64/boot/Image.gz
# arch/arm64/boot/Image.gz: gzip compressed data, max compression, from Unix, original size modulo 2^32 42064384
```

# 解包 boot.img

## 提取本机 boot.img

```shell
dd if=/dev/block/by-name/boot of=/storage/emulated/0/boot.img
```

## 传输到电脑上备份并解包

在电脑上执行以下命令进行解包操作，将输出的 `mkbootimg` 格式信息保存、后面重新封包会用到：

```shell
./unpack_bootimg.py --boot_img boot.img --format mkbootimg
```

未指定时默认输出目录为 `out/`，检查输出、可以看到 `ramdisk.img` 中包含：

```shell
file dtb
# dtb: Device Tree Blob version 17, size=555343, boot CPU=0, string block size=47887, DT structure block size=507400
file kernel
# kernel: gzip compressed data, max compression, from Unix, original size modulo 2^32 57466896
file ramdisk
# ramdisk: gzip compressed data, from Unix, original size modulo 2^32 1093888
```

`kernel` 对应 `Image.gz`, `dtb` 是编译过的设备树文件。解包 `ramdisk(.cpio.gz)`

```shell
mkdir ramdisk_dir && cd ramdisk_dir
gzip -dc ../ramdisk | cpio -idv
```

检查 cpio 解包内容，发现几乎无用、主要内容已经出现在 `ramdisk.img` 中。

```shell
tree -a
# .
# ├── fstab.qcom
# └── system
#     └── bin
#         └── e2fsck
# 
# 3 directories, 2 files
```


# 组装

```shell
./mkbootimg.py \
    --header_version 2 \
    --os_version 11.0.0 \
    --os_patch_level 2020-11 \
    --kernel out/kernel \
    --ramdisk out/ramdisk \
    --dtb out/dtb \
    --pagesize 0x00001000 \
    --base 0x00000000 \
    --kernel_offset 0x00008000 \
    --ramdisk_offset 0x02000000 \
    --second_offset 0x00000000 \
    --tags_offset 0x00000100 \
    --dtb_offset 0x0000000001f00000 \
    --board '' \
    --cmdline 'earlycon=msm_geni_serial,0xa90000 androidboot.hardware=qcom androidboot.console=ttyMSM0 androidboot.memcg=1 lpm_levels.sleep_disabled=1 video=vfb:640x400,bpp=32,memsize=3072000 msm_rtb.filter=0x237 service_locator.enable=1 androidboot.usbcontroller=a600000.dwc3 swiotlb=2048 loop.max_part=7 cgroup.memory=nokmem,nosocket reboot=panic_warm unmovable_isolate1=2:256M,3:312M,4:348M buildvariant=user' \
    --output boot_new.img
```

其实该镜像缺乏合适的设备树，压根没法启动。下一节将聚焦于编写合适的设备树。
