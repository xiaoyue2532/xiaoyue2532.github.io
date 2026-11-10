---
title: 构建 LLVM 交叉工具链-2 实践
date: 2026-11-11 01:40:00 +0800
categories: [ LLVM ]
---

本文以 `arm-unknown-linux-musleabihf` 为例，搭建了一套 LLVM 交叉编译工具链，即：

- LLVM version: 20.1.8
- 工具链运行平台: x86_64-pc-linux-gnu
- 工具链产出目标: arm-unknown-linux-musleabihf

## 构建纯正的 LLVM 工具链

```shell
# 安装发行版提供的 LLVM 套装
sudo pacman -S clang llvm lld

# 参考 https://github.com/xiaoyue2532/CMLFS/tree/main/2-Stage1
# 本工具链的很多思路都来自 CMLFS 和我对其进一步完善的 Fork
local cmake_args=(
    -DCMAKE_BUILD_TYPE=Release 
    -DCMAKE_INSTALL_PREFIX=/llvmtools
    -DCMAKE_C_COMPILER=clang
    -DCMAKE_CXX_COMPILER=clang++
    # Specify llvm binutils with absolute path
    -DCMAKE_AR="/usr/bin/llvm-ar"
    -DCMAKE_NM="/usr/bin/llvm-nm"
    -DCMAKE_RANLIB="/usr/bin/llvm-ranlib"
    # Clang specific configurations
    -DCLANG_DEFAULT_CXX_STDLIB=libc++
    -DCLANG_DEFAULT_LINKER=lld
    -DCLANG_DEFAULT_OBJCOPY=llvm-objcopy
    -DCLANG_DEFAULT_RTLIB=compiler-rt
    -DCLANG_DEFAULT_UNWINDLIB=libunwind
    -DENABLE_LINKER_BUILD_ID=ON
    # llvm specific configurations
    -DLLVM_ENABLE_PROJECTS="clang;lld"
    -DLLVM_ENABLE_RTTI=ON
    -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-gnu"
    -DLLVM_HOST_TRIPLE="x86_64-pc-linux-gnu"
    # Reduce compile workload
    -DLLVM_INCLUDE_BENCHMARKS=OFF
    -DLLVM_INCLUDE_EXAMPLES=OFF
    -DLLVM_INCLUDE_TESTS=OFF
    # Build and link to libLLVM.so
    -DLLVM_BUILD_LLVM_DYLIB=ON
    -DLLVM_LINK_LLVM_DYLIB=ON
    # 减小安装大小 不用安装LLVM开发的库
    -DLLVM_INSTALL_TOOLCHAIN_ONLY=ON
    # Use in-place lld to build LLVM
    -DLLVM_ENABLE_LLD=ON
    # Disable unnecessary dependencies
    -DLLVM_ENABLE_LIBEDIT=OFF
    -DLLVM_ENABLE_LIBXML2=OFF
    ## 选中所需的目标 省得构建不需要的目标而构建速度太久
    # -DLLVM_TARGETS_TO_BUILD="AArch64;ARM;X86;BPF;WebAssembly;RISCV;NVPTX"
)
cmake \
    -G Ninja -S ../llvm \
    "${cmake_args[@]}"

# 编译并安装到 /llvmtools
ninja && ninja install/strip
```

## 建立工具链文件布局

```shell
# 将汇编器 as 链接到 clang 而非 llvm-as
# 前者是调用 clang 的集成汇编器 后者是 LLVM Bitcode 的编译器
ln -sf clang-20 /llvmtools/bin/arm-unknown-linux-musleabihf-as

# 编译器
ln -sf clang-20 /llvmtools/bin/arm-unknown-linux-musleabihf-cc
ln -sf clang-20 /llvmtools/bin/arm-unknown-linux-musleabihf-c++
ln -sf clang-20 /llvmtools/bin/arm-unknown-linux-musleabihf-clang
ln -sf clang-20 /llvmtools/bin/arm-unknown-linux-musleabihf-clang++

# 预处理器
ln -sf clang-20 /llvmtools/bin/arm-unknown-linux-musleabihf-cpp

# 链接器
ln -sf lld /llvmtools/bin/arm-unknown-linux-musleabihf-ld
ln -sf lld /llvmtools/bin/arm-unknown-linux-musleabihf-ld.lld

# Binutils 兼容链接
ln -sf llvm-symbolizer /llvmtools/bin/arm-unknown-linux-musleabihf-addr2line
ln -sf llvm-ar /llvmtools/bin/arm-unknown-linux-musleabihf-ar
ln -sf llvm-nm /llvmtools/bin/arm-unknown-linux-musleabihf-nm
ln -sf llvm-ar /llvmtools/bin/arm-unknown-linux-musleabihf-ranlib
ln -sf llvm-objcopy /llvmtools/bin/arm-unknown-linux-musleabihf-objcopy
ln -sf llvm-objdump /llvmtools/bin/arm-unknown-linux-musleabihf-objdump
ln -sf llvm-readobj /llvmtools/bin/arm-unknown-linux-musleabihf-readelf
ln -sf llvm-size /llvmtools/bin/arm-unknown-linux-musleabihf-size
ln -sf llvm-strings /llvmtools/bin/arm-unknown-linux-musleabihf-strings
ln -sf llvm-objcopy /llvmtools/bin/arm-unknown-linux-musleabihf-strip

# 若想建立 mingw 的工具链 还可以考虑创建对 llvm-windres llvm-dlltool 等工具的链接 此处不赘述

# 创建 Sysroot 目录
mkdir /llvmtools/arm-unknown-linux-musleabihf
```

## 创建配置文件

若 `-march` 未指定，默认目标是古早的 `armv6kz` 架构。
我的 ARM 目标平台是 RK3506，这是一个 Cortex-A7 核心的 armv7 架构处理器，可以选择 `-mtune=cortex-a7`。
若不确定 `-mtune` 取值，可以不指定或显式指定 `-mtune=generic`。

```shell
cat > /llvmtools/bin/arm-unknown-linux-musleabihf.cfg << EOF
-march=armv7-a -mtune=generic
--sysroot=<CFGDIR>/../arm-unknown-linux-musleabihf
EOF
```

## 安装内核头文件

```shell
# make headers_install 需要 rsync
make \
    ARCH=arm \
    INSTALL_HDR_PATH=/llvmtools/arm-unknown-linux-musleabihf \
    headers_install
```

## 安装 musl 头文件

```shell
# 需要指定正确的 host 否则默认会安装 build 架构的头文件
# 即本应安装 arm 头文件却误安装成 x86_64 的头文件
# 导致 musl libc 编译报错 float.h 中的类型与编译器中的类型不匹配
CC=arm-unknown-linux-musleabihf-clang \
./configure \
    --prefix=/ \
    --build=x86_64-pc-linux-gnu \
    --host=arm-unknown-linux-musleabihf

make DESTDIR=/llvmtools/arm-unknown-linux-musleabihf install-headers
```

## 编译 compiler-rt

```shell
local cmake_args=(
    -DCMAKE_BUILD_TYPE=Release
    -DCMAKE_INSTALL_PREFIX=/llvmtools/lib/clang/20/
    # Specify llvm binutils with absolute path
    -DCMAKE_AR="/llvmtools/bin/llvm-ar"
    -DCMAKE_NM="/llvmtools/bin/llvm-nm"
    -DCMAKE_RANLIB="/llvmtools/bin/llvm-ranlib"
    # Specify compilers
    -DCMAKE_ASM_COMPILER=clang
    -DCMAKE_C_COMPILER=clang
    -DCMAKE_CXX_COMPILER=clang++
    # Don't check whether the compilers work
    -DCMAKE_C_COMPILER_WORKS=1
    -DCMAKE_CXX_COMPILER_WORKS=1
    # Specify compiler targets so that COMPILER_RT_DEFAULT_TARGET_ONLY works
    # When COMPILER_RT_DEFAULT_TARGET_ONLY does not work,
    # compiler-rt would not build CRT or builtin library.
    -DCMAKE_ASM_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCMAKE_C_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCMAKE_CXX_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON
    # At the bootstrap stage we need to turn them off
    -DCOMPILER_RT_BUILD_CTX_PROFILE=OFF
    -DCOMPILER_RT_BUILD_GWP_ASAN=OFF
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF
    -DCOMPILER_RT_BUILD_MEMPROF=OFF
    -DCOMPILER_RT_BUILD_ORC=OFF
    -DCOMPILER_RT_BUILD_PROFILE=OFF
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF
    -DCOMPILER_RT_BUILD_XRAY=OFF
)

# 仅编译 Builtins
cmake \
    -G Ninja -S ../compiler-rt \
    "${cmake_args[@]}"

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
CC=arm-unknown-linux-musleabihf-clang \
LIBCC=/llvmtools/lib/clang/20/lib/linux/libclang_rt.builtins-armhf.a \
./configure \
    --prefix=/ \
    --build=x86_64-pc-linux-gnu \
    --host=arm-unknown-linux-musleabihf \
    --disable-wrapper
# 若采纳了 rpmalloc 修补 记得用 --with-malloc=rpmalloc 来启用它

make -j`nproc --all`

make DESTDIR=/llvmtools/arm-unknown-linux-musleabihf install
```

可以考虑安装个 qemu-user-static，并创建对 musl 加载器的链接，这样就可以便利地转译执行异构程序。

```shell
ln -s /llvmtools/arm-unknown-linux-musleabihf/ld-musl-armhf.so.1 /lib/ld-musl-armhf.so.1
```

## 编译 llvm-runtimes

一次性编译 libc++, libc++abi, compiler-rt 和 llvm-libunwind。

```shell
local cmake_args=(
    -DCMAKE_BUILD_TYPE=Release
    -DCMAKE_INSTALL_PREFIX=/llvmtools/arm-unknown-linux-musleabihf
    # Specify llvm binutils with absolute path
    -DCMAKE_AR="/llvmtools/bin/llvm-ar"
    -DCMAKE_NM="/llvmtools/bin/llvm-nm"
    -DCMAKE_RANLIB="/llvmtools/bin/llvm-ranlib"
    # Specify compilers
    -DCMAKE_ASM_COMPILER=clang
    -DCMAKE_C_COMPILER=clang
    -DCMAKE_CXX_COMPILER=clang++
    # Don't check whether the compilers work
    -DCMAKE_C_COMPILER_WORKS=1
    -DCMAKE_CXX_COMPILER_WORKS=1
    # Specify compiler targets so that COMPILER_RT_DEFAULT_TARGET_ONLY works
    # When COMPILER_RT_DEFAULT_TARGET_ONLY does not work,
    # compiler-rt would not build CRT or builtin library.
    -DCMAKE_ASM_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCMAKE_C_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCMAKE_CXX_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON
    # specify sysroot or clang would look for host's sysroot /usr
    -DCMAKE_SYSROOT=/llvmtools/arm-unknown-linux-musleabihf
    # musl hides `stat64` under `_LARGEFILE64_SOURCE`
    -DCMAKE_C_FLAGS="-D_LARGEFILE64_SOURCE"
    -DCMAKE_CXX_FLAGS="-D_LARGEFILE64_SOURCE"
    # At the bootstrap stage we need to turn them off
    -DCOMPILER_RT_BUILD_CTX_PROFILE=OFF
    -DCOMPILER_RT_BUILD_GWP_ASAN=OFF
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF
    -DCOMPILER_RT_BUILD_MEMPROF=OFF
    -DCOMPILER_RT_BUILD_ORC=OFF
    -DCOMPILER_RT_BUILD_PROFILE=OFF
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF
    -DCOMPILER_RT_BUILD_XRAY=OFF
    # Enable the two lines below so that compiler-rt miscs would build
    -DCOMPILER_RT_USE_BUILTINS_LIBRARY=ON
    -DCOMPILER_RT_USE_LLVM_UNWINDER=ON
    -DCOMPILER_RT_INSTALL_PATH="/llvmtools/lib/clang/20"
    # Bootstrap libunwind detects whether clang++ builds. A few libraries are missing, 
    # so the build always fail. According to `libunwind/src/CMakeLists.txt`:
    # message(FATAL_ERROR
    #         "Compiler doesn't support generation of unwind tables if exception "
    #         "support is disabled.  Building libunwind DSO with runtime dependency "
    #         "on C++ ABI library is not supported.")
    # cmake would report a error saying that the compiler does not support rtti even
    # though `LLVM_ENABLE_RTTI=ON`, so break the condition
    -DCXX_SUPPORTS_FNO_EXCEPTIONS_FLAG=1
    ## libc++abi library
    # link libc++abi.so to libunwind statically, or libc++abi.so depends on libunwind.so
    -DLIBCXXABI_ENABLE_STATIC_UNWINDER=ON
    -DLIBCXXABI_USE_COMPILER_RT=ON
    -DLIBCXXABI_USE_LLVM_UNWINDER=ON
    ## libc++ library
    -DLIBCXX_CXX_ABI=libcxxabi
    # link libc++.so to libc++abi statically, or libc++.so depends on libc++abi.so
    -DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=ON
    -DLIBCXX_USE_COMPILER_RT=ON
    # musl must enable this!
    -DLIBCXX_HAS_MUSL_LIBC=ON
    ## runtimes to build
    -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind;compiler-rt"
)

cmake \
    -G Ninja -S ../runtimes \
    "${cmake_args[@]}"

ninja && ninja install/strip
```

## 完备配置文件

~~此时使用 arm-unknown-linux-musleabihf-clang++ 会找不到 C++ 头文件，需要补上搜索目录~~

```shell
cat >> /llvmtools/bin/arm-unknown-linux-musleabihf.cfg << EOF
-nostdinc++
-I/llvmtools/arm-unknown-linux-musleabihf/include/c++/v1
-I/llvmtools/arm-unknown-linux-musleabihf/include
EOF
```

~~加上以上搜索路径后，有几处弊端：~~

~~- 用工具链构建 musl 时，强制指定的搜索目录会与 musl 源码的搜索目录冲突~~
~~- 用工具链构建 llvm-runtimes 时，强制指定的搜索目录会与 llvm-runtimes 源码的搜索目录冲突~~

~~因此在构建上述项目时，需要将这3行搜索路径用`#`注释掉。~~

UPDATE: 不必再 `-nostdinc++` 并手动指定 libc++ 搜索路径，只需要创建一个符号链接。

```shell
ln -s . /llvmtools/arm-unknown-linux-musleabihf/usr
```

## Rebuild llvm-runtimes

```shell
local cmake_args=(
    -DCMAKE_BUILD_TYPE=Release
    -DCMAKE_INSTALL_PREFIX=/llvmtools/arm-unknown-linux-musleabihf
    # Specify llvm binutils with absolute path
    -DCMAKE_AR="/llvmtools/bin/llvm-ar"
    -DCMAKE_NM="/llvmtools/bin/llvm-nm"
    -DCMAKE_RANLIB="/llvmtools/bin/llvm-ranlib"
    # Specify compilers
    -DCMAKE_ASM_COMPILER=clang
    -DCMAKE_C_COMPILER=clang
    -DCMAKE_CXX_COMPILER=clang++
    # Specify compiler targets so that COMPILER_RT_DEFAULT_TARGET_ONLY works
    # When COMPILER_RT_DEFAULT_TARGET_ONLY does not work,
    # compiler-rt would not build CRT or builtin library.
    -DCMAKE_ASM_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCMAKE_C_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCMAKE_CXX_COMPILER_TARGET=arm-unknown-linux-musleabihf
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON
    # specify sysroot or clang would look for host's sysroot /usr
    -DCMAKE_SYSROOT=/llvmtools/arm-unknown-linux-musleabihf
    # musl hides `stat64` under `_LARGEFILE64_SOURCE`
    -DCMAKE_C_FLAGS="-D_LARGEFILE64_SOURCE"
    -DCMAKE_CXX_FLAGS="-D_LARGEFILE64_SOURCE"
    # gwp_asan need a backtrace library, to be exactly libexecinfo
    -DCOMPILER_RT_BUILD_GWP_ASAN=OFF
    # Enable the two lines below so that compiler-rt miscs would build
    -DCOMPILER_RT_USE_BUILTINS_LIBRARY=ON
    -DCOMPILER_RT_USE_LLVM_UNWINDER=ON
    -DCOMPILER_RT_INSTALL_PATH="/llvmtools/lib/clang/20"
    ## libc++abi library
    # link libc++abi.so to libunwind statically, or libc++abi.so depends on libunwind.so
    -DLIBCXXABI_ENABLE_STATIC_UNWINDER=ON
    -DLIBCXXABI_USE_COMPILER_RT=ON
    -DLIBCXXABI_USE_LLVM_UNWINDER=ON
    ## libc++ library
    -DLIBCXX_CXX_ABI=libcxxabi
    # link libc++.so to libc++abi statically, or libc++.so depends on libc++abi.so
    -DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=ON
    -DLIBCXX_USE_COMPILER_RT=ON
    # musl must enable this!
    -DLIBCXX_HAS_MUSL_LIBC=ON
    ## runtimes to build
    -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind;compiler-rt"
)

cmake \
    -G Ninja -S ../runtimes \
    "${cmake_args[@]}"

ninja && ninja install/strip
```