---
title: 构建纯正的 LLVM 工具链 (1) x86_64 原生工具链
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
    -DENABLE_LINKER_BUILD_ID=ON \
    -DLLVM_BUILD_LLVM_DYLIB=ON \
    -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-pc-linux-gnu" \
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

作为HOSTCC无需安装compiler-rt、libc++、libunwind等，于是写入配置文件/llvmtools/bin/x86_64-pc-linux-gnu.cfg
```shell
--rtlib=libgcc
--unwindlib=libgcc
--stdlib=libstdc++
```