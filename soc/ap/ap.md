# AP (Application Processor) 子系统设计

## 概述

AP 子系统是 V2P SoC 的高性能应用处理域，包含 4 核 VexRiscv-SMP (RV64) 处理器集群及配套的中断控制器、定时器、看门狗、调试模块等私密外设。AP 通过 AXI 总线访问公共外设（LSIO）和共享系统资源，与 FSI（安全核）及 NPU 之间通过 IPC_CTRL / SPINLOCK 进行核间通信。

## 处理器核

| 项目 | 说明 |
|------|------|
| IP | VexRiscv-SMP ×4（SpinalHDL）|
| 架构 | RV64IMAFDC |
| MMU | Sv39 页表方案 |
| 缓存 | 硬件维护缓存一致性（SMP 协同）|
| 调试 | 内置 RISC-V Debug Spec v0.13 Plugin（halt/step/resume、寄存器/内存访问）|

> 四核共享内存一致性域，通过 AXI 互联访问 DDR3 主存及所有公共外设。

## 内存布局

### AP 私密外设

| 地址范围 | 容量 | 说明 |
|----------|------|------|
| `0x3000_0000` – `0x3003_FFFF` | 256KB | PLIC，外部中断控制器 |
| `0x3004_0000` – `0x3004_FFFF` | 64KB | CLINT，本地中断控制器 |
| `0x3005_0000` – `0x3005_0FFF` | 4KB | WDT (Core0) |
| `0x3005_1000` – `0x3005_1FFF` | 4KB | WDT (Core1) |
| `0x3005_2000` – `0x3005_2FFF` | 4KB | WDT (Core2) |
| `0x3005_3000` – `0x3005_3FFF` | 4KB | WDT (Core3) |
| `0x3005_4000` – `0x3005_4FFF` | 4KB | Timer (Core0) |
| `0x3005_5000` – `0x3005_5FFF` | 4KB | Timer (Core1) |
| `0x3005_6000` – `0x3005_6FFF` | 4KB | Timer (Core2) |
| `0x3005_7000` – `0x3005_7FFF` | 4KB | Timer (Core3) |
| `0x3005_8000` – `0x3005_8FFF` | 4KB | UART（AP Dedicated），主 Log 输出口；FSI 可跨域读取以监控 AP 状态 |
| `0x3005_9000` – `0x3005_9FFF` | 4KB | Debug Module (AP)，JTAG 调试接口，始终使能 |

### AP 可访问的共享资源

| 地址范围 | 容量 | 说明 |
|----------|------|------|
| `0x0001_0000` – `0x000B_7FFF` | 672KB | SYSTEM SRAM，核间共享数据区 |
| `0x000C_2000` – `0x000C_2FFF` | 4KB | IPC_CTRL，核间通信控制器（Doorbell + IPI）|
| `0x000C_3000` – `0x000C_3FFF` | 4KB | SPINLOCK，硬件同步锁（32 路 test-and-set）|
| `0x0010_0000` – `0x0010_FFFF` | 64KB | QSPI0（LSIO）|
| `0x0011_0000` – `0x0011_FFFF` | 64KB | QSPI1（LSIO）|
| `0x0012_0000` – `0x0012_FFFF` | 64KB | I2C0（LSIO，挂载 EEPROM 24LC02）|
| `0x0013_0000` – `0x0013_FFFF` | 64KB | I2C1（LSIO）|
| `0x0014_0000` – `0x0014_FFFF` | 64KB | DMA（LSIO，仅 AP 使用，FSI 只读寄存器）|
| `0x0015_0000` – `0x0015_FFFF` | 64KB | UART（扩展）（LSIO）|
| `0x0016_0000` – `0x0016_FFFF` | 64KB | GPIO（LSIO，无中断）|
| `0x0017_0000` – `0x0017_FFFF` | 64KB | Ethernet MAC（LSIO，内部 DMA）|
| `0x0018_0000` – `0x0018_FFFF` | 64KB | SD/MMC（LSIO，内部 DMA）|
| `0x0019_0000` – `0x0019_FFFF` | 64KB | RTC（LSIO）|
| `0x001A_0000` – `0x001A_FFFF` | 64KB | PWM（LSIO，6 通道 16-bit 脉宽调制器）|
| `0xC000_0000` – `0xFFFF_FFFF` | 1GB | DDR3 主存 |

> **AXI 隔离**：AP 专有域硬件硬编码隔离，不可访问 FSI 及 NPU 私有外设空间。
>
> **LSIO**（`0x0010_0000`–`0x001A_FFFF`）：公共外设区域，AP 通过 PLIC 获取全部 LSIO 中断（QSPI0/1、I2C0/1、DMA、Ethernet、SD/MMC、RTC）。QSPI1/I2C1 在 FSI 域仅 Poll，但在 AP 域均可中断。

## 中断布局

### AP 本地中断 (CLINT)

CLINT（`0x3004_0000`，OpenPULP）为每核提供 1 路软件中断和 1 路定时器中断（mtime），共 8 路本地中断。

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | 软件中断 (Core0) | AP 核间通信 |
| 1 | 软件中断 (Core1) | AP 核间通信 |
| 2 | 软件中断 (Core2) | AP 核间通信 |
| 3 | 软件中断 (Core3) | AP 核间通信 |
| 4 | 定时器中断 (Core0) | CLINT mtime 作操作系统时基 |
| 5 | 定时器中断 (Core1) | CLINT mtime 作操作系统时基 |
| 6 | 定时器中断 (Core2) | CLINT mtime 作操作系统时基 |
| 7 | 定时器中断 (Core3) | CLINT mtime 作操作系统时基 |

> AP 私有 Timer（OpenCores `timer`，PLIC #16–19）用于高精度计时及额外定时中断，与 CLINT mtime 互补。

### AP 外部中断 (PLIC)

PLIC（`0x3000_0000`，OpenHW core-v-plic）将 SoC 内所有外设中断集中路由至 AP 四核，支持优先级配置与中断屏蔽。

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | — | Reserved（PLIC ID 0 保留）|
| 1 | AP UART | Log 输出 |
| 2 | LSIO UART | 扩展串口 |
| 3 | LSIO QSPI0 | SPI Flash 等 |
| 4 | LSIO QSPI1 | SPI 外设 |
| 5 | LSIO I2C0 | EEPROM 及其他 I2C 设备 |
| 6 | LSIO I2C1 | 扩展 I2C |
| 7 | LSIO DMA | DMA 传输完成 |
| 8 | LSIO GPIO | 无中断，仅 Poll |
| 9 | Ethernet | 网络收发 |
| 10 | SD/MMC | 存储卡 |
| 11 | AP Core0 WDT | 每核独立看门狗预警 |
| 12 | AP Core1 WDT | 每核独立看门狗预警 |
| 13 | AP Core2 WDT | 每核独立看门狗预警 |
| 14 | AP Core3 WDT | 每核独立看门狗预警 |
| 15 | LSIO RTC | 实时时钟 |
| 16 | AP Core0 Timer | 每核独立高精度计时 |
| 17 | AP Core1 Timer | 每核独立高精度计时 |
| 18 | AP Core2 Timer | 每核独立高精度计时 |
| 19 | AP Core3 Timer | 每核独立高精度计时 |
| 20 | — | Reserved |
| 21 | Doorbell DB_NPU_AP0 | NPU→AP0 核间通知 |
| 22 | Doorbell DB_NPU_AP1 | NPU→AP1 核间通知 |
| 23 | Doorbell DB_NPU_AP2 | NPU→AP2 核间通知 |
| 24 | Doorbell DB_NPU_AP3 | NPU→AP3 核间通知 |
| 25 | Doorbell DB_FSI_AP0 | FSI→AP0 核间通知 |
| 26 | Doorbell DB_FSI_AP1 | FSI→AP1 核间通知 |
| 27 | Doorbell DB_FSI_AP2 | FSI→AP2 核间通知 |
| 28 | Doorbell DB_FSI_AP3 | FSI→AP3 核间通知 |

## 看门狗 (WDT)

AP 四核各有独立 WDT（SiFive sifive-blocks `wdt`，OpenCores `watch_dog` 同源），支持两阶段超时机制：

| 阶段 | 超时事件 | 动作 |
|------|----------|------|
| 第一次超时 | WDT 超时预警 | 向 PLIC 发送中断（各核对应 PLIC #11–#14），软件可刷新 WDT 阻止最终复位 |
| 最终超时 | 软件未及时刷新 | 向 FSI PMU 发送复位请求（FSI PLIC #2–#5），FSI 固件收到中断后复位对应 Core 或整个 AP 域 |

> **复位路径**：AP WDT → PLIC 预警（软件可恢复）→ FSI PLIC 中断 → FSI 固件决策 → PMU `ap_rst_n` 输出。

## 定时器

AP 四核各有独立高精度 Timer（OpenCores `timer`，`0x3005_4000`–`0x3005_7FFF`），通过 PLIC #16–#19 上报中断。与 CLINT mtime 的关系：

| 定时器 | 地址 | 中断路径 | 用途 |
|--------|------|----------|------|
| CLINT mtime | `0x3004_0000` 内部 | 本地中断 #4–#7 | OS 时基（RISC-V 标准）|
| Core Timer ×4 | `0x3005_4000`–`0x3005_7FFF` | PLIC #16–#19 | 高精度计时、额外定时中断 |

## 调试 (Debug)

AP 调试**始终使能**，不受 PMU 控制。

| 项目 | 说明 |
|------|------|
| 协议 | RISC-V Debug Spec v0.13（halt/step/resume、寄存器/内存访问）|
| JTAG 路径 | BSCANE2（复用 Xilinx JTAG 引脚）→ DTM → DM |
| 调试能力 | 独立 halt/step 4 核 |
| 主机侧 | OpenOCD + GDB |

## UART（AP Dedicated）

AP 主 Log 输出口（SiFive sifive-blocks `uart`，`0x3005_8000`），PLIC #1。

- AP 系统主控制台，用于内核启动日志、应用程序输出
- FSI 可通过 AXI 跨域读取 UART 寄存器以监控 AP 状态（日志轮询）

## 外设清单

| IP | 来源 | 地址 | 说明 |
|----|------|------|------|
| PLIC | OpenHW core-v-plic | `0x3000_0000` | 平台级中断控制器，256KB |
| CLINT | OpenPULP | `0x3004_0000` | 核级本地中断控制器 |
| WDT (Core0) | SiFive sifive-blocks `wdt` | `0x3005_0000` | 两阶段看门狗 |
| WDT (Core1) | SiFive sifive-blocks `wdt` | `0x3005_1000` | 两阶段看门狗 |
| WDT (Core2) | SiFive sifive-blocks `wdt` | `0x3005_2000` | 两阶段看门狗 |
| WDT (Core3) | SiFive sifive-blocks `wdt` | `0x3005_3000` | 两阶段看门狗 |
| Timer (Core0) | OpenCores `timer` | `0x3005_4000` | 高精度定时器 |
| Timer (Core1) | OpenCores `timer` | `0x3005_5000` | 高精度定时器 |
| Timer (Core2) | OpenCores `timer` | `0x3005_6000` | 高精度定时器 |
| Timer (Core3) | OpenCores `timer` | `0x3005_7000` | 高精度定时器 |
| UART (AP Dedicated) | SiFive sifive-blocks `uart` | `0x3005_8000` | 主 Log 输出口 |
| Debug Module (AP) | VexRiscv-SMP 内置 | `0x3005_9000` | JTAG 调试接口，始终使能 |

### AP 可访问的公共外设（LSIO）

| IP | 来源 | 地址 | 中断 (PLIC#) | 说明 |
|----|------|------|-------------|------|
| QSPI0 | SiFive sifive-blocks `spi` | `0x0010_0000` | #3 | SPI Flash；AP/FSI 读写，AP/FSI 中断 |
| QSPI1 | SiFive sifive-blocks `spi` | `0x0011_0000` | #4 | SPI 外设；AP/FSI 读写，AP 中断，FSI 仅 Poll |
| I2C0 | SiFive sifive-blocks `i2c` | `0x0012_0000` | #5 | EEPROM 等；AP/FSI 读写，AP/FSI 中断 |
| I2C1 | SiFive sifive-blocks `i2c` | `0x0013_0000` | #6 | 扩展 I2C；AP/FSI 读写，AP 中断，FSI 仅 Poll |
| DMA | SiFive sifive-blocks `dma` | `0x0014_0000` | #7 | 通用 DMA；AP 读写，FSI 只读 |
| UART (扩展) | SiFive sifive-blocks `uart` | `0x0015_0000` | #2 | 扩展串口；AP/FSI 读写，AP 中断 |
| GPIO | SiFive sifive-blocks `gpio` | `0x0016_0000` | — | AP/FSI 读写；无中断 |
| Ethernet MAC | OpenCores `ethmac`（保留，QEMU `opencores_eth` 模型现成） | `0x0017_0000` | #9 | 内部 DMA；AP/FSI 读写，AP 中断 |
| SD/MMC | 标准 SDHCI | `0x0018_0000` | #10 | 内部 DMA；AP/FSI 读写，AP 中断 |
| RTC | SiFive sifive-blocks `mockaon` RTC | `0x0019_0000` | #15 | 实时时钟；AP/FSI 读写，AP 中断 |
| PWM | SiFive sifive-blocks `pwm` | `0x001A_0000` | — | AP/FSI 读写；无中断，6 通道 |

## 复位与启动

### 复位向量

| 项目 | 说明 |
|------|------|
| 复位地址 | `0xC000_0000`（DDR3 起始地址）|
| 引导流程 | FSI 加载 OpenSBI 至 DDR，然后释放 AP 复位信号 |
| 复位信号 | `ap_rst_n`，由 PMU 控制 |

### 启动流程

AP 自身不参与早期启动。由 FSI 安全核完成以下步骤后释放 AP 复位：

1. FSI BootROM 自校验
2. FSI 固件加载至 `0x000D_0000`
3. FSI 固件验签并加载 AP 镜像（OpenSBI + OS）至 DDR `0xC000_0000`
4. FSI 通过 PMU 释放 `ap_rst_n`，AP 从 `0xC000_0000` 开始执行

### 复位源

AP 域复位可由以下事件触发：

| 复位源 | 说明 |
|--------|------|
| POR | 上电复位，全芯片复位 |
| AP WDT 最终超时 | 经 FSI 中断 → FSI 固件 → PMU 输出 `ap_rst_n` |
| FSI SW 写 PMU 寄存器 | FSI 固件主动复位 AP 域 |
| Debug 请求 | JTAG 调试器发起 |

## 核间通信

AP 通过以下机制与 FSI、NPU 通信：

- **IPC_CTRL**（`0x000C_2000`）：含 Doorbell 寄存器，支持 AP→FSI（PLIC FSI #10）、AP→NPU（NPU 中断 #1）、NPU→AP（PLIC #21–#24）、FSI→AP（PLIC #25–#28）的双向通知
- **SPINLOCK**（`0x000C_3000`）：32 路硬件 test-and-set 锁，用于共享数据结构的互斥访问
- **SYSTEM SRAM**（`0x0001_0000`–`0x000B_7FFF`）：核间共享数据区，配合 SPINLOCK 实现核间通信协议

详见 [`../ip/ipc.md`](../ip/ipc.md)。

## 时钟与电源

| 项目 | 说明 |
|------|------|
| 时钟门控 | PMU `PMU_CLK_GATE.bit0` 控制 AP 域时钟开关 |
| 复位输出 | PMU 输出 `ap_rst_n` 控制 AP 域复位 |
| 调试控制 | PMU `PMU_DBG_CTRL.bit0` 恒为 1，AP JTAG 始终使能 |

> AP 域时钟门控由 FSI 通过 PMU 管理。AP 不可自行门控自身时钟。
