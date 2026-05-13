---
title: RISCV 大师
date: 2026-04-09 00:21:00 +0800
categories: [ RISCV ]
---

## RISCV 大师

### Kernel

参见https://linuxdev.cc/article/a0idrf.html

```shell
qemu-system-riscv64 -machine virt,dumpdtb=virt.dtb
dtc -i dtb virt.dtb > virt.dts
```

```
		serial@10000000 {
			interrupts = <0x0a>;
			interrupt-parent = <0x03>;
			clock-frequency = "", "8@";
			reg = <0x00 0x10000000 0x00 0x100>;
			compatible = "ns16550a";
		};
```

可以看到riscv64的serial驱动是16550，需要开启配置：
CONFIG_SERIAL_8250
CONFIG_SERIAL_8250_CONSOLE
CONFIG_SERIAL_OF_PLATFORM

### u-boot 2026.04

对于riscv64-lp64f-linux-musl关闭
RISCV_ISA_D
对于riscv64-lp64-linux-musl关闭
RISCV_ISA_F
RISCV_ISA_D