---
title: LLVM 交叉编译链（新版）
date: 2026-03-24 16:16:00 +0800
categories: [ 嵌入式, RK3506 ]
---

LLVM 是天生的交叉编译器

```shell
ln -sf clang-21 x86_64-unknown-linux-musl-as
ln -sf clang-21 x86_64-unknown-linux-musl-cc
ln -sf clang-21 x86_64-unknown-linux-musl-c++
ln -sf clang-21 x86_64-unknown-linux-musl-clang
ln -sf clang-21 x86_64-unknown-linux-musl-clang++
ln -sf clang-21 x86_64-unknown-linux-musl-cpp

ln -sf lld x86_64-unknown-linux-musl-ld
ln -sf lld x86_64-unknown-linux-musl-ld.lld

ln -s llvm-dwp x86_64-unknown-linux-musl-dwp
ln -s llvm-strings x86_64-unknown-linux-musl-strings
ln -sf llvm-ar x86_64-unknown-linux-musl-ar
ln -sf llvm-ar x86_64-unknown-linux-musl-ranlib
ln -sf llvm-cxxfilt x86_64-unknown-linux-musl-c++filt
ln -sf llvm-nm x86_64-unknown-linux-musl-nm
ln -sf llvm-objcopy x86_64-unknown-linux-musl-objcopy
ln -sf llvm-objcopy x86_64-unknown-linux-musl-strip
ln -sf llvm-objdump x86_64-unknown-linux-musl-objdump
ln -sf llvm-readobj x86_64-unknown-linux-musl-readelf
ln -sf llvm-size x86_64-unknown-linux-musl-size
ln -sf llvm-symbolizer x86_64-unknown-linux-musl-addr2line

touch x86_64-unknown-linux-musl.cfg
```

## linux-api-headers

```shell
make ARCH=x86_64 \
    mrproper

make ARCH=x86_64 \
    INSTALL_HDR_PATH=/llvmtools/x86_64-unknown-linux-musl \
    headers_install
```

## musl-headers

```shell
./configure \
    --prefix=/ \
    --build=x86_64-pc-linux-gnu \
    --host=x86_64-unknown-linux-musl

make DESTDIR=/llvmtools/x86_64-unknown-linux-musl install-headers
```

## compiler-rt

```shell
cmake \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/llvmtools/lib/clang/21/ \
    -DCMAKE_ASM_COMPILER=clang \
    -DCMAKE_ASM_COMPILER_TARGET=x86_64-unknown-linux-musl \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_TARGET=x86_64-unknown-linux-musl \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_TARGET=x86_64-unknown-linux-musl \
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

ninja

ninja install/strip
```

## musl

```shell
make distclean

# arm32 的 musl 不能应用 rpmalloc, memset 家族函数, 和 fortify 的修补
# 似乎对原版 musl 的不当修改都会导致真机平台报错非法指令
# 如果遇到该错误 尝试回滚到原版 musl
CC=x86_64-unknown-linux-musl-clang \
LIBCC=/llvmtools/lib/clang/21/lib/linux/libclang_rt.builtins-x86_64.a \
./configure \
    --prefix=/ \
    --build=x86_64-pc-linux-gnu \
    --host=x86_64-unknown-linux-musl \
    --disable-wrapper

make -j16

make DESTDIR=/llvmtools/x86_64-unknown-linux-musl install
```

## llvm-runtimes

```shell
cmake \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/llvmtools/x86_64-unknown-linux-musl \
    -DCMAKE_ASM_COMPILER=clang \
    -DCMAKE_ASM_COMPILER_TARGET=x86_64-unknown-linux-musl \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_TARGET=x86_64-unknown-linux-musl \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_TARGET=x86_64-unknown-linux-musl \
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

ninja

DESTDIR=$(pwd)/_install ninja install/strip
# 手动调整目录结构 compiler-rt 需要从 $(pwd)/_install/llvmtools/x86_64-unknown-linux-musl/lib/linux 处移出
# compiler-rt 应当安装到 /llvmtools/lib/clang/21/lib/linux 处
# 剩余的 llvm-runtimes 安装到 /llvmtools/x86_64-unknown-linux-musl 处
```