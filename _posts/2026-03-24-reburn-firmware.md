---
title: RK3506 折腾 - 2. 重新烧录 验证固件分析正误
date: 2026-03-24 02:53:00 +0800
categories: [嵌入式, RK3506]
---

## 创建分区

```shell
parted /dev/sda
# 切换单位到 sector
unit s
# 创建 GPT 分区表
mklabel gpt
# 创建 GPT 分区
mkpart uboot 8192 16383
mkpart boot 16384 40959
mkpart rootfs 65536 -1
# parted 提示可以管理的扇区范围比总扇区数量少33 Ignore
quit
```


## 烧写数据

```shell
dd --help
# 得知默认的单元是 512B=1s

# 从第 64 扇区开始写 idblock
dd if=rk3506_idblock_v1.04.110.img of=/dev/sda seek=64

dd if=uboot.img of=/dev/sda1
#dd if=uboot.img of=/dev/sda seek=8192

dd if=boot.img of=/dev/sda2
#dd if=boot.img of=/dev/sda seek=16384

dd if=rootfs.img of=/dev/sda3
#dd if=rootfs.img of=/dev/sda seek=65536
```
