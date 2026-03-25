---
title: RK3506 折腾 - 4. 调试 arm-unknown-musleabihf-clang
date: 2026-03-24 04:03:00 +0800
categories: [ 嵌入式, RK3506 ]
---

用 arm-unknown-musleabihf-clang 编译产出的 static ELF 文件，无论是在 arm64 还是 armv7l 真机上都报错 Illegal Instruction。可是 qemu-arm-static 运行倒是没问题。
