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

## 不接受来自 bootloader 的 cmdline

CONFIG_CMDLINE为`console=tty0 clk_ignore_unused`
启用CONFIG_CMDLINE_FORCE

## SimpleFB/SimpleDRM

参见https://mainlining.dev/2021/03/02/booting-mainline-kernel/，提出了使用simplefb的方法

可以看到postmarketos默认启用的就是simpleDRM，我还很奇怪为什么权威config也没开simplefb
原来是simpleDRM驱动代替simpleFB,都能匹配compatible="simpleframebuffer"使用起来没有区别。

显示开启后dmesg 10s多提示rsc等待/psci/power-domain-cpu-cluster0

需要开启CONFIG_ARM_PSCI_CPUIDLE_DOMAIN，然后simplefb显示会一闪而过

这里就需要为simplefb节点添加power-domain和clocks避免未使用的节点清理，
拷贝arch/arm64/boot/dts/qcom/sm8250-sony-xperia-edo.dtsi，然后添加cmdline 'clk_ignore_unused'

## PMIC

可以照抄`sm8250-xiaomi-pad-elish-common.dtsi`、SONY等设备的regualtor0~2配置，可是我发现DBY-W09上似乎没有PM8009、相比其他设备多了PMK8002。

rsc启用后发现只有pm8650l的regulators有被设定，而pm8650的regulators-0、pm8009的regulator-2报错

```
qcom-rpmh-regulator couldn't find RPMh address for resource for regulator-0 smps1
qcom-rpmh-regulator couldn't find RPMh address for resource for regulator-2 smps4
```

我分析了`drivers/regulator/qcom-rpmh-regulator.c`,
那里是有个循环`for_each_available_child_of_node_scoped`，只要有一个ldo/spms结点找不到RPMh address，则`return ret;`、之后结点都不会探测到。
可以comment掉`return ret;`,看看有哪些nodes是找不到地址的，在相应的regulators组中删掉该node。

我是删掉了pm8150 `regualtor-0`的部分节点和整个`regulator-2`，可以看到`regulator-fixed`之外的结点都有在运作。

## USB功能、UFS存储

devicetree配置直接复制`sm8250-xiaomi-pad-elish-common.dtsi`。可以看到该设备只启用了USB2.0，我这里注释了这些USB2.0属性后发现USB用不了。

ufs_mem_phy ufs_mem_hs似乎不能禁用，会导致simplefb一闪而过后重启

## 小玩法

修改`include/linux/uts.h`的`UTS_SYSNAME`为`HongMeng Kernel`