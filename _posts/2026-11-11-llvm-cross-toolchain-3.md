---
title: 构建 LLVM 交叉工具链-3 裸金属
date: 2026-11-11 18:00:00 +0800
categories: [ LLVM ]
---

```shell
## Refer to [Better firmware with LLVM/Clang](https://interrupt.memfault.com/blog/arm-cortexm-with-llvm-clang)

## Sorry i messed up assembler:
cat > /llvmtools/bin/arm-unknown-none-eabi-as << EOF
#!/usr/bin/env bash
clang -x assembler-with-cpp --target=arm-unknown-none-eabi $@
EOF
chmod +x arm-unknown-none-eabi-as

## Set up the configuration file
cat > arm-unknown-none-eabi.cfg << EOF
-march=armv7-m -mtune=cortex-m3
--sysroot=<CFGDIR>/../arm-unknown-none-eabi
EOF

## mkdir sysroot
mkdir /llvmtools/arm-unknown-none-eabi

## Build compiler-rt
local cmake_args=(
    -DCMAKE_INSTALL_PREFIX="/llvmtools/lib/clang/22/"
    -DCMAKE_BUILD_TYPE=Release
    # Specify the ASM compiler
    -DCMAKE_ASM_COMPILER=arm-unknown-none-eabi-as
    # Specify the C compiler and don't check whether it works
    -DCMAKE_C_COMPILER=clang
    -DCMAKE_C_COMPILER_TARGET=arm-unknown-none-eabi
    -DCMAKE_C_COMPILER_WORKS=1
    # Specify the CXX compiler and don't check whether it works
    -DCMAKE_CXX_COMPILER=clang++
    -DCMAKE_CXX_COMPILER_TARGET=arm-unknown-none-eabi
    -DCMAKE_CXX_COMPILER_WORKS=1
    # Specify llvm binutils with absolute path
    -DCMAKE_AR="/llvmtools/bin/llvm-ar"
    -DCMAKE_NM="/llvmtools/bin/llvm-nm"
    -DCMAKE_RANLIB="/llvmtools/bin/llvm-ranlib"
    # Specify C and CXX compiler targets so that COMPILER_RT_DEFAULT_TARGET_ONLY can work
    # When COMPILER_RT_DEFAULT_TARGET_ONLY does not work, compiler-rt would only install several header files.
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON
    # Build builtins only (crt included)
    -DCOMPILER_RT_BUILD_CTX_PROFILE=OFF
    -DCOMPILER_RT_BUILD_GWP_ASAN=OFF
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF
    -DCOMPILER_RT_BUILD_MEMPROF=OFF
    -DCOMPILER_RT_BUILD_ORC=OFF
    -DCOMPILER_RT_BUILD_PROFILE=OFF
    -DCOMPILER_RT_BUILD_SANITIZERS=OFF
    -DCOMPILER_RT_BUILD_XRAY=OFF
    # Build for baremetal
    -DCOMPILER_RT_BAREMETAL_BUILD=ON
    # If not specified, it will install to :
    # /llvmtools/lib/clang/22/lib/linux/libclang_rt.builtins-arm.a
    # If specified:
    # /llvmtools/lib/clang/22/lib/baremetal/libclang_rt.builtins-arm.a
    # -DCOMPILER_RT_OS_DIR="baremetal"
    # Or install to:
    # /llvmtools/lib/clang/22/lib/arm-unknown-none-eabi/libclang_rt.builtins.a
    -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON
)

cmake \
    -G Ninja -S ../compiler-rt \
    "${cmake_args[@]}"
```