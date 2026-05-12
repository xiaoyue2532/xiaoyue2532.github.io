---
title: 构建纯正的 LLVM 工具链 (2) arm64 交叉工具链
date: 2026-05-12 12:55:00 +0800
categories: [ LLVM ]
---

LLVM 是一套天生的交叉编译器。参考[LLVM cross-compiled Linux From Scratch: C & C++ libraries](https://12101111.github.io/llvm-cross-compile/)，本文以 aarch64-unknown-linux-musl 为例，搭建了一套 LLVM 交叉编译工具链，即：

- 工具链运行平台: x86_64-pc-linux-gnu
- 工具链产出目标: aarch64-unknown-linux-musl

## 选择合适的目标三元组

根据[Cross-compilation using Clang](https://clang.llvm.org/docs/CrossCompilation.html)所说：

> 三元组有通用的格式 `<arch><sub>-<vendor>-<sys>-<env>` 其中:
> arch = x86_64, i386, arm, thumb, mips 等
> - sub = for ex. on ARM: v5, v6m, v7a, v7m 等
> - vendor = pc, apple, nvidia, ibm 等
> - sys = none, linux, win32, darwin, cuda 等
> - env = eabi, gnu, android, macho, elf 等

参照[Rust Platform Support](https://doc.rust-lang.org/nightly/rustc/platform-support.html), 
本文选择`unknown`作为厂商名，对于不同的平台三元组，有：

|          三元组名称          |      musl 动态连接器      |        compiler-rt 名称        |
| :--------------------------: | :-----------------------: | :----------------------------: |
|  arm-unknown-linux-musleabi  |   /lib/ld-musl-arm.so.1   |   libclang_rt.builtins-arm.a   |
| arm-unknown-linux-musleabihf |  /lib/ld-musl-armhf.so.1  |  libclang_rt.builtins-armhf.a  |
|  aarch64-unknown-linux-musl  | /lib/ld-musl-aarch64.so.1 | libclang_rt.builtins-aarch64.a |
|  x86_64-unknown-linux-musl   | /lib/ld-musl-x86_64.so.1  | libclang_rt.builtins-x86_64.a  |
|   i386-unknown-linux-musl    |  /lib/ld-musl-i386.so.1   |  libclang_rt.builtins-i386.a   |
|  riscv64-unknown-linux-musl  | /lib/ld-musl-riscv64.so.1 | libclang_rt.builtins-riscv64.a |
|  riscv32-unknown-linux-musl  | /lib/ld-musl-riscv32.so.1 | libclang_rt.builtins-riscv32.a |

若选择三元组`armv8-unknown-linux-musl`，compiler-rt的文件名就是`libclang_rt.builtins-armv8.a.`。事实上，很多 configure 脚本不认识 `armv8-unknown-linux-musl`,`armv7-unknown-linux-musl`，只认可元组`arm-unknown-linux-musleabi`、`aarch64-unknown-linux-musl`。若想指定架构，使用参数`-march=armv7-a`、`-march=armv8-a`。

若选择三元组`i686-unknown-linux-musl`，compiler-rt的文件名却仍然是`libclang_rt.builtins-i386.a.`, musl libc加载器的名称也仍然是`/lib/ld-musl-i386.so.1`。类似的用`-march=i686 -mtune=i686`来指定子架构。

`arm-unknown-linux-musleabi` 默认采用软浮点，在clang cc1 参数中有 `-mfloat-abi=soft`。而`arm-unknown-linux-musleabihf`采用硬浮点，在clang cc1参数中有`-mfloat-abi=hard`。

## 建立工具链文件布局

```shell
# 将汇编器 as 链接到 clang 而非 llvm-as
# 前者是调用 clang 的集成汇编器 后者是 LLVM Bitcode 的编译器
ln -sf clang-20 /llvmtools/bin/aarch64-unknown-linux-musl-as

# 编译器
ln -sf clang-20 /llvmtools/bin/aarch64-unknown-linux-musl-cc
ln -sf clang-20 /llvmtools/bin/aarch64-unknown-linux-musl-c++
ln -sf clang-20 /llvmtools/bin/aarch64-unknown-linux-musl-clang
ln -sf clang-20 /llvmtools/bin/aarch64-unknown-linux-musl-clang++

# 预处理器
ln -sf clang-20 /llvmtools/bin/aarch64-unknown-linux-musl-cpp

# 链接器
ln -sf lld /llvmtools/bin/aarch64-unknown-linux-musl-ld
ln -sf lld /llvmtools/bin/aarch64-unknown-linux-musl-ld.lld

# Binutils 兼容链接
ln -sf llvm-symbolizer /llvmtools/bin/aarch64-unknown-linux-musl-addr2line
ln -sf llvm-ar /llvmtools/bin/aarch64-unknown-linux-musl-ar
ln -sf llvm-nm /llvmtools/bin/aarch64-unknown-linux-musl-nm
ln -sf llvm-ar /llvmtools/bin/aarch64-unknown-linux-musl-ranlib
ln -sf llvm-objcopy /llvmtools/bin/aarch64-unknown-linux-musl-objcopy
ln -sf llvm-objdump /llvmtools/bin/aarch64-unknown-linux-musl-objdump
ln -sf llvm-readobj /llvmtools/bin/aarch64-unknown-linux-musl-readelf
ln -sf llvm-size /llvmtools/bin/aarch64-unknown-linux-musl-size
ln -s llvm-strings /llvmtools/bin/aarch64-unknown-linux-musl-strings
ln -sf llvm-objcopy /llvmtools/bin/aarch64-unknown-linux-musl-strip

# 创建 Sysroot 目录
mkdir /llvmtools/aarch64-unknown-linux-musl
```

## 创建配置文件

```shell
cat > /llvmtools/bin/aarch64-unknown-linux-musl-clang.cfg << EOF
--sysroot=/llvmtools/aarch64-unknown-linux-musl
EOF
cat > /llvmtools/bin/aarch64-unknown-linux-musl-clang++.cfg << EOF
--sysroot=/llvmtools/aarch64-unknown-linux-musl
EOF
```

## 安装内核头文件

```shell
# make headers_install 需要 rsync
make ARCH=arm64 \
    INSTALL_HDR_PATH=/llvmtools/aarch64-unknown-linux-musl \
    headers_install
```

## 安装 musl 头文件

```shell
# 需要指定正确的 host 否则默认会安装 build 架构的头文件
# 即本应安装 arm 头文件却误安装成 x86_64 的头文件
# 导致 musl libc 编译报错 float.h 中的类型与编译器中的类型不匹配
./configure \
    --prefix=/ \
    --build=x86_64-pc-linux-gnu \
    --host=aarch64-unknown-linux-musl

make DESTDIR=/llvmtools/aarch64-unknown-linux-musl install-headers
```

## 编译 compiler-rt

```shell
# 仅编译 CRT 和 Builtins
cmake \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/llvmtools/lib/clang/20/ \
    -DCMAKE_ASM_COMPILER=clang \
    -DCMAKE_ASM_COMPILER_TARGET=aarch64-unknown-linux-musl \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_TARGET=aarch64-unknown-linux-musl \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_TARGET=aarch64-unknown-linux-musl \
    -DCMAKE_CXX_COMPILER_WORKS=1 \
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
    -DCOMPILER_RT_BUILD_CRT=ON \
    -DCOMPILER_RT_BUILD_CTX_PROFILE=OFF \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
    -DCOMPILER_RT_BUILD_MEMPROF=OFF \
    -DCOMPILER_RT_BUILD_ORC=OFF \
    -DCOMPILER_RT_BUILD_PROFILE=OFF \
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF \
    -DCOMPILER_RT_BUILD_XRAY=OFF \
    ../compiler-rt

ninja && ninja install/strip
```

## 编译 musl libc

对于 x86_64 和 arm64 目标，可以考虑采纳来自 [@12101111](https://github.com/12101111) 的 patch，他做了以下工作：

- [malloc 替换为 rpmalloc](https://github.com/12101111/overlay/raw/refs/heads/master/sys-libs/musl/files/Add-rpmalloc-for-musl.patch)
- [mem 系列函数替换为更高效的汇编实现](https://github.com/12101111/overlay/raw/refs/heads/master/sys-libs/musl/files/use-optimized-memcpy-memset.patch)

> 我已经测试在 arm 平台的 musl 不能应用以上 patch
> 
> ```shell
> # 编译一个简单的 Helloworld 程序
> # 静态链接到采用以上修补的 musl libc
> $ arm-unknown-linux-musleabihf-clang -static helloworld.c
> 
> $ file a.out
> a.out: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, BuildID[sha1]=39a96d756ecac010f0208bc629a72a594006933c, not stripped
> 
> # 可以通过 qemu-user-arm 在 x86_64 机上转译运行
> $ ./a.out
> Helloworld
> 
> # 但是在 arm 和 arm64 真机上运行均报错
> $ ./a.out
> Illegal Instruction
> 
> # 重新编译 Helloworld 此次链接到原版 musl
> # 在 arm 和 arm64 真机上均可正常运行
> $ ./a.out
> Helloworld
> ```
> 对于没法替换 malloc 的情形 我建议采用 mimalloc 编译产出 libmimalloc.so 后设置 LD_PRELOAD 变量
> 这样可以实现不修改现有源码和二进制的条件下动态替换 malloc 具体不属于本文的范畴 本文仅考虑原版 musl

```shell
CC=aarch64-unknown-linux-musl-clang \
LIBCC=/llvmtools/lib/clang/20/lib/linux/libclang_rt.builtins-aarch64.a \
./configure \
    --prefix=/ \
    --build=x86_64-pc-linux-gnu \
    --host=aarch64-unknown-linux-musl \
    --disable-wrapper
# 若采纳了 rpmalloc 修补 记得用 --with-malloc=rpmalloc 来启用它

make -j`nproc --all`

make DESTDIR=/llvmtools/aarch64-unknown-linux-musl install
```

修复loader symlink，并创建对 musl 加载器的链接。可以考虑安装个 qemu-user-static，这样就可以便利地转译执行异构程序。

```shell
ln -sf libc.so /llvmtools/lib/ld-musl-aarch64.so.1
ln -sf /llvmtools/aarch64-unknown-linux-musl/lib/libc.so /lib/ld-musl-aarch64.so.1
```

## 编译 llvm-runtimes

一次性编译 libc++, libc++abi, compiler-rt 和 llvm-libunwind。

```shell
cmake \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/llvmtools/aarch64-unknown-linux-musl \
    -DCMAKE_ASM_COMPILER=clang \
    -DCMAKE_ASM_COMPILER_TARGET=aarch64-unknown-linux-musl \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_TARGET=aarch64-unknown-linux-musl \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_TARGET=aarch64-unknown-linux-musl \
    -DCMAKE_CXX_COMPILER_WORKS=1 \
    -DCOMPILER_RT_BUILD_CRT=ON \
    -DCOMPILER_RT_BUILD_CTX_PROFILE=OFF \
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
    -DCOMPILER_RT_BUILD_MEMPROF=OFF \
    -DCOMPILER_RT_BUILD_ORC=OFF \
    -DCOMPILER_RT_BUILD_PROFILE=OFF \
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF \
    -DCOMPILER_RT_BUILD_XRAY=OFF \
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
    -DCOMPILER_RT_INSTALL_PATH=/llvmtools/lib/clang/20 \
    -DCXX_SUPPORTS_FNO_EXCEPTIONS_FLAG=1 \
    -DLIBCXXABI_ENABLE_STATIC_UNWINDER=ON \
    -DLIBCXXABI_USE_COMPILER_RT=ON \
    -DLIBCXXABI_USE_LLVM_UNWINDER=ON \
    -DLIBCXX_CXX_ABI=libcxxabi \
    -DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=ON \
    -DLIBCXX_USE_COMPILER_RT=ON \
    -DLIBCXX_HAS_MUSL_LIBC=ON \
    -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind;compiler-rt" \
    ../runtimes

ninja && ninja install/strip
```

## 完备配置文件

此时使用 aarch64-unknown-linux-musl-clang++ 会找不到 C++ 头文件，需要补上搜索目录

```shell
cat >> /llvmtools/bin/aarch64-unknown-linux-musl-clang++.cfg << EOF
-nostdinc++
-I/llvmtools/aarch64-unknown-linux-musl/include/c++/v1
-I/llvmtools/aarch64-unknown-linux-musl/include
EOF
```

加上以上搜索路径后，有几处弊端：

- 用工具链构建 musl 时，强制指定的搜索目录会与 musl 源码的搜索目录冲突
- 用工具链构建 llvm-runtimes 时，强制指定的搜索目录会与 llvm-runtimes 源码的搜索目录冲突

因此在构建上述项目时，需要将这3行搜索路径用`#`注释掉。
