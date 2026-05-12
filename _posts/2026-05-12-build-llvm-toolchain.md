---
title: 构建纯正的 LLVM 工具链 （1）x86_64 原生工具链
date: 2026-05-12 12:50:00 +0800
categories: [ LLVM ]
---

## 构建纯正的 LLVM 工具链


```shell
# 安装 ArchLinux 提供的 LLVM 套装
sudo pacman -S clang llvm lld

cat > /usr/bin/x86_64-pc-linux-gnu.cfg << EOF
-fuse-ld=lld
EOF

# 参考 https://github.com/xiaoyue2532/CMLFS/tree/main/2-Stage1
# 本工具链的很多思路都来自 CMLFS 和我对其进一步完善的 Fork
# LLVM_TARGETS_TO_BUILD 选中所需的目标 省得构建不需要的目标而构建速度太久
# LLVM_INSTALL_TOOLCHAIN_ONLY 减小安装大小 不用安装LLVM开发的库
cmake \
    -G Ninja \
    -DCMAKE_INSTALL_PREFIX=/llvmtools \
    -DCMAKE_BUILD_TYPE=Release  \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_LAUNCHER=sccache \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_LAUNCHER=sccache \
    -DCMAKE_AR="/usr/bin/llvm-ar" \
    -DCMAKE_NM="/usr/bin/llvm-nm" \
    -DCMAKE_RANLIB="/usr/bin/llvm-ranlib" \
    -DCLANG_DEFAULT_CXX_STDLIB=libc++ \
    -DCLANG_DEFAULT_LINKER=lld \
    -DCLANG_DEFAULT_OBJCOPY=llvm-objcopy \
    -DCLANG_DEFAULT_RTLIB=compiler-rt \
    -DCLANG_DEFAULT_UNWINDLIB=libunwind \
    -DDEFAULT_SYSROOT="/llvmtools" \
    -DENABLE_LINKER_BUILD_ID=ON \
    -DLLVM_BUILD_LLVM_DYLIB=ON \
    -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-musl" \
    -DLLVM_ENABLE_PROJECTS="clang;lld" \
    -DLLVM_ENABLE_RTTI=ON \
    -DLLVM_HOST_TRIPLE="x86_64-pc-linux-gnu" \
    -DLLVM_INCLUDE_BENCHMARKS=OFF \
    -DLLVM_INSTALL_TOOLCHAIN_ONLY=ON \
    -DLLVM_LINK_LLVM_DYLIB=ON \
    -DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64;RISCV" \
    ../llvm

# 编译并安装到 /llvmtools
ninja && ninja install/strip
```

## 安装内核头文件

```shell
# make headers_install 需要 rsync
make ARCH=x86_64 \
    INSTALL_HDR_PATH=/llvmtools \
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
    --host=x86_64-pc-linux-musl

make DESTDIR=/llvmtools/x86_64-pc-linux-musl install-headers
```

## 编译 compiler-rt

```shell
# 仅编译 CRT 和 Builtins
cmake \
    -G Ninja \
    -DCMAKE_INSTALL_PREFIX=/llvmtools/lib/clang/20/ \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_ASM_COMPILER=clang \
    -DCMAKE_ASM_COMPILER_TARGET=x86_64-pc-linux-musl \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_TARGET=x86_64-pc-linux-musl \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_TARGET=x86_64-pc-linux-musl \
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

```shell
CC=clang AR=llvm-ar RANLIB=llvm-ranlib \
LIBCC=/llvmtools/lib/clang/20/lib/linux/libclang_rt.builtins-x86_64.a \
./configure \
    --prefix=/ \
    --build=x86_64-pc-linux-gnu \
    --host=x86_64-pc-linux-musl \
    --disable-wrapper

make -j`nproc --all`

make DESTDIR=/llvmtools/x86_64-pc-linux-musl install
```

修复loader symlink，并创建对 musl 加载器的链接。

```shell
ln -sf libc.so /llvmtools/lib/ld-musl-x86_64.so.1
ln -sf /llvmtools/lib/libc.so /lib/ld-musl-x86_64.so.1
```

## 编译 llvm-runtimes

一次性编译 libc++, libc++abi, compiler-rt 和 llvm-libunwind。

```shell
cmake \
    -G Ninja \
    -DCMAKE_INSTALL_PREFIX=/llvmtools \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_ASM_COMPILER=clang \
    -DCMAKE_ASM_COMPILER_TARGET=x86_64-pc-linux-musl \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_TARGET=x86_64-pc-linux-musl \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_TARGET=x86_64-pc-linux-musl \
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

此时使用 clang++ 会找不到 C++ 头文件，需要补上搜索目录

```shell
cat > /llvmtools/bin/x86_64-pc-linux-musl-clang++.cfg << EOF
-nostdinc++
-I/llvmtools/include/c++/v1
-I/llvmtools/include
EOF
```

加上以上搜索路径后，有几处弊端：

- 用工具链构建 musl 时，强制指定的搜索目录会与 musl 源码的搜索目录冲突
- 用工具链构建 llvm-runtimes 时，强制指定的搜索目录会与 llvm-runtimes 源码的搜索目录冲突

因此在构建上述项目时，需要将这3行搜索路径用`#`注释掉。

## ncurses
