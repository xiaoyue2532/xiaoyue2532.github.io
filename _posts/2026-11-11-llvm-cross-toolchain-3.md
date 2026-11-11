---
title: 构建 LLVM 交叉工具链-3 裸金属
date: 2026-11-11 18:00:00 +0800
categories: [ LLVM ]
---

```shell
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
    -DCMAKE_ASM_COMPILER=clang
    -DCMAKE_C_COMPILER_TARGET=arm-unknown-none-eabi
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

## Build llvm-libc
cmake \
    -G Ninja \
    -DCMAKE_INSTALL_PREFIX=/usr \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_COMPILER=arm-unknown-none-eabi-clang \
    -DCMAKE_C_COMPILER_WORKS=1 \
    -DCMAKE_CXX_COMPILER=arm-unknown-none-eabi-clang++ \
    -DCMAKE_CXX_COMPILER_WORKS=1 \
    -DLIBC_ENABLE_USE_BY_CLANG=ON \
    -DLIBC_TARGET_TRIPLE="arm-unknown-none-eabi" \
    -DLLVM_ENABLE_RUNTIMES="libc" \
    -DLLVM_LIBC_FULL_BUILD=ON \
    ../runtimes
```

```
tree /llvmtools/arm-unknown-none-eabi
.
├── include
│   ├── assert.h
│   ├── complex.h
│   ├── ctype.h
│   ├── elf.h
│   ├── errno.h
│   ├── features.h
│   ├── fenv.h
│   ├── float.h
│   ├── inttypes.h
│   ├── limits.h
│   ├── __llvm-libc-common.h
│   ├── llvm-libc-macros
│   │   ├── assert-macros.h
│   │   ├── baremetal
│   │   │   └── time-macros.h
│   │   ├── cfloat128-macros.h
│   │   ├── cfloat16-macros.h
│   │   ├── complex-macros.h
│   │   ├── containerof-macro.h
│   │   ├── elf-macros.h
│   │   ├── error-number-macros.h
│   │   ├── features-macros.h
│   │   ├── fenv-macros.h
│   │   ├── file-seek-macros.h
│   │   ├── float16-macros.h
│   │   ├── float-macros.h
│   │   ├── generic-error-number-macros.h
│   │   ├── inttypes-macros.h
│   │   ├── limits-macros.h
│   │   ├── locale-macros.h
│   │   ├── math-function-macros.h
│   │   ├── math-macros.h
│   │   ├── null-macro.h
│   │   ├── offsetof-macro.h
│   │   ├── signal-macros.h
│   │   ├── stdbit-macros.h
│   │   ├── stdckdint-macros.h
│   │   ├── stdfix-macros.h
│   │   ├── stdint-macros.h
│   │   ├── stdio-macros.h
│   │   ├── stdlib-macros.h
│   │   ├── sys-queue-macros.h
│   │   ├── time-macros.h
│   │   └── wchar-macros.h
│   ├── llvm-libc-types
│   │   ├── __atexithandler_t.h
│   │   ├── cfloat128.h
│   │   ├── cfloat16.h
│   │   ├── char16_t.h
│   │   ├── char32_t.h
│   │   ├── char8_t.h
│   │   ├── clockid_t.h
│   │   ├── clock_t.h
│   │   ├── cookie_io_functions_t.h
│   │   ├── div_t.h
│   │   ├── double_t.h
│   │   ├── Elf32_Addr.h
│   │   ├── Elf32_Chdr.h
│   │   ├── Elf32_Dyn.h
│   │   ├── Elf32_Ehdr.h
│   │   ├── Elf32_Half.h
│   │   ├── Elf32_Lword.h
│   │   ├── Elf32_Nhdr.h
│   │   ├── Elf32_Off.h
│   │   ├── Elf32_Phdr.h
│   │   ├── Elf32_Rela.h
│   │   ├── Elf32_Rel.h
│   │   ├── Elf32_Shdr.h
│   │   ├── Elf32_Sword.h
│   │   ├── Elf32_Sym.h
│   │   ├── Elf32_Verdaux.h
│   │   ├── Elf32_Verdef.h
│   │   ├── Elf32_Vernaux.h
│   │   ├── Elf32_Verneed.h
│   │   ├── Elf32_Versym.h
│   │   ├── Elf32_Word.h
│   │   ├── Elf64_Addr.h
│   │   ├── Elf64_Chdr.h
│   │   ├── Elf64_Dyn.h
│   │   ├── Elf64_Ehdr.h
│   │   ├── Elf64_Half.h
│   │   ├── Elf64_Lword.h
│   │   ├── Elf64_Nhdr.h
│   │   ├── Elf64_Off.h
│   │   ├── Elf64_Phdr.h
│   │   ├── Elf64_Rela.h
│   │   ├── Elf64_Rel.h
│   │   ├── Elf64_Shdr.h
│   │   ├── Elf64_Sword.h
│   │   ├── Elf64_Sxword.h
│   │   ├── Elf64_Sym.h
│   │   ├── Elf64_Verdaux.h
│   │   ├── Elf64_Verdef.h
│   │   ├── Elf64_Vernaux.h
│   │   ├── Elf64_Verneed.h
│   │   ├── Elf64_Versym.h
│   │   ├── Elf64_Word.h
│   │   ├── Elf64_Xword.h
│   │   ├── fenv_t.h
│   │   ├── fexcept_t.h
│   │   ├── FILE.h
│   │   ├── float128.h
│   │   ├── float_t.h
│   │   ├── imaxdiv_t.h
│   │   ├── int_hk_t.h
│   │   ├── int_hr_t.h
│   │   ├── int_k_t.h
│   │   ├── int_lk_t.h
│   │   ├── int_lr_t.h
│   │   ├── int_r_t.h
│   │   ├── __jmp_buf.h
│   │   ├── jmp_buf.h
│   │   ├── ldiv_t.h
│   │   ├── lldiv_t.h
│   │   ├── locale_t.h
│   │   ├── mbstate_t.h
│   │   ├── off64_t.h
│   │   ├── off_t.h
│   │   ├── __qsortcompare_t.h
│   │   ├── __qsortrcompare_t.h
│   │   ├── __search_compare_t.h
│   │   ├── sigjmp_buf.h
│   │   ├── sigset_t.h
│   │   ├── size_t.h
│   │   ├── ssize_t.h
│   │   ├── struct_lconv.h
│   │   ├── struct_timespec.h
│   │   ├── struct_timeval.h
│   │   ├── struct_tm.h
│   │   ├── suseconds_t.h
│   │   ├── time_t.h
│   │   ├── uint_uhk_t.h
│   │   ├── uint_uhr_t.h
│   │   ├── uint_uk_t.h
│   │   ├── uint_ulk_t.h
│   │   ├── uint_ulr_t.h
│   │   ├── uint_ur_t.h
│   │   ├── wchar_t.h
│   │   └── wint_t.h
│   ├── locale.h
│   ├── math.h
│   ├── setjmp.h
│   ├── stdbit.h
│   ├── stdckdint.h
│   ├── stdfix.h
│   ├── stdint.h
│   ├── stdio.h
│   ├── stdlib.h
│   ├── string.h
│   ├── strings.h
│   ├── sys
│   │   └── queue.h
│   ├── time.h
│   ├── uchar.h
│   ├── wchar.h
│   └── wctype.h
├── lib
│   ├── crt1.o
│   ├── libc.a
│   └── libm.a
└── usr -> .
```

## 尚未复现的

```shell
## According to https://llvm.org/devmtg/2024-10/slides/techtalk/Hosek-ModernEmbeddedDevelopment-with-LLVM.pdf
cmake \
    -G Ninja \
    -DCMAKE_INSTALL_PREFIX=/usr \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_ASM_COMPILER="/home/arch/llvmtools/bin/clang" \
    -DCMAKE_ASM_COMPILER_TARGET=armv7m-unknown-none-eabi \
    -DCMAKE_C_COMPILER="/home/arch/llvmtools/bin/clang" \
    -DCMAKE_C_COMPILER_TARGET=armv7m-unknown-none-eabi \
    -DCMAKE_CXX_COMPILER="/home/arch/llvmtools/bin/clang++" \
    -DCMAKE_CXX_COMPILER_TARGET=armv7m-unknown-none-eabi \
    -DCMAKE_AR="/home/arch/llvmtools/bin/llvm-ar" \
    -DCMAKE_NM="/home/arch/llvmtools/bin/llvm-nm" \
    -DCMAKE_RANLIB="/home/arch/llvmtools/bin/llvm-ranlib" \
    -DCMAKE_SYSROOT="/home/arch/llvmtools/lib/clang-runtimes/arm-unknown-none-eabi/thumb/v7-m/soft" \
    -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
    -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
    -DCOMPILER_RT_BAREMETAL_BUILD=ON \
    -DLIBCXX_CXX_ABI=none \
    -DLIBCXX_ENABLE_EXCEPTIONS=OFF \
    -DLIBCXX_ENABLE_FILESYSTEM=OFF \
    -DLIBCXX_ENABLE_LOCALIZATION=OFF \
    -DLIBCXX_ENABLE_MONOTONIC_CLOCK=OFF \
    -DLIBCXX_ENABLE_RANDOM_DEVICE=OFF \
    -DLIBCXX_ENABLE_RTTI=OFF \
    -DLIBCXX_ENABLE_SHARED=OFF \
    -DLIBCXX_ENABLE_THREADS=OFF \
    -DLIBCXX_ENABLE_UNICODE=OFF \
    -DLIBCXX_ENABLE_WIDE_CHARACTERS=OFF \
    -DLIBCXX_HAS_TERMINAL_AVAILABLE=OFF \
    -DLIBCXX_USE_COMPILER_RT=ON \
    -DLIBC_ENABLE_USE_BY_CLANG=ON \
    -DLIBC_TARGET_TRIPLE="armv7m-unknown-none-eabi" \
    -DLLVM_ENABLE_ASSERTIONS=OFF \
    -DLLVM_ENABLE_RUNTIMES="compiler-rt;libcxx;libc" \
    -DLLVM_LIBC_FULL_BUILD=ON \
    ../runtimes

## According to https://llvm.org/devmtg/2020-09/slides/Qadeer-LLVM_in_a_Bare_Metal_Environment.pdf
# which has libunwind
-DLIBUNWIND_ENABLE_SHARED=OFF
-DLIBUNWIND_ENABLE_THREADS=OFF
-DLIBUNWIND_IS_BAREMETAL=ON
-DLIBUNWIND_USE_COMPILER_RT=ON
# which has libc++abi
-DLIBCXXABI_BAREMETAL=ON
-DLIBCXXABI_ENABLE_EXCEPTIONS=ON
-DLIBCXXABI_ENABLE_SHARED=OFF
-DLIBCXXABI_ENABLE_THREADS=OFF
-DLIBCXXABI_USE_COMPILER_RT=ON
-DLIBCXXABI_USE_LLVM_UNWINDER=ON
# libc++
-DLIBCXX_BAREMETAL=ON
-DLIBCXX_CXX_ABI=libcxxabi
-DLIBCXX_ENABLE_EXCEPTIONS=ON
-DLIBCXX_ENABLE_FILESYSTEM=OFF
-DLIBCXX_ENABLE_MONOTONIC_CLOCK=OFF
-DLIBCXX_ENABLE_SHARED=OFF
-DLIBCXX_ENABLE_THREADS=OFF
-DLIBCXX_USE_COMPILER_RT=ON
```

## stub

LLVM-Libc 不提供 crt0.o. 不相干但值得注意的是，符号 `__libc_init_array` 由 `libc.a` 提供.

```shell
touch /llvmtools/lib/clang-runtimes/arm-unknown-none-eabi/thumb/v7-m/soft/lib/crt0.o
```

clang baremetal 将 crt0.o 作为 Gcc 安装的标准。

只有检测到 Gcc 安装，才会链接 crtbegin.o / crtend.o (-rtlib=libgcc) 
或 clang_rt.crtbegin.o / clang_rt.crtend.o (-rtlib=compiler-rt).

一旦检测到 Gcc 安装，就会链接 libgloss。

```shell
touch /llvmtools/arm-unknown-none-eabi/lib/crt0.o
touch /llvmtools/lib/clang-runtimes/arm-unknown-none-eabi/thumb/v7-m/soft/lib/libgloss.a
```

## Refereces

- [Better firmware with LLVM/Clang](https://interrupt.memfault.com/blog/arm-cortexm-with-llvm-clang)
- [](https://llvm.org/devmtg/2024-10/slides/techtalk/Hosek-ModernEmbeddedDevelopment-with-LLVM.pdf)
- [](https://llvm.org/devmtg/2020-09/slides/Qadeer-LLVM_in_a_Bare_Metal_Environment.pdf)
- [](https://llvm.org/devmtg/2024-10/slides/lightning/Smith-Using-llvm-libc.pdf)
- [](https://clang.llvm.org/docs/Multilib.html)