---
title: 构建 LLVM 交叉工具链-1 入门知识
date: 2026-11-11 01:20:00 +0800
categories: [ LLVM ]
---

LLVM 是一套天生的交叉编译器。

## 选择合适的目标元组

根据[Cross-compilation using Clang](https://clang.llvm.org/docs/CrossCompilation.html)所说：

> 元组有通用的格式 `<arch><sub>-<vendor>-<sys>-<env>` 其中:
> arch = x86_64, i386, arm, thumb, mips 等
> - sub = for ex. on ARM: v5, v6m, v7a, v7m 等
> - vendor = pc, apple, nvidia, ibm 等
> - sys = none, linux, win32, darwin, cuda 等
> - env = eabi, gnu, android, macho, elf 等

参照[Rust Platform Support](https://doc.rust-lang.org/nightly/rustc/platform-support.html), 
本文选择`unknown`作为厂商名，对于不同的平台元组，有：

### x86_64

|         元组名称          |   musl 动态连接器名称    |       compiler-rt 名称        |
| :-----------------------: | :----------------------: | :---------------------------: |
| x86_64-unknown-linux-musl | /lib/ld-musl-x86_64.so.1 | libclang_rt.builtins-x86_64.a |

### AArch64

若选择元组 `armv8-unknown-linux-musl` ，compiler-rt 的文件名就是 `libclang_rt.builtins-armv8.a.`。

事实上，很多 configure 脚本不认识 `armv7-unknown-linux-musl`、`armv8-unknown-linux-musl`，
只认可GNU风格的元组 `arm-unknown-linux-musleabi`、`aarch64-unknown-linux-musl`。

若想指定架构，使用参数 `-march=armv7-a`、`-march=armv8-a`。

|          元组名称          |    musl 动态连接器名称    |        compiler-rt 名称        |
| :------------------------: | :-----------------------: | :----------------------------: |
| aarch64-unknown-linux-musl | /lib/ld-musl-aarch64.so.1 | libclang_rt.builtins-aarch64.a |

### ARM

`arm-unknown-linux-musleabi` 采用软浮点, `clang -cc1` 参数中有 `-mfloat-abi=soft`。
`arm-unknown-linux-musleabihf` 采用硬浮点, `clang -cc1` 参数中有 `-mfloat-abi=hard`。

|           元组名称           |   musl 动态连接器名称   |       compiler-rt 名称       |
| :--------------------------: | :---------------------: | :--------------------------: |
|  arm-unknown-linux-musleabi  |  /lib/ld-musl-arm.so.1  |  libclang_rt.builtins-arm.a  |
| arm-unknown-linux-musleabihf | /lib/ld-musl-armhf.so.1 | libclang_rt.builtins-armhf.a |

### RISCV64

`riscv64-unknown-linux-musl` 默认 `-mabi=lp64d`。本文自定义了 `vendor` 为 `lp64`、`lp64f`、`lp64d` 的 RISCV64 目标。

|         元组名称         |     ABI     |     musl 动态连接器名称      |        compiler-rt 名称        |
| :----------------------: | :---------: | :--------------------------: | :----------------------------: |
| riscv64-lp64-linux-musl  | -mabi=lp64  | /lib/ld-musl-riscv64-sf.so.1 | libclang_rt.builtins-riscv64.a |
| riscv64-lp64f-linux-musl | -mabi=lp64f | /lib/ld-musl-riscv64-sp.so.1 | libclang_rt.builtins-riscv64.a |
| riscv64-lp64d-linux-musl | -mabi=lp64d |  /lib/ld-musl-riscv64.so.1   | libclang_rt.builtins-riscv64.a |

### RISCV32

`riscv32-unknown-linux-musl` 默认 `-mabi=ilp32d`。本文自定义了 `vendor` 为 `ilp32`、`ilp32f`、`ilp32d` 的 RISCV32 目标。

|         元组名称          |     ABI      |     musl 动态连接器名称      |        compiler-rt 名称        |
| :-----------------------: | :----------: | :--------------------------: | :----------------------------: |
| riscv32-ilp32-linux-musl  | -mabi=ilp32  | /lib/ld-musl-riscv32-sf.so.1 | libclang_rt.builtins-riscv32.a |
| riscv32-ilp32f-linux-musl | -mabi=ilp32f | /lib/ld-musl-riscv32-sp.so.1 | libclang_rt.builtins-riscv32.a |
| riscv32-ilp32d-linux-musl | -mabi=ilp32d |  /lib/ld-musl-riscv32.so.1   | libclang_rt.builtins-riscv32.a |

### x86

若选择元组 `i686-unknown-linux-musl`: 

- compiler-rt 的文件名是 `libclang_rt.builtins-i386.a.`;
- musl 加载器 (loader) 的名称是 `/lib/ld-musl-i386.so.1`;
- musl 自身的编译参数是 `-march=i486 -mtune=generic`;

类似的用 `-march=i686 -mtune=generic` 来指定子架构。

|        元组名称         |  musl 动态连接器名称   |      compiler-rt 名称       |
| :---------------------: | :--------------------: | :-------------------------: |
| i386-unknown-linux-musl | /lib/ld-musl-i386.so.1 | libclang_rt.builtins-i386.a |
| i686-unknown-linux-musl | /lib/ld-musl-i386.so.1 | libclang_rt.builtins-i386.a |
