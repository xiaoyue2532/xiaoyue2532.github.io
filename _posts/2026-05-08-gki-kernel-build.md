# GKI Kernel Build

Let us take android12-5.10 GKI kernel building for example. Currently, android12-5.10-2026-04 (kernel 5.10.262) is the latest.

## Basically, we need git-repo

```shell
mkdir ~/.bin
PATH=~/.bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
chmod a+x ~/.bin/repo
```

## Get GKI kernel source

```shell
mkdir android-kernel && cd android-kernel
repo init -u https://mirrors.bfsu.edu.cn/git/AOSP/kernel/manifest -b common-android12-5.10-2026-04
```

## Start Build!

```shell
LTO=thin BUILD_CONFIG=common/build.config.gki.aarch64 build/build.sh
```

## Add KernelSU

```shell
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -
```

## Rebuild boot.img

```shell
tools/mkbootimg/unpack_bootimg.py \
    --boot_img boot.img \
    --format mkbootimg

# Maybe we shall modify `os_patch_level`
tools/mkbootimg/mkbootimg.py \
    --header_version 4 \
    --os_version 12.0.0 \
    --os_patch_level 2026-04 \
    --kernel out/android12-5.10/dist/Image \
    --ramdisk out/ramdisk \
    --cmdline '' \
    --output new-boot.img
```

## Identify kernel

```shell
# HyperOS 2.0.214
strings boot.img | grep 'Linux version'
# Linux version 5.10.236-android12-9-00003-gfb24cf99ad97-ab14313284 (build-user@build-host) (Android (7284624, based on r416183b) clang version 12.0.5 (https://android.googlesource.com/toolchain/llvm-project c935d99d7cf2016289302412d708641d52d2f7ee), LLD 12.0.5 (/buildbot/src/android/llvm-toolchain/out/llvm-project/lld c935d99d7cf2016289302412d708641d52d2f7ee)) #1 SMP PREEMPT Tue Oct 21 03:03:12 UTC 2025

# Google
strings kernelsu_patched_20260507_031155.img | grep 'Linux version'
# Linux version 5.10.252-android12-9-00009-gf960ed27302b-ab15212796 (build-user@build-host) (Android (7284624, based on r416183b) clang version 12.0.5 (https://android.googlesource.com/toolchain/llvm-project c935d99d7cf2016289302412d708641d52d2f7ee), LLD 12.0.5 (/buildbot/src/android/llvm-toolchain/out/llvm-project/lld c935d99d7cf2016289302412d708641d52d2f7ee)) #1 SMP PREEMPT Wed Apr 15 15:56:05 UTC 2026

# Mine
strings out/android12-5.10/dist/Image | grep 'Linux version'
# Linux version 5.10.252-android12-9-00002-gecf9d191c9af (build-user@build-host) (Android (7284624, based on r416183b) clang version 12.0.5 (https://android.googlesource.com/toolchain/llvm-project c935d99d7cf2016289302412d708641d52d2f7ee), LLD 12.0.5 (/buildbot/src/android/llvm-toolchain/out/llvm-project/lld c935d99d7cf2016289302412d708641d52d2f7ee)) #1 SMP PREEMPT Tue May 5 17:15:05 UTC 2026
```