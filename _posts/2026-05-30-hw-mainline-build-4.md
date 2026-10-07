---
title: 华为主线内核移植尝试4 基本设备树
date: 2026-05-29 17:00:02 +0800
categories: [ Kernel ]
---

# 开始编写设备树

## SimpleFB/SimpleDRM

通过屏幕显示是测试正常运作的最好方法。

[Booting mainline kernel](https://mainlining.dev/2021/03/02/booting-mainline-kernel/) 提出了使用 `simplefb` 的方法，
开启 `CONFIG_FB_SIMPLE`、同时在设备树中插入 `compatible="simpleframebuffer"` 节点。

可以考虑使用 `simpledrm` 代替 `simplefb`、即开启 `CONFIG_DRM_SIMPLEDRM`。`simplefb` 和 `simpledrm` 都能匹配 `compatible="simpleframebuffer"`。
注意 `CONFIG_FB_SIMPLE` 与 `CONFIG_DRM_SIMPLEDRM` 互斥。非特定说明，后文默认采用 `simpledrm`。

[Mainline Kernel Howto](https://linux-sunxi.org/Mainline_Kernel_Howto#Early_printk) 提到要设置 `console=tty1`，
才能在 framebuffer 上看到输出。我设备的实验结果是 `console=tty0`。

`fbcon=rotate:1` 参数可以让显示顺时针旋转 90 度。

## 拒收 bootloader 发来的额外 cmdline

除 `boot.img` 传递的参数外，bootloader 还会对内核 `cmdline` 产生额外影响：

- 向 `cmdline` 写入 `console=NULL`。此时系统仍有 tty 输出，但 `CONFIG_LOGO` 的企鹅启动图标无法显示。
- 在 `cmdline` 后追加 `loglevel=0`。该参数会覆盖 `boot.img` 设置的 `loglevel=5`，导致后者不生效。

需要设置 `CONFIG_CMDLINE` 为你需要的 cmdline，同时启用 `CONFIG_CMDLINE_FORCE`。打包 `boot.img` 时可以将 cmdline 字段置空。

## 时钟管理

显示开启后 dmesg 10s多提示 rsc 等待 /psci/power-domain-cpu-cluster0

对于 tinyconfig，开启 `CONFIG_ARM_PSCI_CPUIDLE_DOMAIN` 后 `simpledrm` 显示一闪而过。

这里就需要为 `framebuffer` 节点添加 power-domain 和 clocks 避免未使用的节点清理，
拷贝 arch/arm64/boot/dts/qcom/sm8250-sony-xperia-edo.dtsi，然后添加cmdline 'clk_ignore_unused'

## PMIC/电源管理

可以照抄`sm8250-xiaomi-pad-elish-common.dtsi`、SONY等设备的regualtor0~2配置，可是我发现 DBY-W09 上似乎没有 PM8009、相比其他设备多了 PMK8002。

rsc 启用后发现只有 pm8650l 的 regulators 有被设定，而 pm8650 的regulators-0、pm8009的regulator-2报错

```
qcom-rpmh-regulator couldn't find RPMh address for resource for regulator-0 smps1
qcom-rpmh-regulator couldn't find RPMh address for resource for regulator-2 smps4
```

我分析了 `drivers/regulator/qcom-rpmh-regulator.c`，那里是有个循环 `for_each_available_child_of_node_scoped`，
只要有一个 ldo/smps 结点找不到 RPMh address，则`return ret;`、之后结点都不会探测到。
可以 comment 掉`return ret;`,看看有哪些nodes是找不到地址的，在相应的regulators组中删掉该node。

我是删掉了 pm8150 `regualtor-0`的部分节点和整个`regulator-2`，可以看到`regulator-fixed`之外的结点都有在运作。

压根没有pm8009、pmxprairie，但是有pmk8002。

## USB功能、UFS存储

~~devicetree 配置直接复制`sm8250-xiaomi-pad-elish-common.dtsi`。可以看到该设备只启用了 USB2.0，我这里注释了这些 USB2.0 属性后发现 USB 用不了。~~

USB3.0已经运作检查lsusb 3.0设备是否在 3.0 Hub下，注意检查TypeC-usb转接线、USB扩展HUB是否也是3.0的、USB3.0设备插入速度（插入时触点接触先后顺序影响2.0/3.0工作方式识别）才能让USB3.0设备正常运作。

ufs_mem_phy ufs_mem_hs似乎不能配置 `status = 'disable'`，会导致 `simpledrm` 一闪而过后重启。

# 打包

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
    --output boot_new.img
```

图省事可以直接使用 postmarketos 的 kernel config，他的配置完全可以识别ufs上的存储。
不过很多驱动被构建成了模块，比如启用 USB 后发现外接键盘无法输入。而 install_module 同时 strip 安装，会让 ramdisk.img 大小超限。
可以关闭 Enable loadable module support。
