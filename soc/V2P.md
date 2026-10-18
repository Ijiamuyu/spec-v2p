# V2P SoC 硬件设计

## 内存布局

> 未列出的地址范围为 Reserved。访问 Reserved 区域将产生错误响应。

### 区域 0: `0x0000_0000` – `0x0FFF_FFFF`（Boot & 系统资源）

| 地址范围 | 容量 | 归属 | 说明 |
|----------|------|------|------|
| `0x0000_0000` – `0x0000_0FFF` | 4KB | FSI | eFuse OTP，位流固化，只读不可写，详见 [`fsi/fsi.md`](fsi/fsi.md) |
| `0x0000_1000` – `0x0000_7FFF` | 28KB | FSI | BootROM 代码区，独立 BRAM，仅 FSI 可读 |
| `0x0000_8000` – `0x0000_FFFF` | 32KB | — | Reserved |
| `0x0001_0000` – `0x000B_7FFF` | 672KB | 全局 | SYSTEM SRAM，核间共享数据区，所有主设备可访问 |
| `0x000B_8000` – `0x000C_1FFF` | 40KB | NPU | NPU TCM（Coral ITCM 8KB + DTCM 32KB）|
| `0x000C_2000` – `0x000C_2FFF` | 4KB | 全局 | IPC_CTRL，核间通信控制器，所有主设备可访问，详见 [`ip/ipc.md`](ip/ipc.md) |
| `0x000C_3000` – `0x000C_3FFF` | 4KB | 全局 | **SPINLOCK**，硬件同步锁控制器，所有主设备可读写，详见 [`ip/ipc.md`](ip/ipc.md) |
| `0x000C_4000` – `0x000C_FFFF` | 48KB | — | Reserved |
| `0x000D_0000` – `0x000E_FFFF` | 128KB | FSI | FSI 私有 SRAM，固件加载目标 |
| `0x000F_0000` – `0x000F_FFFF` | 64KB | — | Reserved |
| `0x0010_0000` – `0x0010_FFFF` | 64KB | 全局 | QSPI0 | AP/FSI 读写；AP 中断使能，FSI 中断使能 |
| `0x0011_0000` – `0x0011_FFFF` | 64KB | 全局 | QSPI1 | AP/FSI 读写；AP 中断使能，FSI 仅 Poll |
| `0x0012_0000` – `0x0012_FFFF` | 64KB | 全局 | I2C0 | AP/FSI 读写；AP 中断使能，FSI 中断使能；挂载 EEPROM 24C64 |
| `0x0013_0000` – `0x0013_FFFF` | 64KB | 全局 | I2C1 | AP/FSI 读写；AP 中断使能，FSI 仅 Poll |
| `0x0014_0000` – `0x0014_FFFF` | 64KB | — | Reserved（DMA 已移至 `0x0020_0000`）|
| `0x0015_0000` – `0x0015_FFFF` | 64KB | 全局 | UART（扩展）| AP/FSI 读写；AP 中断使能 |
| `0x0016_0000` – `0x0016_FFFF` | 64KB | 全局 | GPIO | AP/FSI 读写；无中断 |
| `0x0017_0000` – `0x0017_FFFF` | 64KB | 全局 | Ethernet MAC | AP/FSI 读写；内部 DMA，AP 中断使能 |
| `0x0018_0000` – `0x0018_FFFF` | 64KB | 全局 | SD/MMC | AP/FSI 读写；内部 DMA，AP 中断使能 |
| `0x0019_0000` – `0x0019_FFFF` | 64KB | 全局 | RTC | AP/FSI 读写；AP 中断使能 |
| `0x001A_0000` – `0x001A_FFFF` | 64KB | 全局 | PWM | AP/FSI 读写；无中断，6 通道脉宽调制器 |
| `0x001B_0000` – `0x001F_FFFF` | 320KB | — | Reserved（预留扩展）|
| `0x0020_0000` – `0x002F_FFFF` | 1MB | 全局 | DMA（SiFive FU540 PDMA，1MB 对齐；AP 读写，FSI 只读）|
| `0x0030_0000` – `0x0FFF_FFFF` | ~255MB | — | Reserved（后续可能添加系统控制寄存器映射）|

### 区域 1: `0x1000_0000` – `0x1FFF_FFFF`（FSI IP 内存映射）

| 地址范围 | 容量 | 归属 | 说明 |
|----------|------|------|------|
| `0x1000_0000` – `0x1000_FFFF` | 64KB | FSI | AES-256 |
| `0x1001_0000` – `0x1001_FFFF` | 64KB | FSI | SHA-256 |
| `0x1002_0000` – `0x1002_FFFF` | 64KB | FSI | RSA-2048 |
| `0x1003_0000` – `0x1003_FFFF` | 64KB | FSI | TRNG |
| `0x1004_0000` – `0x1004_0FFF` | 4KB | FSI | WDT |
| `0x1004_1000` – `0x1004_1FFF` | 4KB | FSI | UART |
| `0x1004_2000` – `0x1004_2FFF` | 4KB | FSI | Timer |
| `0x1004_3000` – `0x1007_FFFF` | ~240KB | — | Reserved |
| `0x1008_0000` – `0x100B_FFFF` | 256KB | FSI | PLIC（core-v-plic，M模式）|
| `0x100C_0000` – `0x100C_FFFF` | 64KB | FSI | CLINT |
| `0x100D_0000` – `0x100D_0FFF` | 4KB | FSI | PMU，电源管理与复位控制器，详见「PMU」节 |
| `0x100D_1000` – `0x1FFF_FFFF` | ~255MB | — | Reserved |

### 区域 2: `0x3000_0000` – `0x3FFF_FFFF`（AP 内存映射）

| 地址范围 | 容量 | 归属 | 说明 |
|----------|------|------|------|
| `0x3000_0000` – `0x3003_FFFF` | 256KB | AP | PLIC（SpinalHDL `BmbPlic`，M/S模式），外部中断控制器 |
| `0x3004_0000` – `0x3004_FFFF` | 64KB | AP | CLINT，本地中断控制器 |
| `0x3005_0000` – `0x3005_0FFF` | 4KB | AP | WDT (Core0) |
| `0x3005_1000` – `0x3005_1FFF` | 4KB | AP | WDT (Core1) |
| `0x3005_2000` – `0x3005_2FFF` | 4KB | AP | WDT (Core2) |
| `0x3005_3000` – `0x3005_3FFF` | 4KB | AP | WDT (Core3) |
| `0x3005_4000` – `0x3005_4FFF` | 4KB | AP | Timer (Core0) |
| `0x3005_5000` – `0x3005_5FFF` | 4KB | AP | Timer (Core1) |
| `0x3005_6000` – `0x3005_6FFF` | 4KB | AP | Timer (Core2) |
| `0x3005_7000` – `0x3005_7FFF` | 4KB | AP | Timer (Core3) |
| `0x3005_8000` – `0x3005_8FFF` | 4KB | AP | UART（AP Dedicated），主 Log 输出口；FSI 可跨域读取以监控 AP 状态 |
| `0x3005_9000` – `0x3005_9FFF` | 4KB | AP | Debug Module（AP），JTAG 调试接口，始终使能，详见「Debug」节 |
| `0x3005_A000` – `0x3FFF_FFFF` | ~256MB | — | Reserved |

### 区域 3: `0x4000_0000` – `0x4FFF_FFFF`（NPU 内存映射，待细化）

| 地址范围 | 容量 | 归属 | 说明 |
|----------|------|------|------|
| `0x4000_0000` – `0x4FFF_FFFF` | 256MB | NPU | NPU 子系统（详细映射待定）|

### DDR3 主存

| 地址范围 | 容量 | 归属 | 说明 |
|----------|------|------|------|
| `0xC000_0000` – `0xFFFF_FFFF` | 1GB | 全局 | DDR3 主存 |

> **AXI 隔离**：FSI/AP/NPU 专有域硬件硬编码隔离。FSI 作为安全核具有最高权限，可跨域读取 AP UART（日志监控）。
>
> **IPC_CTRL / SPINLOCK**（`0x000C_2000` / `0x000C_3000`）：所有主设备可读写，详见 [`ip/ipc.md`](ip/ipc.md)。
>
> **LSIO**（`0x0010_0000`–`0x001A_FFFF`）：QSPI0/1 + I2C0/1 等公共外设。EEPROM（24C64）挂 I2C0，存 MAC 地址、板级版本、校准数据等非安全持久化配置。AP 通过 PLIC 获取全部 LSIO 中断；FSI 本地中断仅侦听 QSPI0 和 I2C0，QSPI1/I2C1 仅 Poll。DMA 仅 AP 使用，FSI 只读寄存器。Ethernet MAC / SD/MMC 使用内部 DMA。
>
> **编址粒度**：FSI 密码协处理器（AES/SHA/RSA/TRNG，`0x1000_0000`–`0x1003_FFFF`）以 64KB 步进；FSI 简单 IP（WDT/UART/Timer/PMU，`0x1004_0000`–`0x100D_0FFF`）以 4KB 步进；AP 私密外设（`0x3005_0000`–`0x3005_9FFF`）以 4KB 步进；LSIO 外设（`0x0010_0000`–`0x001A_FFFF`）以 64KB 步进。

## 中断布局

### AP 本地中断 (CLINT)

AP CLINT 本地中断详见 [`ap/ap.md`](ap/ap.md)。

### AP 外部中断 (PLIC)

AP 使用 **SpinalHDL BmbPlic**（与 SpinalHDL 核同源生成、参数耦合），支持 M/S 双模式中断转发。
- **M 模式**：各核 MEIP 接收 OpenSBI（SEE）中断，用于异常转发与核间中断管理。
- **S 模式**：各核 SEIP 接收 OS（Linux）外部中断，经 OpenSBI 委托后直通 Linux。

AP PLIC 外部中断详见 [`ap/ap.md`](ap/ap.md)。

### FSI 本地中断

FSI 使用 **core-v-plic**（仅 M 模式），接收来自 QSPI0、I2C0、AP/NPU WDT 超时等外部中断。

FSI CLINT 本地中断与 PLIC 外部中断详见 [`fsi/fsi.md`](fsi/fsi.md)。

### NPU 本地中断

NPU 本地中断详见 [`npu/npu.md`](npu/npu.md)。

## 看门狗 (WDT)

各子系统均有独立 WDT。AP/NPU WDT 支持两阶段：首次超时发 PLIC 中断预警（AP 各核对应 PLIC #11–#14），最终超时向 FSI 发复位请求。FSI WDT 超时直接触发 SoC 全局复位（最后防线）。

| 触发源 | 动作 |
|--------|------|
| AP Core0–3 WDT | PLIC #11–#14 预警 → FSI 中断 #2–#5 → FSI 固件复位对应 Core 或整个 AP |
| NPU WDT | FSI 中断 #9 → FSI 固件复位 NPU |
| FSI WDT | SoC 全局复位，无需软件干预 |

> PMU 统一接管复位分发。FSI 通过 PMU 寄存器查询复位源及各域状态。详见「PMU」节。

## 调试 (Debug)

AP 和 FSI 均使用 VexRiscv 内置 RISC-V Debug Spec v0.13 Plugin（halt/step/resume、寄存器/内存访问）。JTAG DTM 通过 BSCANE2 复用 Xilinx JTAG 引脚。主机侧 OpenOCD + GDB。

| 域 | 使能条件 | JTAG 路径 |
|----|---------|----------|
| AP | **始终使能** | BSCANE2 → DTM → DM（独立 halt/step 4 核）|
| FSI | Debug/Test 模式使能；**Release JTAG 硬件断开** | 同上 JTAG 链路，PMU `PMU_DBG_CTRL.bit1` gates TAP 选择 |

> Release 模式 FSI JTAG 由 PMU 硬件强制禁用，仅 POR 可重新配置，软复位不变。

## PMU（电源管理与复位控制器）

`0x100D_0000`，仅 FSI 可写。负责复位分发、时钟门控、调试访问控制。

| 功能 | 说明 |
|------|------|
| **复位聚合** | POR / FSI WDT（全芯片复位）、Debug 请求 / SW 写寄存器（各域复位）、AP/NPU WDT → FSI → PMU（AP/NPU 域复位）|
| **复位输出** | `soc_rst_n`（全芯片）、`ap_rst_n`、`npu_rst_n`、`fsi_rst_n`（仅 POR 触发）|
| **时钟门控** | `PMU_CLK_GATE`：AP bit0、NPU bit1、Reserved bit2、Ethernet bit3。FSI 域不可门控 |
| **调试控制** | `PMU_DBG_CTRL`：bit0=AP JTAG（恒 1），bit1=FSI JTAG（Debug/Test=1，Release=0，仅 POR 可改）|

## PWM（脉宽调制器）

`0x001A_0000`，6 通道 16-bit，AP/FSI 均可配置。SiFive sifive-blocks `pwm` + AXI4-Lite 包裹，每通道独立使能/极性/占空比，无中断。典型用途：LED、背光、蜂鸣器、电机。

## I2C

`0x0012_0000`（I2C0）/ `0x0013_0000`（I2C1），AP/FSI 均可配置。SiFive sifive-blocks `i2c`，寄存器级定义详见 [`ip/sifive-i2c.md`](ip/sifive-i2c.md)。

- **I2C0**：挂载 **EEPROM 24C64**，存 MAC 地址、板级版本、校准数据等非安全持久化配置。中断接 AP PLIC #5 / FSI PLIC #8。
- **I2C1**：扩展 I2C 设备。中断接 AP PLIC #6（FSI 仅 Poll）。

## eFuse OTP

eFuse OTP 详见 [`fsi/fsi.md`](fsi/fsi.md)。

> 详细存储布局（Lifecycle、根密钥种子、启动参数、分区表等）见 [`boot.md`](../sw/boot.md)。

## 挂载 IP

### 处理器核

| IP | 归属 | 说明 |
|----|------|------|
| VexRiscv-SMP ×4 | AP | SpinalHDL，RV32IMAFDC，Sv32 MMU，M/S/U 特权态，缓存一致性 |
| Coral NPU | NPU | Google-Verisilicon，RV32IMF_Zve32x，标量+向量+矩阵 MAC；内置 WDT |
| VexRiscv RV32 | FSI | SpinalHDL，RV32IM，安全核 |

### FSI 外设

FSI 外设清单详见 [`fsi/fsi.md`](fsi/fsi.md)。

### AP 外设

AP 外设清单详见 [`ap/ap.md`](ap/ap.md)。

### 公共外设

| IP | 来源 | 地址 | 说明 |
|----|------|------|------|
| QSPI0 | SiFive sifive-blocks `spi`（QSPI 模式） | `0x0010_0000` | AP/FSI 读写；AP 中断使能，FSI 中断使能 |
| QSPI1 | SiFive sifive-blocks `spi`（QSPI 模式） | `0x0011_0000` | AP/FSI 读写；AP 中断使能，FSI 仅 Poll |
| I2C0 | SiFive sifive-blocks `i2c`（compatible `sifive,fu540-c000-i2c`） | `0x0012_0000` | AP/FSI 读写；AP 中断使能，FSI 中断使能；挂载 EEPROM 24C64 |
| I2C1 | SiFive sifive-blocks `i2c`（compatible `sifive,fu540-c000-i2c`） | `0x0013_0000` | AP/FSI 读写；AP 中断使能，FSI 仅 Poll |
| DMA | SiFive FU540 PDMA | `0x0020_0000` | AP 读写，FSI 只读；FSI 不得发起传输 |
| UART (扩展) | SiFive sifive-blocks `uart` | `0x0015_0000` | AP/FSI 读写；AP 中断使能 |
| GPIO | SiFive sifive-blocks `gpio` | `0x0016_0000` | AP/FSI 读写；无中断 |
| Ethernet MAC | OpenCores `ethmac`（保留，QEMU `opencores_eth` 模型现成） | `0x0017_0000` | AP/FSI 读写；内部 DMA，AP 中断使能 |
| SD/MMC | 标准 SDHCI | `0x0018_0000` | AP/FSI 读写；内部 DMA，AP 中断使能 |
| RTC | SiFive sifive-blocks `mockaon` RTC | `0x0019_0000` | AP/FSI 读写；AP 中断使能 |
| PWM | SiFive sifive-blocks `pwm` | `0x001A_0000` | AP/FSI 读写；无中断，6 通道脉宽调制器，16-bit 精度 |

### SoC 基础设施

| IP | 来源 | 说明 |
|----|------|------|
| AXI 互联 | OpenPULP `axi` | 多主多从总线 |
| DDR3 控制器 | Xilinx MIG | DDR3 接口 |
| PLIC（AP） | SpinalHDL `BmbPlic` | `0x3000_0000`，支持 M/S 模式，8 个上下文（每核 M+S），详见 ap.md |
| PLIC（FSI） | OpenHW core-v-plic | `0x1008_0000`，仅 M 模式，单目标上下文 |
| **IPC_CTRL** | **自研** | **`0x000C_2000`，核间通信控制器，含 Doorbell + IPI 状态 + 队列控制寄存器**，详见 [`ip/ipc.md`](ip/ipc.md) |
| **SPINLOCK** | **自研** | **`0x000C_3000`，硬件同步锁控制器，含 32 路 test-and-set 锁寄存器**，详见 [`ip/ipc.md`](ip/ipc.md) |

## 启动

| 阶段 | 硬件动作 |
|------|----------|
| 0 | FPGA 位流加载，固化 eFuse OTP 及 BootROM 至 `0x0000_0000`，DDR 初始化；仅 FSI 退出复位 |
| 1 | FSI (RV32) 从 `0x0000_1000`（BootROM 代码区）取指执行，自校验后加载 FSI 固件至 `0x000D_0000` |
| 2 | FSI 固件加载 NPU 至 `0x000B_8000`、AP 镜像至 DDR `0xC000_0000`，逐一验签 |
| 3 | FSI 释放 NPU / AP 复位信号 |

> 固件镜像来源介质（QSPI Flash 等）、验签算法及详细加载流程见 [`boot.md`](../sw/boot.md)。

## 复位向量

| 核 | 复位向量 | 说明 |
|----|----------|------|
| FSI (RV32) | `0x0000_1000` | BootROM 代码区入口（前 4KB 为 eFuse OTP）|
| NPU (RV32) | `0x000B_8000` | FSI 加载后跳转 |
| AP (RV32) | `0xC000_0000` | FSI 加载 OpenSBI 后跳转 |