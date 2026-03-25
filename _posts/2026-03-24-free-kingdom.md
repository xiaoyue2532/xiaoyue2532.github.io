---
title: RK3506 折腾 - 3. 驶向必然的自由王国 - 上游
date: 2026-03-24 04:03:00 +0800
categories: [ 嵌入式, RK3506 ]
---

## 上游 idblock.img

尝试烧录 Rockchip 提供的 idblock.img，这个是实在没法做开源替代的。

```shell
git clone https://github.com/rockchip-linux/rkbin.git

cd ./rkbin

./tools/boot_merger ./RKBOOT/RK3506MINIALL.ini
```

## 主线 uboot

## 主线 kernel

## Alpine

Alpine 是基于 musl-libc 的轻量级发行版，我认为它尤其适合 RK3506 这种小平台，同时也为后来自定义发行版铺平道路。

