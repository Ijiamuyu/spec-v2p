# AP (Application Processor) 子系统设计

## 概述

AP 子系统是 V2P SoC 的高性能应用处理域，包含 4 核 VexRiscv-SMP (RV32) 处理器集群及配套的中断控制器、定时器、看门狗、调试模块等私密外设。AP 通过 AXI 总线访问公共外设（LSIO）和共享系统资源，与 FSI（安全核）及 NPU 之间通过 IPC_CTRL / SPINLOCK 进行核间通信。

## 处理器核

| 项目 | 说明 |
|------|------|
| IP | VexRiscv-SMP ×4（SpinalHDL）|
| 架构 | RV32IMAFDC |
| MMU | Sv32 页表方案 |
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
| `0x3005_9000` – `0x3005_9FFF` | 4KB | Debug Module (AP)，外部 JTAG 调试为主路径；`0x3005_9000` 窗供 FSI 片上访问，**权限访问机制未实现**（认证/访问控制待补），始终使能 |

### AP 可访问的共享资源

| 地址范围 | 容量 | 说明 |
|----------|------|------|
| `0x0001_0000` – `0x000B_7FFF` | 672KB | SYSTEM SRAM，核间共享数据区 |
| `0x000C_2000` – `0x000C_2FFF` | 4KB | IPC_CTRL，核间通信控制器（Doorbell + IPI）|
| `0x000C_3000` – `0x000C_3FFF` | 4KB | SPINLOCK，硬件同步锁（32 路 test-and-set）|
| `0x0010_0000` – `0x0010_FFFF` | 64KB | QSPI0（LSIO）|
| `0x0011_0000` – `0x0011_FFFF` | 64KB | QSPI1（LSIO）|
| `0x0012_0000` – `0x0012_FFFF` | 64KB | I2C0（LSIO，挂载 EEPROM 24C64）|
| `0x0013_0000` – `0x0013_FFFF` | 64KB | I2C1（LSIO）|
| `0x0014_0000` – `0x0014_FFFF` | 64KB | Reserved（DMA 已移至 `0x0020_0000`，见下）|
| `0x0015_0000` – `0x0015_FFFF` | 64KB | UART（扩展）（LSIO）|
| `0x0016_0000` – `0x0016_FFFF` | 64KB | GPIO（LSIO，无中断）|
| `0x0017_0000` – `0x0017_FFFF` | 64KB | Ethernet MAC（LSIO，内部 DMA）|
| `0x0018_0000` – `0x0018_FFFF` | 64KB | SD/MMC（LSIO，内部 DMA）|
| `0x0019_0000` – `0x0019_FFFF` | 64KB | RTC（LSIO）|
| `0x001A_0000` – `0x001A_FFFF` | 64KB | PWM（LSIO，6 通道 16-bit 脉宽调制器）|
| `0x0020_0000` – `0x002F_FFFF` | 1MB | DMA（SiFive FU540 PDMA，1MB 对齐；仅 AP 使用，FSI 只读寄存器）|
| `0xC000_0000` – `0xFFFF_FFFF` | 1GB | DDR3 主存 |

> **AXI 隔离**：AP 专有域硬件硬编码隔离，不可访问 FSI 及 NPU 私有外设空间。
>
> **LSIO**（`0x0010_0000`–`0x001A_FFFF`）：公共外设区域，AP 通过 PLIC 获取全部 LSIO 中断（QSPI0/1、I2C0/1、DMA、Ethernet、SD/MMC、RTC）。QSPI1/I2C1 在 FSI 域仅 Poll，但在 AP 域均可中断。

## 中断布局

### AP 本地中断 (CLINT)

CLINT（`0x3004_0000`，SpinalHDL `Clint`，寄存器偏移与 QEMU riscv_aclint 兼容）为每核提供 1 路软件中断和 1 路定时器中断（mtime），共 8 路本地中断。

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

PLIC（`0x3000_0000`，SpinalHDL `BmbPlic`）将 SoC 内所有外设中断集中路由至 AP 四核，支持优先级配置与中断屏蔽。

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | — | Reserved（PLIC ID 0 保留）|
| 1 | AP UART | Log 输出 |
| 2 | LSIO UART | 扩展串口 |
| 3 | LSIO QSPI0 | SPI Flash 等 |
| 4 | LSIO QSPI1 | SPI 外设 |
| 5 | LSIO I2C0 | EEPROM 及其他 I2C 设备 |
| 6 | LSIO I2C1 | 扩展 I2C |
| 7 | LSIO DMA done | DMA 传输完成 |
| 8 | LSIO DMA err | DMA 传输错误 |
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
| 29 | — | Reserved |
| 30 | — | Reserved |
| 31 | — | Reserved |

### PLIC 上下文与优先级

| 项目 | 说明 |
|------|------|
| 优先级 | 7 级（`priorityWidth=3`，0 为禁用，值越大优先级越高）|
| 上下文数 | 8 个（4 核 × M/S 模式），target 编号 = hart × 2 + mode（M=0，S=1）|
| threshold | `0x38000 + target × 0x1000` |
| claim / complete | threshold 基址 + `0x4`（读为 claim，写为 complete）|

> 上下文区落在 PLIC 256KB 窗内（8 个 target 填至 `0x3F004`），布局与 QEMU（`sifive_plic_create`，context_base=`0x38000`）及 RTOS 驱动（`PLIC_CTX_BASE`）一致。

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
| JTAG 路径 | 外部 TAP / BSCANE2（复用 Xilinx JTAG 引脚）→ DTM → DM |
| 调试能力 | 独立 halt/step 4 核 |
| 主机侧 | OpenOCD + GDB |

## UART（AP Dedicated）

AP 主 Log 输出口（自研 sifive 兼容 `uart`，`0x3005_8000`，寄存器集兼容 SiFive sifive-blocks），PLIC #1。

- AP 系统主控制台，用于内核启动日志、应用程序输出
- FSI 可通过 AXI 跨域读取 UART 寄存器以监控 AP 状态（日志轮询）

## DMA（通用 DMA，M2M）

AP 域通用 DMA 采用 **SiFive FU540 PDMA**（QEMU 模型 `sifive_pdma`，compatible `sifive,fu540-c000-pdma`），单实例 4 通道。DMA 为**软件配置型内存搬运引擎**（写 NEXT_SRC/NEXT_DST/NEXT_BYTES 指定地址），**仅支持 M2M 传输**（无外设 handshake 接口，外设 DMA 不可行）。

| 实例 | 地址 | 中断 (PLIC#) | 通道 |
|------|------|-------------|------|
| DMA | `0x0020_0000`（1MB） | done #7 / err #8 | ch0–3 |

### 通道分配约定（软件约定，非硬件绑定）

| 通道 | 用途 |
|------|------|
| ch0 | M2M 读（A→B） |
| ch1 | M2M 写（B→A） |
| ch2 | 预留（M2M 扩展） |
| ch3 | 预留（M2M 扩展） |

> 每通道 2 个中断（done/err，`CONTROL.DONE_IE`/`CONTROL.ERR_IE` 分别使能），多通道共享同一 done 源与同一 err 源（软件读 CONTROL 区分通道）。
> **IRQ 合并（方案 A）**：硬件 8 根 IRQ 线（4 通道 × done/err）在 RTL 层合并——4×done 线 OR → PLIC #7，4×err 线 OR → PLIC #8。
> **通道识别**：PDMA 无全局通道状态寄存器，软件收到中断后需轮询各通道 CONTROL 的 DONE/ERR 位（bit30/bit31）判断来源通道并清除。
> 权限：AP 读写，FSI 只读；FSI 不得发起传输。
> ⚠️ PDMA 仅支持 M2M（无外设 handshake），UART/SPI/I2C 外设 DMA 在 RTL 上不可行，外设传输走 CPU 中断。

## 外设清单

| IP | 来源 | 地址 | 说明 |
|----|------|------|------|
| PLIC | SpinalHDL `BmbPlic` | `0x3000_0000` | 平台级中断控制器，256KB |
| CLINT | SpinalHDL `Clint` | `0x3004_0000` | 核级本地中断控制器，寄存器偏移兼容 QEMU riscv_aclint |
| WDT (Core0) | SiFive sifive-blocks `wdt` | `0x3005_0000` | 两阶段看门狗 |
| WDT (Core1) | SiFive sifive-blocks `wdt` | `0x3005_1000` | 两阶段看门狗 |
| WDT (Core2) | SiFive sifive-blocks `wdt` | `0x3005_2000` | 两阶段看门狗 |
| WDT (Core3) | SiFive sifive-blocks `wdt` | `0x3005_3000` | 两阶段看门狗 |
| Timer (Core0) | OpenCores `timer` | `0x3005_4000` | 高精度定时器 |
| Timer (Core1) | OpenCores `timer` | `0x3005_5000` | 高精度定时器 |
| Timer (Core2) | OpenCores `timer` | `0x3005_6000` | 高精度定时器 |
| Timer (Core3) | OpenCores `timer` | `0x3005_7000` | 高精度定时器 |
| UART (AP Dedicated) | 自研 sifive 兼容 `uart` | `0x3005_8000` | 主 Log 输出口，寄存器集兼容 SiFive sifive-blocks |
| Debug Module (AP) | VexRiscv-SMP 内置 | `0x3005_9000` | JTAG 调试接口，始终使能 |

### AP 可访问的公共外设（LSIO）

| IP | 来源 | 地址 | 中断 (PLIC#) | 说明 |
|----|------|------|-------------|------|
| QSPI0 | SiFive sifive-blocks `spi` | `0x0010_0000` | #3 | SPI Flash；AP/FSI 读写，AP/FSI 中断 |
| QSPI1 | SiFive sifive-blocks `spi` | `0x0011_0000` | #4 | SPI 外设；AP/FSI 读写，AP 中断，FSI 仅 Poll |
| I2C0 | SiFive sifive-blocks `i2c`（compatible `sifive,fu540-c000-i2c`） | `0x0012_0000` | #5 | EEPROM 24C64 等；AP/FSI 读写，AP/FSI 中断 |
| I2C1 | SiFive sifive-blocks `i2c`（compatible `sifive,fu540-c000-i2c`） | `0x0013_0000` | #6 | 扩展 I2C；AP/FSI 读写，AP 中断，FSI 仅 Poll |
| DMA | SiFive FU540 PDMA | `0x0020_0000` | #7/#8 | 通用 DMA（M2M，1MB 对齐）；done #7，err #8；AP 读写，FSI 只读 |
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
