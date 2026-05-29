---
title: 华为主线内核移植尝试3 Basic DeviceTree
date: 2026-05-29 17:00:02 +0800
categories: [ Kernel ]
---

上一回我们做到用mainline kernel + vendor dtb启动卡第一屏，我想是不是
mainline kernel(6.6) 不认识 vendor dtb(4.19)，于是本节我们写个匹配的devicetree，
看看能否改善问题。

## 问题引入

很庆幸骁龙865(sm8250)已经为主线内核所支持，我们为设备写一个简朴的devicetree。
并将该dts加入Makefile。重新构建

```makefile
dtb-$(CONFIG_ARCH_QCOM)	+= sm8250-huawei-dby-w09.dtb
```

重新打包 boot.img 并刷入

```shell
cp $KERNEL_OUT_DIR/sm8250-huawei-dby-w09 ./dtb
./magiskboot repack boot.img
fastboot flash boot new-boot.img
```

很遗憾的是，第一屏LOGO后很快跳到了fastboot界面。

## 问题分析

应该是android 9之后google提出了dtb overlay方案，旨在实现上游代码与下游厂商代码分离。
简而言之boot.img(Image.gz-dtb/Image-dtb/Image and dtb)中的dtb仅包含基本的配置信息，
启动时由bootloader弥合boot和dtbo分区中的差异，将dtbo分段中的信息盖在dtb表面。
具体参见https://source.android.com/docs/core/architecture/dto/compile。

这也是为什么反编译boot.img中dtb文件，board-id为0 0，而从Android /proc/device-tree里面查询
qcom,board-id为0x0000 206A 0000 0000。

```shell
# 在 Android Shell 中查询实际生效的 devicetree 属性
xxd /proc/device-tree/qcom,board-id
```

明白了dtb overlay的基本道理，为何使用mainline dtb会跳转fastboot呢？https://blog.xzr.moe/archives/398/
提到，如果在kernel dtb编译时加入-@参数，输出dtb不会生成__symbol__符号，这种dtb没法和dtbo中的合并、而且bootloader
也拒绝接受。

### 创建最小的 dtbo.img

为避免vendor dtbo.img内容覆盖mainline dtb，我们需要创建一个最小的dtbo映像。
这样的dtbo映像是什么样子的？参见https://github.com/barni2000/dtbo-lk2nd。
就是给 dts 写上最基本的 qcom,board-id 即可。我注意到该项目是采用`mkfdimg`和cfg文件生成的，
我想也没有必要，直接`mkfdimg create`即可

进入实操阶段，参见https://zhouym.tech/2022/Protect-battery-on-Android，
有前辈为我们提供了预编译的`mkfdimg`工具。

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

> postmarketos wiki说三星机型提供一个空的dtbo分区是不可行的，还需要对其进行avb签名。
> ```shell
> # 从 AOSP 下载 avbtool 工具
> git clone https://mirrors.bfsu.edu.cn/git/AOSP/platform/external/avb.git --depth=1
> ```

> 网友也提供了手动编译 overlay 风格 dtb 的方案，可以供参考
> https://bbs.deepin.org.cn/phone/zh/post/289590

### 修改 mainline kernel dtb 编译参数

在内核源码树中搜索DTC_FLAGS，看到allwinver等厂商已经有考虑到生成dt overlay了，
直接为我们的dtb使用即可。

```makefile
# Enable support for device-tree overlays
DTC_FLAGS_sm8250-huawei-dby-w09 += -@
```

### 重新组装

```shell
cp $KERNEL_OUT/arch/arm64/boot/dts/qcom/sm8250-huawei-dby-w09.dtb dtb
./magiskboot repack boot.img
fastboot flash boot new-boot.img
fastboot reboot
```

我观察到的是启动不再跳转 fastboot，可惜的是和第二话的现象一样，卡第一屏。