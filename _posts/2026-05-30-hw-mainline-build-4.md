---
title: 华为主线内核移植尝试4 使用mkbootimg工具解包与打包
date: 2026-05-29 17:00:02 +0800
categories: [ Kernel ]
---

参见https://mainlining.dev/2021/03/02/booting-mainline-kernel/，提出了使用simplefb的方法

到达这个阶段，小小 magiskboot 已经不足以满足我们的胃口了

## 解包

```shell
# 下载 mkbootimg 工具
git clone https://mirrors.bfsu.edu.cn/git/AOSP/platform/system/tools/mkbootimg.git

# 解包 boot.img 注意复制输出的参数 重新打包会用到
./unpack_bootimg.py \
    --boot_img boot.img \
    --format mkbootimg \
    --out out-boot

# 解包 ramdisk.img 注意复制输出的参数 重新打包会用到
./unpack_bootimg.py \
    --boot_img ramdisk.img \
    --format mkbootimg \
    --out out-ramdisk
```

分析后注意到boot.img中的kernel 是 Image.gz,ramdisk 是 ramdisk.cpio.gz.
ramdisk.img中的kernel是空文件,ramdisk是ramdisk.cpio.gz.

```shell
export KERNEL_SRCTREE=/home/arch/sources/linux-6.6
cp $KERNEL_SRCTREE/arch/arm64/boot/Image.gz out-boot/kernel

# 可以采用任意的 initramfs 格式,只要 kernel config 中配置了即可
find . | cpio -H newc -o | zstd > ../ramdisk.cpio.zstd
cp ramdisk.cpio.zstd out-ramdisk/ramdisk
```

## 打包

修改为boot.img的cmdline并打包

> https://linux-sunxi.org/Mainline_Kernel_Howto#Early_printk 提到
> 要设定bootargs为console=tty1 才能够在simplefb看到输出 我这里实验结果是console=tty0

> 给 cmdline 置空,在kernel config中设定 CONFIG_CMDLINE="console=tty0 loglevel=5"
> 同时启用CONFIG_CMDLINE_FORCE,这么做拒收来自bootloader的cmdline
> 我注意到 bootloader 会发来 console=NULL 采用以上举措后可以看到bootlogo

> 也没有必要给 vendor boot.img 设定 loglevel，bootloader会追加loglevel=0到cmdline
> 同时 vendor kernel 也没开启 DEBUG 宏或 CONFIG_DYNAMIC_DEBUG，dev_dbg 信息压根看不见

```shell
./mkbootimg.py \
    --header_version 2 \
    --os_version 11.0.0 \
    --os_patch_level 2020-11 \
    --kernel out-boot/kernel \
    --ramdisk out-boot/ramdisk \
    --dtb out-boot/dtb \
    --pagesize 0x00001000 \
    --base 0x00000000 \
    --kernel_offset 0x00008000 \
    --ramdisk_offset 0x02000000 \
    --second_offset 0x00000000 \
    --tags_offset 0x00000100 \
    --dtb_offset 0x0000000001f00000 \
    --board '' \
    --cmdline '' \
    --output new-boot.img
```

图省事可以直接使用postmarketos的kernel config，他的配置完全可以识别ufs上的存储。
不过很多驱动被构建成了模块，比如启用USB后发现外接键盘无法输入。而install_module同时strip安装，会让ramdisk.img大小超限。
可以关闭Enable loadable module support。

```shell
curl -O https://gitlab.postmarketos.org/postmarketOS/pmaports/-/raw/main/device/testing/linux-postmarketos-qcom-sm8250/config-postmarketos-qcom-sm8250.aarch64
```

## SimpleFB/SimpleDRM

可以考虑使用simpleDRM驱动代替simpleFB,都能匹配compatible="simpleframebuffer"使用起来没有区别。

显示开启后dmesg 10s多提示rsc等待/psci/power-domain-cpu-cluster0

需要开启CONFIG_ARM_PSCI_CPUIDLE_DOMAIN，然后simplefb显示会一闪而过

这里就需要为simplefb节点添加power-domain和clocks避免未使用的节点清理，
拷贝arch/arm64/boot/dts/qcom/sm8250-sony-xperia-edo.dtsi，然后添加cmdline 'clk_ignore_unused'


## USB功能、UFS存储

devicetree配置直接复制xiaomi-pad-elish-common.dtsi。可以看到该设备只启用了USB2.0，我这里注释了这些USB2.0属性后发现USB用不了。

## 小玩法

修改include/linux/uts.h的UTS_SYSNAME为HongMeng Kernel