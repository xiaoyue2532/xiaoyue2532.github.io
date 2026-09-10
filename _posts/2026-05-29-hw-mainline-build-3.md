---
title: 华为主线内核移植尝试3 创建最小的 dtbo.img
date: 2026-05-29 17:00:02 +0800
categories: [ Kernel ]
---

# 前置知识：为何我的设备树文件导致设备启动跳 Fastboot？

Android 9 之后，Google 提出了 DTB Overlay 方案，旨在实现上游代码与下游厂商代码分离。简而言之，`boot.img` 中的“主 DTB”仅包含基本配置信息。
启动时，由 Bootloader 弥合 `boot` 和 `dtbo` 分区中的差异，将 `dtbo` 分段（fragment）中的信息覆盖到 DTB 上。
详见 [Compile and verify](https://source.android.com/docs/core/architecture/dto/compile)。

DTBO 要覆盖到主 DTB 上，必须知道“覆盖到哪里”。这个索引就是主 DTB 里的 `__symbols__` 节点。
只有编译主 DTB 时加上 `-@` 参数，DTC 才会生成 `__symbols__` 节点。DTBO 中的 fragment 通过 label 引用主 DTB 中的节点。
若主 DTB 没有 `__symbols__`，Bootloader 就无法解析 overlay 引用，也就无法完成合并。

`boot.img` 的主设备树有写：

```dts
qcom,board-id = <0x00 0x00>;
```

但从 Android Shell 查询实际生效的 `qcom,board-id`：

```shell
xxd /proc/device-tree/qcom,board-id
# 00000000: 0000 206a 0000 0000                      .. j....
# 即实际生效的是 qcom,board-id = <0x206a 0x0>;
```

# 开始行动：创建最小的 dtbo.img

为避免vendor dtbo.img内容覆盖mainline dtb，我们需要创建一个最小的dtbo映像，其中的 dtbo fragment 只需要最基本的 qcom,board-id。
参见[dtbo-lk2nd](https://github.com/barni2000/dtbo-lk2nd)。

[Protect-battery-on-Android](https://zhouym.tech/2022/Protect-battery-on-Android)，有前辈为我们提供了预编译的`mkfdimg`工具。

```shell
./mkfdimg dump dtbo.img -b dtb

# 注意 dtc 的 -@ 参数不能少
# 将 9 个 DTB 转为 DTC
for i in {0..8}; do ./dtc -@ -I dtb -O dts dtb.$i -o dts.$i; done

# 修改 9 个 dtbo 源码

# 再将 9 个 DTC 转为 DTB
for i in {0..8}; do ./dtc -@ -I dts -O dtb dts.$i -o dtb.$i; done

# 重新包装 dtbo
mkfdimg create new-dtbo.img dtb.{0,1,2,3,4,5,6,7,8}
```

# 准备编写设备树

利用 dtc 工具反编译 `boot.img` 中的 dtb 文件，可以作为编写设备树文件的参考：

```shell
./scripts/dtc/dtc -I dtb -O dts dtb -o dby-w09.dts
```

新建设备树文件 `arch/arm64/boot/dts/qcom/sm8250-huawei-dby-w09.dts`，并将其加入 `arch/arm64/boot/dts/qcom/Makefile`。

```makefile
dtb-$(CONFIG_ARCH_QCOM)	+= sm8250-huawei-dby-w09.dtb
# Enable support for device-tree overlays
DTC_FLAGS_sm8250-huawei-dby-w09 += -@
```

[如何优雅地编译Linux内核并做成boot.img与dtbo文件](https://bbs.deepin.org.cn/phone/zh/post/289590)提供了手动编译 overlay 风格 dtb 的方案，
它也提到了`pd_ignore_unused clk_ignore_unused`等启动参数，可供参考。