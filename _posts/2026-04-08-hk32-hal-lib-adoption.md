---
title: HK32 替代 STM32
date: 2026-04-08 23:30:00 +0800
categories: [ ARM 原理与应用, HK32 ]
---

画了一块 F103C8T6 最小系统板，采用国产芯片 HK32F103C8T6A 作为主控芯片，看中这款芯片的原因是相比 STM32F103C8T6 的 72Mhz 主频，该芯片的主频可以达到 120Mhz，而且在该频率下合理配置时钟树可以让 USB 正常运作。

相比国产 GD32F103C8T6（最高主频108Mhz，且该频率下没法分频得到 USB 48Mhz） 价格便宜几毛钱，同时 HK32 还配备 Cache，个人觉得比 GD32 基于 SRAM 的 Flash 零等待卖点更实用。

我是通过正点原子精英板入门 STM32 的，该开发板的主控为 STM23F103ZET6。本文从正点原子例程出发，透过修改工程、源码的方式，使 STM32 的 HAL 库在 HK32 芯片上运作。

## Prequires

- Keil 5.36
- STM32 HAL Library 1.8.7
- 正点原子 HAL 例程跑马灯实验

## 安装 HK32 KEIL 器件包

## 修改项目文件

点进“魔术棒”窗口：

- Device 选项卡，选中 HKMicrochip 下的 HK32F103C8T6A
- C++ 选项卡，在 Define 宏中 将 STM32F103xE 修改为 STM32F103xB
- Debug 选项卡，我修改调试器为手头的 ST-Link Debugger

## 修改 HAL 源码

根据 HK32 手册时钟树部分，主频 120Mhz 可以由 8Mhz HSE 经过 15 倍频得到。又根据 Flash 部分，主频在 120Mhz 时 Flash 应当等待 4 个机器周期。

修改 `User/main.c`：

```c++
int main(void)
{
    HAL_Init();                                 /* 初始化HAL库 */
    sys_stm32_clock_init(RCC_PLL_MUL15);         /* 设置时钟,120M */
    delay_init(120);                             /* 初始化延时函数 */
```

修改 `Drivers/STM32F1xx_HAL_Driver/Inc/stm32f1xx_hal_flash.h`，原本等待周期最多只有`FLASH_LATENCY_2`。这里加上等待 3、4 周期:

问了 DeepSeek，`FLASH_ACR_LATENCY_k` 是延迟的第 k 位数码（2^k），需要将各位数码组合才能得到延迟值`FLASH_LATENCY_n`。

```c++
#if   defined(FLASH_ACR_LATENCY)
/** @defgroup FLASH_Latency FLASH Latency
  * @{
  */
#define FLASH_LATENCY_0            0x00000000U                                  /*!< FLASH Zero Latency cycle */
#define FLASH_LATENCY_1            FLASH_ACR_LATENCY_0                          /*!< FLASH One Latency cycle */
#define FLASH_LATENCY_2            FLASH_ACR_LATENCY_1                          /*!< FLASH Two Latency cycles */
#define FLASH_LATENCY_3            FLASH_ACR_LATENCY_0 | FLASH_ACR_LATENCY_1    /*!< FLASH Three Latency cycles */
#define FLASH_LATENCY_4            FLASH_ACR_LATENCY_2                          /*!< FLASH Four Latency cycles */
```

修改 `Drivers/SYSTEM/sys/sys.c` 的 `sys_stm32_clock_init(uint32_t plln)` 函数，修改等待周期为 4：

```c++
    rcc_clk_init.APB1CLKDivider = RCC_HCLK_DIV2;                /* APB1分频系数为2 */
    rcc_clk_init.APB2CLKDivider = RCC_HCLK_DIV1;                /* APB2分频系数为1 */
    ret = HAL_RCC_ClockConfig(&rcc_clk_init, FLASH_LATENCY_4);  /* 同时设置FLASH延时周期为4WS，也就是5个CPU周期。 */
```

我的单板上只有 PB12 一个用户 LED，因此也对 `Drivers/BSP/LED` 下做了修改。