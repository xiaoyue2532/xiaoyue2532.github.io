---
title: llvm-linux-cross
date: 2026-02-19 00:35:00 +0800
categories: [ 嵌入式, RK3506 ]
---

[Cross-compilation using Clang](https://clang.llvm.org/docs/CrossCompilation.html) says: 
> The triple has the general format `<arch><sub>-<vendor>-<sys>-<env>`, where:
> arch = x86_64, i386, arm, thumb, mips, etc.
> - sub = for ex. on ARM: v5, v6m, v7a, v7m, etc.
> - vendor = pc, apple, nvidia, ibm, etc.
> - sys = none, linux, win32, darwin, cuda, etc.
> - env = eabi, gnu, android, macho, elf, etc.

refer to [Rust Platform Support](https://doc.rust-lang.org/nightly/rustc/platform-support.html), 
I choose `unknown` as the vendor name, so we have these triples: 

| triple name                  | dynamic linker            | compiler-rt name               |
| ---------------------------- | ------------------------- | ------------------------------ |
| arm-unknown-linux-musleabi   | /lib/ld-musl-arm.so.1     | libclang_rt.builtins-arm.a     |
| arm-unknown-linux-musleabihf | /lib/ld-musl-armhf.so.1   | libclang_rt.builtins-armhf.a   |
| aarch64-unknown-linux-musl   | /lib/ld-musl-aarch64.so.1 | libclang_rt.builtins-aarch64.a |
| x86_64-unknown-linux-musl    | /lib/ld-musl-x86_64.so.1  | libclang_rt.builtins-x86_64.a  |
| i386-unknown-linux-musl      | /lib/ld-musl-i386.so.1    | libclang_rt.builtins-i386.a    |

- If you use the target `armv8-unknown-linux-musl`, the compiler-rt file name is `libclang_rt.builtins-armv8.a.` In fact, there's no need to do that, use -march instead.
- `arm-unknown-linux-musleabi` use `-mfloat-abi=soft`, but `arm-unknown-linux-musleabihf` use `-mfloat-abi=hard`.
- If you use the target `i686-unknown-linux-musl`, the compiler-rt file name is `libclang_rt.builtins-i386.a.`, musl libc name is also `/lib/ld-musl-i386.so.1`. Use `-march=i686 -mtune=i686` instead.

in the following, I use `arm-unknown-linux-musleabihf` to take an example.

## build a pure llvm toolchain

```shell
apt install clang-21 llvm-21 libclang-rt-21-dev libc++-21-dev libc++abi-21-dev libunwind-21-dev

cat > /usr/lib/llvm/21/bin/x86_64-unknown-linux-gnu.cfg << EOF
-fuse-ld=lld
-rtlib=compiler-rt
-stdlib=libc++
-unwindlib=libunwind
EOF

# no need to build llvm-runtimes
cmake \
    -G Ninja
    -DCMAKE_BUILD_TYPE=Release 
    -DCMAKE_INSTALL_PREFIX=/llvmtools
    -DCMAKE_C_COMPILER=clang
    -DCMAKE_CXX_COMPILER=clang++
    -DCMAKE_AR=`which llvm-ar`
    -DCMAKE_RANLIB=`which llvm-ranlib`
    -DCLANG_RTLIB=compiler-rt
    -DCLANG_LINKER=lld
    -DCLANG_STDLIB=libc++
    -DCLANG_UNWINDLIB=libunwind
    -DLLVM_ENABLE_PROJECTS="clang;lld"
    ../llvm
```

## create cross toolchain files and links

```shell
# use clang intergrated assembler, not llvm-as
ln -s clang-21 arm-unknown-linux-musleabihf-as

ln -s clang-21 arm-unknown-linux-musleabihf-cc
ln -s clang-21 arm-unknown-linux-musleabihf-c++

ln -s clang-21 arm-unknown-linux-musleabihf-clang
ln -s clang-21 arm-unknown-linux-musleabihf-clang++

# C preprocessor
ln -s clang-21 arm-unknown-linux-musleabihf-cpp

# Linker
ln -s ld.lld arm-unknown-linux-musleabihf-ld
ln -s ld.lld arm-unknown-linux-musleabihf-ld.lld

# LLVM Binutils
ln -sf llvm-ar arm-unknown-linux-musleabihf-ar
ln -sf llvm-nm arm-unknown-linux-musleabihf-nm
ln -sf llvm-objdump arm-unknown-linux-musleabihf-objdump
ln -sf llvm-objcopy arm-unknown-linux-musleabihf-objcopy
ln -sf llvm-readelf arm-unknown-linux-musleabihf-readelf
ln -sf llvm-ranlib arm-unknown-linux-musleabihf-ranlib
ln -sf llvm-strip arm-unknown-linux-musleabihf-size
ln -sf llvm-strip arm-unknown-linux-musleabihf-strip

# Configuration file
# if you want to build for armv9-a, you still need to use target aarch64-linux-musl
# but you can assign march value armv9-a
# for my x86_64 platform, I prefer -march=znver3
cat > /llvmtools/bin/arm-unknown-linux-musleabihf.cfg << EOF
-march=armv7-a -mtune=cortex-a55
--sysroot=/llvmtools/arm-unknown-linux-musleabihf
-nostdinc++
-I/llvmtools/arm-unknown-linux-musleabihf/include/c++/v1
-I/llvmtools/arm-unknown-linux-musleabihf/include
EOF

# Sysroot Directory
mkdir /llvmtools/arm-unknown-linux-musleabihf
```

## kernel-headers

```shell
# requires rsync
make \
    INSTALL_HDR_PATH=/llvmtools/arm-unknown-linux-musleabihf \
    ARCH=arm \
    headers_install
```

## musl-headers

```shell
./configure --prefix=/
make \
    DESTDIR=/llvmtools/arm-unknown-linux-musleabihf \
    install-headers
```

## compiler-rt

```shell
# if you want compiler-rt files installed to 
#     /llvmtools/lib/clang/21/lib/arm-unknown-linux-musleabihf/libclang_rt.builtins.a
# turn LLVM_ENABLE_PER_TARGET_RUNTIME_DIR to ON
# or they will be installed to 
#     /llvmtools/lib/clang/21/lib/linux/libclang_rt.builtins-armhf.a
cmake \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/llvmtools/lib/clang/21/ \
    -DCMAKE_ASM_COMPILER=clang \
    -DCMAKE_ASM_COMPILER_TARGET=arm-unknown-linux-musleabihf \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_TARGET=arm-unknown-linux-musleabihf \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_TARGET=arm-unknown-linux-musleabihf \
    -DCMAKE_CXX_COMPILER_WORKS=1 \
    -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON \
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
```

## musl

```shell
# @12101111 provides two musl patches
# - provide optimized memory functions (like memcpy) for x86_64 and arm64
# - provide a rapid malloc functions called rpmalloc()
CC=arm-unknown-linux-musleabihf-clang \
LIBCC=/llvmtools/lib/clang/21/lib/arm-unknown-linux-musleabihf/libclang_rt.builtins.a \
./configure \
    --prefix=/ \
    --build=x86_64-pc-linux-gnu \
    --host=arm-unknown-linux-musleabihf \
    --with-malloc=rpmalloc \
    --disable-wrapper

make DESTDIR=/llvmtools/arm-unknown-linux-musleabihf install
```

you need to install `qemu-user-static` to run arm programs on x86_64

```shell
# create a symbol link to the musl dynamic linker
ln -s /llvmtools/arm-unknown-linux-musleabihf/lib/libc.so /lib/ld-musl-armhf.so.1
# fix a symbol link
ln -sf libc.so /llvmtools/arm-unknown-linux-musleabihf/lib/ld-musl-armhf.so.1
```

## llvm-runtimes

```shell
cmake \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/llvmtools/arm-unknown-linux-musleabihf \
    -DCMAKE_ASM_COMPILER=clang \
    -DCMAKE_ASM_COMPILER_TARGET=arm-unknown-linux-musleabihf \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_C_COMPILER_TARGET=arm-unknown-linux-musleabihf \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_CXX_COMPILER_TARGET=arm-unknown-linux-musleabihf \
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
    -DLIBCXX_USE_COMPILER_RT=ON \
    -DLIBCXX_HAS_MUSL_LIBC=ON \
    -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind;compiler-rt" \
    ../runtimes
```