---
title: RK3506 折腾 - 1. 抽丝剥茧 分析烧录固件结构
date: 2026-03-24 01:29:00 +0800
categories: [ 嵌入式, RK3506 ]
---

购入了一块小巧的板子 Luckfox Lyra M，没有板载存储芯片，需要从 TF 卡启动。对于烧录 TF 卡的情况，Luckfox的Wiki上说要借助读卡器将TF卡与PC连接，
需要借助专门的烧录工具 SDDiskTool 才能将 update.img 写入 TF 卡。出于追求透明和开放的态度（其实是手上的读卡器不好使了），我研究了一下该工具是
怎么烧录 update.img 的。

~~rk-open-docs说选中SD启动等不同模式时，磁盘的布局会不同。~~

将U盘连接到PC，选中“SD启动”，选择官方提供的 Ubuntu 系统的 update.img文件，并开始创建。

![](https://img.cdn1.vip/i/69c18e748ac7d_1774292596.webp)

## 从可目视的角度解读固件结构

```
CMDLINE: mtdparts=rk29xxnand:0x00002000@0x00002000(uboot),0x00006000@0x00004000(boot),-@0x00010000(rootfs:grow)
```

通过解读 Ubuntu 系统包的 parameter.txt 内容，我们可以知道：

|    分区名称    |       起始扇区        | 结束扇区  |        扇区数        |
|:----------:|:-----------------:|:-----:|:-----------------:|
|   uboot    | 0x00002000(8192)  | 16383 | 0x00002000(8192)  |
|    boot    | 0x00004000(16384) | 40959 | 0x00006000(24576) |
| **Unused** |       40960       | 65535 |         ?         |
|   rootfs   | 0x00010000(65536) |   -   |         -         |

以上与烧录后的U盘分区结构一致：

|    分区名称    | 分区大小 |
|:----------:|:----:|
|   uboot    |  4M  |
|    boot    | 12M  |
| **Unused** | 12M  |
|   rootfs   | 剩余空间 |

有了以上可见数据，我心中还有几个疑问：

- 查阅 Rockchip 的开源资料，单板首先由BootROM中的程序拉起，寻找外存0x00000040扇区偏移处的idloader.img、作为PPL/SPL初始化DDR，
  然后才调起0x00004000扇区偏移处的uboot。这个idloader.img并不是可见的U盘分区，能看见的Uboot分区并不在开源资料中的扇区位置，
  **开源资料有一定的道理，但是与该单板脱节了**。

- uboot分区从8192扇区才开始，可能我们uboot前面还有某些引导数据？boot与rootfs之间还有12M的空闲区域，启动uboot的引导程序或许又在这里？

- 看到Ubuntu_Luckfox_Lyra_MicroSD_250417目录下还有个MiniLoaderAll.bin， 我猜想MiniLoaderAll.bin是不是所谓的idloader，
  也被写入了sd卡的某处。

看来仅靠能看的见的部分是没法分析了。

## 从十六进制角度外存结构

将烧录了固件的U盘接到 Android 平板上，读出rootfs分区之前的原始数据。已经知道sd卡的扇区大小必须为512B，透过官方工具、选中烧写SD卡模式时U盘的扇区
大小也是512B，即读出前32MB。

```sh
# dd if=/dev/block/sdg of=/sdcard/udisk.raw bs=512B count=65536
```

将原始U盘数据拷贝到电脑上，并使用 VSCode 的 HexEditor 功能查看十六进制数据。数据的结构如下：

|     数据段      | 起始扇区  |  扇区数   |         扇区范围          |  字节数   |         字节范围          |
|:------------:|:-----:|:------:|:---------------------:|:------:|:---------------------:|
|     MBR      |   0   |   1    |      0x00000000       |  512B  | 0x00000000~0x000001FF |
|     GPT      | 1-33  |   33   | 0X00000001~0x00000021 | 16.5KB | 0x00000200~0x000043FF |
| **Unused-1** | 34-63 |   30   | 0x00000022~0x0000003F |  15KB  | 0x00004400~0x00007FFF |
| idblock.img  |  64   |  384   | 0x00000040~0x0000017F | 192KB  | 0x00008000~0x00037FFF |
| **Unused-2** |       |        |                       |        |                       |
|  uboot.img   | 8192  |   8K   | 0x00002000~0x00003FFF |  4MB   | 0x00400000~0x007FFFFF |
|   boot.img   | 16384 |  24K   | 0x00004000~0x0009FFFF |  12MB  | 0x00800000~0x00BFFFFF |
| **Unused-3** |       |        |                       |  12MB  | 0x00C00000~0X01FFFFFF |
|  rootfs.img  | 65536 |   -    |     0x00010000 之后     |   -    |    0x0x02000000 之后    |
|    备份GPT     |       | 最后33扇区 |        N-32~N         | 16.5KB |                       |

### GPT 表

字节0x00000200处，GPT表开始。有字面值`EFI PART`作为标志。

|         字节范围          |     含义     |
|:---------------------:|:----------:|
| 0x00000200~0x000003FF |   GPT表头    |
| 0x00000400~0x0000047F | uboot分区条目  |
| 0x00000480~0x000004FF |  boot分区条目  |
| 0x00000500~0x0000057F | rootfs分区条目 |
| 0x00000580~0x000043FF | GPT空条目，全零  |

字节0x000043FF处，GPT表结束。在Unused-1部分，我看见未擦除的脏数据。
字节0x00008000之后，十六进制数据明显和Rockchip启动有关。会是固件包里的 MiniLoaderAll.bin 吗？

### MiniLoaderAll.bin

MiniLoaderAll.bin有`IDR`头部标志，字节0x00008000之后的原始数据中也有多处`IDR`标志，可是几乎能断定和 MiniLoaderAll.bin 无关。

```shell
$ cd Ubuntu_Luckfox_Lyra_MicroSD_250417 && sha256sum MiniLoaderAll.bin
# 346bb66e44e2b7521942e54ba6198fd86e258331d3a66d4a077f0ab7056d7a23  Ubuntu_Luckfox_Lyra_MicroSD_250417/MiniLoaderAll.bin

$ xxd MiniLoaderAll.bin | head
# 00000000: 4c44 5220 6600 0101 0000 0000 0001 e907  LDR f...........
# 00000010: 0b18 140b 2e46 3035 3302 6600 0000 3901  .....F053.f...9.
# 00000020: d800 0000 3903 1101 0000 3900 0101 0000  ....9.....9.....
# 00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000060: 0000 0000 0000 3901 0000 0055 0073 0062  ......9....U.s.b
# 00000070: 0048 0065 0061 0064 0000 0000 0000 0000  .H.e.a.d........
# 00000080: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000090: 0000 00bc 0100 0000 0800 0000 0000 0039  ...............9
```

~~rk-open-doc说MiniLoaderAll.bin是搭配RkDevTool，给有板载存储的机型刷固件等等的功能。相当于老开源文档中ddr.bin的角色~~

编译Luckfox_Lyra_SDK_250815中的u-boot后，可以看到 MiniLoaderAll.bin 是指向 rk3506_spl_loader_v1.04.110.bin 的符号链接。

```shell
$ cd output/firmware && ls -l
# MiniLoaderAll.bin -> spl_loader.img

$ sha256sum rk3506_spl_loader_v1.04.110.bin
# 56c76a63b38088fb4785bb28b22a5f3df2ea8cfb773a3a22c5cc585907b69de6  rk3506_spl_loader_v1.04.110.bin

$ xxd rk3506_spl_loader_v1.04.110.bin | head
# 00000000: 4c44 5220 6600 0101 0000 0000 0001 ea07  LDR f...........
# 00000010: 0318 021b 1846 3035 3302 6600 0000 3901  .....F053.f...9.
# 00000020: d800 0000 3903 1101 0000 3900 0101 0000  ....9.....9.....
# 00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000060: 0000 0000 0000 3901 0000 0055 0073 0062  ......9....U.s.b
# 00000070: 0048 0065 0061 0064 0000 0000 0000 0000  .H.e.a.d........
# 00000080: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000090: 0000 00bc 0100 0000 0800 0000 0000 0039  ...............9
```

###  idblock.img

经过研究，我发现rk3506_spl_loader.bin是通过rkbin的boot_merger工具生成的，同时生成的还有rk3506_idblock.img。

```shell
$ pwd
# rkbin
$ ./tools/boot_merger ./RKBOOT/RK3506MINIALL.ini
# ********boot_merger ver 1.35********
# Info:Pack loader ok.
# creating new idblock from loader...
# idblock binary saving at rk3506_idblock_v1.04.110.img
$ ls -l rk3506*
# -rw-r--r-- 1 arch arch 196608 Mar 24 02:27 rk3506_idblock_v1.04.110.img
# -rw-r--r-- 1 arch arch 268736 Mar 24 02:27 rk3506_spl_loader_v1.04.110.bin
```

同时在sd卡的字节0x00008000处，也有`RKNS`标志。

```shell
$ xxd rk3506_idblock_v1.04.110.img | head
# 00000000: 524b 4e53 0000 0000 8001 0200 0100 0000  RKNS............
# 00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
# 00000070: 0000 0000 0000 0000 0400 2800 ffff ffff  ..........(.....
# 00000080: 0000 0000 0100 0000 0000 0000 0000 0000  ................
# 00000090: 7c86 3123 5a0b 293e fa43 3b55 6394 f695  |.1#Z.)>.C;Uc...
```

所以uboot之前藏了一个idblock.img，这是rockchip提供的闭源预编译TPL。

在下一节我将验证以上分析是否正确。

这是可恶的老文档


他说的是比较新的文档 可惜没有RK3506的
https://github.com/mfkiwl/rk-open-docs/

他俩是说 GPT 布局的
https://github.com/mfkiwl/rk-open-docs/blob/master/MMC/Rockchip_Developer_Guide_SD_Boot_CN.md
https://lo01.g77k.com/aeb/docs/cn/Common/MMC/Rockchip_Developer_Guide_SD_Boot_CN.pdf

他说
FIT SPL TPL
https://github.com/mfkiwl/rk-open-docs/blob/master/UBOOT/CH10-SPL.md
https://github.com/mfkiwl/rk-open-docs/blob/master/UBOOT/CH11-TPL.md
https://github.com/mfkiwl/rk-open-docs/blob/master/UBOOT/CH12-FIT.md

大奥特曼打小怪兽的RK系列说的很好 比较贴切 做了大量工作 可惜不是FIT式启动流程
https://www.cnblogs.com/zyly/p/17418892.html
