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