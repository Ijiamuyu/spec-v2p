# V2P SoC 硬件设计

## 内存布局

> 未列出的地址范围为 Reserved。访问 Reserved 区域将产生错误响应。

| 地址范围 | 容量 | 归属 | 说明 |
|----------|------|------|------|
| `0x0000_0000` – `0x0000_0FFF` | 4KB | FSI | eFuse OTP，位流固化，只读不可写，详见 §eFuse OTP |
| `0x0000_1000` – `0x0000_7FFF` | 28KB | FSI | BootROM 代码区，独立 BRAM，仅 FSI 可读 |
| `0x0000_8000` – `0x0000_FFFF` | 32KB | — | Reserved |
| `0x0001_0000` – `0x000B_7FFF` | 672KB | 全局 | SYSTEM SRAM，核间共享数据区，所有主设备可访问 |
| `0x000B_8000` – `0x000C_1FFF` | 40KB | NPU | NPU TCM（Coral ITCM 8KB + DTCM 32KB）|
| `0x000C_2000` – `0x000C_2FFF` | 4KB | 全局 | IPC_CTRL，核间通信控制器，所有主设备可访问，详见 [`IP/ipc.md`](IP/ipc.md) |
| `0x000C_3000` – `0x000C_3FFF` | 4KB | 全局 | **SPINLOCK**，硬件同步锁控制器，所有主设备可读写，详见 [`IP/ipc.md`](IP/ipc.md) |
| `0x000C_4000` – `0x000C_FFFF` | 48KB | — | Reserved |
| `0x000D_0000` – `0x000E_FFFF` | 128KB | FSI | FSI 私有 SRAM，固件加载目标 |
| `0x000F_0000` – `0x000F_FFFF` | 64KB | — | Reserved |
| `0x0010_0000` – `0x1FFF_FFFF` | ~512MB | — | Reserved |
| `0x2000_0000` – `0x2000_FFFF` | 64KB | FSI | AES-256 |
| `0x2001_0000` – `0x2001_FFFF` | 64KB | FSI | SHA-256 |
| `0x2002_0000` – `0x2002_FFFF` | 64KB | FSI | RSA-2048 |
| `0x2003_0000` – `0x2003_FFFF` | 64KB | FSI | TRNG |
| `0x2004_0000` – `0x2004_FFFF` | 64KB | FSI | WDT |
| `0x2005_0000` – `0x2005_FFFF` | 64KB | FSI | UART |
| `0x2006_0000` – `0x2006_FFFF` | 64KB | FSI | Timer |
| `0x2007_0000` – `0x2007_FFFF` | 64KB | FSI | PLIC |
| `0x2008_0000` – `0x2008_FFFF` | 64KB | FSI | CLINT |
| `0x2009_0000` – `0x2009_FFFF` | 64KB | FSI | PMU，电源管理与复位控制器，详见PMU |
| `0x200A_0000` – `0x2FFF_FFFF` | ~256MB | — | Reserved |
| `0x3000_0000` – `0x3000_FFFF` | 64KB | 全局 | QSPI0 |
| `0x3001_0000` – `0x3001_FFFF` | 64KB | 全局 | QSPI1 |
| `0x3002_0000` – `0x3002_FFFF` | 64KB | 全局 | I2C0 |
| `0x3003_0000` – `0x3003_FFFF` | 64KB | 全局 | I2C1 |
| `0x3004_0000` – `0x3004_FFFF` | 64KB | 全局 | DMA |
| `0x3005_0000` – `0x3005_FFFF` | 64KB | 全局 | UART（扩展）|
| `0x3006_0000` – `0x3006_FFFF` | 64KB | 全局 | GPIO |
| `0x3007_0000` – `0x3007_FFFF` | 64KB | 全局 | Ethernet MAC |
| `0x3008_0000` – `0x3008_FFFF` | 64KB | 全局 | SD/MMC |
| `0x3009_0000` – `0x3009_FFFF` | 64KB | 全局 | RTC |
| `0x300A_0000` – `0x300A_FFFF` | 64KB | 全局 | PWM，6 通道脉宽调制器，详见PWM |
| `0x300B_0000` – `0x3FFF_FFFF` | ~256MB | — | Reserved |
| `0x4000_0000` – `0x4000_FFFF` | 64KB | AP | PLIC |
| `0x4001_0000` – `0x4001_FFFF` | 64KB | AP | CLINT |
| `0x4002_0000` – `0x4002_0FFF` | 4KB | AP | WDT (Core0) |
| `0x4002_1000` – `0x4002_1FFF` | 4KB | AP | WDT (Core1) |
| `0x4002_2000` – `0x4002_2FFF` | 4KB | AP | WDT (Core2) |
| `0x4002_3000` – `0x4002_3FFF` | 4KB | AP | WDT (Core3) |
| `0x4002_4000` – `0x4002_4FFF` | 4KB | AP | Timer (Core0) |
| `0x4002_5000` – `0x4002_5FFF` | 4KB | AP | Timer (Core1) |
| `0x4002_6000` – `0x4002_6FFF` | 4KB | AP | Timer (Core2) |
| `0x4002_7000` – `0x4002_7FFF` | 4KB | AP | Timer (Core3) |
| `0x4002_8000` – `0x4002_8FFF` | 4KB | AP | UART（AP Dedicated）| AP 主 Log 输出口；FSI 可跨域读取以监控 AP 状态 |
| `0x4002_9000` – `0x4002_9FFF` | 4KB | AP | Debug Module（AP）| AP 调试接口，始终使能，详见 §Debug |
| `0x4002_A000` – `0xBFFF_FFFF` | ~2GB | — | Reserved |
| `0xC000_0000` – `0xFFFF_FFFF` | 1GB | 全局 | DDR3 主存 |

> **AXI 隔离**：FSI/AP/NPU 专有域硬件硬编码隔离。FSI 作为安全核具有最高权限，可跨域读取 AP UART（日志监控）。
>
> **IPC_CTRL / SPINLOCK**（`0x000C_2000` / `0x000C_3000`）：所有主设备可读写，详见 [`IP/ipc.md`](IP/ipc.md)。
>
> **LSIO**：QSPI0/1 + I2C0/1。EEPROM（24LC02）挂 I2C0，存 MAC 地址、板级版本、校准数据等非安全持久化配置。AP 通过 PLIC 获取全部 LSIO 中断；FSI 本地中断仅侦听 QSPI0 和 I2C0，QSPI1/I2C1 仅 Poll。DMA 仅 AP 使用，FSI 只读寄存器。Ethernet MAC / SD/MMC 使用内部 DMA。
>
> **编址粒度**：FSI 外设（`0x2000_0000`–`0x2009_FFFF`）和公共外设（`0x3000_0000`–`0x300A_FFFF`）以 64KB 步进；AP 私密外设（`0x4002_0000`–`0x4002_9FFF`）以 4KB 步进。

## 中断布局

### AP 本地中断 (CLINT)

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0–3 | 软件中断 | 每核 1 个，核间通信（AP 核间）|
| 4–7 | 定时器中断 | 每核 1 个，CLINT mtime 作操作系统时基 |

> AP 私有 Timer（OpenCores `timer`，PLIC #16–19）用于高精度计时及额外定时中断，与 CLINT mtime 互补。

### AP 外部中断 (PLIC)

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | — | Reserved（PLIC ID 0 保留）|
| 1 | AP UART | Log 输出 |
| 2 | LSIO UART | 扩展口 |
| 3 | LSIO QSPI0 | |
| 4 | LSIO QSPI1 | |
| 5 | LSIO I2C0 | |
| 6 | LSIO I2C1 | |
| 7 | LSIO DMA | |
| 8 | LSIO GPIO | 无中断，仅 Poll |
| 9 | Ethernet | |
| 10 | SD/MMC | |
| 11 | AP Core0 WDT | 每核独立 |
| 12 | AP Core1 WDT | 每核独立 |
| 13 | AP Core2 WDT | 每核独立 |
| 14 | AP Core3 WDT | 每核独立 |
| 15 | LSIO RTC | |
| 16 | AP Core0 Timer | 每核独立 |
| 17 | AP Core1 Timer | 每核独立 |
| 18 | AP Core2 Timer | 每核独立 |
| 19 | AP Core3 Timer | 每核独立 |
| 20 | — | Reserved |
| 21 | Doorbell DB_NPU_AP0 | NPU→AP0 |
| 22 | Doorbell DB_NPU_AP1 | NPU→AP1 |
| 23 | Doorbell DB_NPU_AP2 | NPU→AP2 |
| 24 | Doorbell DB_NPU_AP3 | NPU→AP3 |
| 25 | Doorbell DB_FSI_AP0 | FSI→AP0 |
| 26 | Doorbell DB_FSI_AP1 | FSI→AP1 |
| 27 | Doorbell DB_FSI_AP2 | FSI→AP2 |
| 28 | Doorbell DB_FSI_AP3 | FSI→AP3 |

### FSI 本地中断

FSI 采用 CLINT + PLIC 双中断控制器架构：

- **CLINT**（`0x2008_0000`）：提供 1 路本地软件中断（MSIP）和 1 路本地定时器中断（mtime）
- **PLIC**（`0x2007_0000`）：将外部设备中断集中路由至 FSI

#### CLINT 本地中断

| 中断类型 | 源 | 说明 |
|----------|-----|------|
| 软件中断 (MSIP) | — | 保留未使用，核间通信走 PLIC #10/#11 |
| 定时器中断 (mtime) | — | OS 时基；PLIC #1 提供独立高精度计时 |

#### PLIC 外部中断

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | — | Reserved（PLIC ID 0 保留）|
| 1 | FSI Timer | 高精度计时 |
| 2 | AP Core0 WDT 复位 | — |
| 3 | AP Core1 WDT 复位 | — |
| 4 | AP Core2 WDT 复位 | — |
| 5 | AP Core3 WDT 复位 | — |
| 6 | FSI UART | 控制台 |
| 7 | LSIO QSPI | QSPI0 中断，QSPI1 仅 Poll |
| 8 | LSIO I2C | I2C0 中断（EEPROM），I2C1 仅 Poll |
| 9 | NPU WDT 复位 | — |
| 10 | Doorbell DB_AP_FSI | AP→FSI |
| 11 | Doorbell DB_NPU_FSI | NPU→FSI |
| 12 | Doorbell DB_ERR | Doorbell 异常 |

> AES / SHA / RSA / TRNG 纯 MMIO 轮询，无中断需求。

### NPU 本地中断

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | NPU Timer | Coral 内部定时器 |
| 1 | Doorbell DB_APx_NPU（线或） | IPC_CTRL AP→NPU 核间通信，4 路 AP Core 门铃（DB_AP0_NPU–DB_AP3_NPU）线或共享此中断 |
| 2 | Doorbell DB_FSI_NPU | IPC_CTRL FSI→NPU 核间通信 |
| 3 | Reserved | — |

## 看门狗 (WDT)

各子系统均有独立 WDT。AP/NPU WDT 支持两阶段：首次超时发 PLIC 中断预警（AP 各核对应 PLIC #11–#14），最终超时向 FSI 发复位请求。FSI WDT 超时直接触发 SoC 全局复位（最后防线）。

| 触发源 | 动作 |
|--------|------|
| AP Core0–3 WDT | PLIC #11–#14 预警 → FSI 中断 #2–#5 → FSI 固件复位对应 Core 或整个 AP |
| NPU WDT | FSI 中断 #9 → FSI 固件复位 NPU |
| FSI WDT | SoC 全局复位，无需软件干预 |

> PMU 统一接管复位分发。FSI 通过 PMU 寄存器查询复位源及各域状态。详见 §PMU。

## 调试 (Debug)

AP 和 FSI 均使用 VexRiscv 内置 RISC-V Debug Spec v0.13 Plugin（halt/step/resume、寄存器/内存访问）。JTAG DTM 通过 BSCANE2 复用 Xilinx JTAG 引脚。主机侧 OpenOCD + GDB。

| 域 | 使能条件 | JTAG 路径 |
|----|---------|----------|
| AP | **始终使能** | BSCANE2 → DTM → DM（独立 halt/step 4 核）|
| FSI | Debug/Test 模式使能；**Release JTAG 硬件断开** | 同上 JTAG 链路，PMU `PMU_DBG_CTRL.bit1` gates TAP 选择 |

> Release 模式 FSI JTAG 由 PMU 硬件强制禁用，仅 POR 可重新配置，软复位不变。

## PMU（电源管理与复位控制器）

`0x2009_0000`，仅 FSI 可写。负责复位分发、时钟门控、调试访问控制。

| 功能 | 说明 |
|------|------|
| **复位聚合** | POR / FSI WDT（全芯片复位）、Debug 请求 / SW 写寄存器（各域复位）、AP/NPU WDT → FSI → PMU（AP/NPU 域复位）|
| **复位输出** | `soc_rst_n`（全芯片）、`ap_rst_n`、`npu_rst_n`、`fsi_rst_n`（仅 POR 触发）|
| **时钟门控** | `PMU_CLK_GATE`：AP bit0、NPU bit1、Reserved bit2、Ethernet bit3。FSI 域不可门控 |
| **调试控制** | `PMU_DBG_CTRL`：bit0=AP JTAG（恒 1），bit1=FSI JTAG（Debug/Test=1，Release=0，仅 POR 可改）|

## PWM（脉宽调制器）

`0x300A_0000`，6 通道 16-bit，AP/FSI 均可配置。OpenCores `pwm` + AXI-Lite 包裹，每通道独立使能/极性/占空比，无中断。典型用途：LED、背光、蜂鸣器、电机。

## eFuse OTP

`0x0000_0000` – `0x0000_0FFF`，BootROM 独立 BRAM 的前 4KB，作为 FPGA 阶段 eFuse 仿真。

| 项目 | 说明 |
|------|------|
| 实现方式 | 独立 BRAM 前 4KB，位流编译时写入初始值 |
| 写操作 | 硬件忽略（返回错误响应），模拟 eFuse 不可编程特性 |
| 读操作 | 正常返回，仅 FSI 可读 |
| 复位 | BRAM 内容在 FPGA 配置期内保持 |
| ASIC 替换 | 替换为工艺 eFuse macro，数字接口不变 |

> 详细存储布局（Lifecycle、根密钥种子、启动参数、分区表等）见 [`boot.md`](../sw/boot.md)。

## 挂载 IP

### 处理器核

| IP | 归属 | 说明 |
|----|------|------|
| VexRiscv-SMP ×4 | AP | SpinalHDL，RV64IMAFDC，Sv39 MMU，缓存一致性 |
| Coral NPU | NPU | Google-Verisilicon，RV32IMF_Zve32x，标量+向量+矩阵 MAC；内置 WDT |
| VexRiscv RV32 | FSI | SpinalHDL，RV32IM，安全核 |

### FSI 外设

| IP | 来源 | 地址 | 说明 |
|----|------|------|------|
| AES-256 | OpenCores `aes_core` | `0x2000_0000` | |
| SHA-256 | OpenCores `sha256` | `0x2001_0000` | |
| RSA-2048 | OpenCores `basicrsa` | `0x2002_0000` | |
| TRNG | OpenTitan `entropy_src` | `0x2003_0000` | |
| WDT | OpenCores `watch_dog` | `0x2004_0000` | |
| UART | OpenCores `uart16550` | `0x2005_0000` | |
| Timer (FSI) | OpenCores `timer` | `0x2006_0000` | |
| PLIC (FSI) | OpenPULP | `0x2007_0000` | |
| CLINT (FSI) | OpenPULP | `0x2008_0000` | |
| PMU | 自研 | `0x2009_0000` | 电源管理与复位控制器，仅 FSI 可写；含复位管理、时钟门控、调试访问控制 |

### AP 外设

| IP | 来源 | 地址 | 说明 |
|----|------|------|------|
| PLIC | OpenPULP | `0x4000_0000` | |
| CLINT | OpenPULP | `0x4001_0000` | |
| WDT (Core0) | OpenCores `watch_dog` | `0x4002_0000` | |
| WDT (Core1) | OpenCores `watch_dog` | `0x4002_1000` | |
| WDT (Core2) | OpenCores `watch_dog` | `0x4002_2000` | |
| WDT (Core3) | OpenCores `watch_dog` | `0x4002_3000` | |
| Timer (Core0) | OpenCores `timer` | `0x4002_4000` | |
| Timer (Core1) | OpenCores `timer` | `0x4002_5000` | |
| Timer (Core2) | OpenCores `timer` | `0x4002_6000` | |
| Timer (Core3) | OpenCores `timer` | `0x4002_7000` | |
| UART (AP Dedicated) | OpenCores `uart16550` | `0x4002_8000` | AP 主 Log 输出口；FSI 可跨域读取以监控 AP 状态 |
| Debug Module (AP) | VexRiscv-SMP 内置 | `0x4002_9000` | JTAG 调试接口，始终使能 |

### 公共外设

| IP | 来源 | 地址 | 说明 |
|----|------|------|------|
| QSPI0 | OpenPULP `spi`（QSPI 模式） | `0x3000_0000` | |
| QSPI1 | OpenPULP `spi`（QSPI 模式） | `0x3001_0000` | |
| I2C0 | OpenCores `i2c_master_top` | `0x3002_0000` | |
| I2C1 | OpenCores `i2c_master_top` | `0x3003_0000` | |
| DMA | OpenCores `wb_dma` | `0x3004_0000` | FSI 可读写寄存器，不得发起传输 |
| UART (扩展) | OpenCores `uart16550` | `0x3005_0000` | 通用扩展串口，供 AP 或外部设备使用 |
| GPIO | OpenCores `gpio` | `0x3006_0000` | |
| Ethernet MAC | OpenCores `ethmac` | `0x3007_0000` | |
| SD/MMC | OpenCores `sd_controller` | `0x3008_0000` | |
| RTC | OpenCores `rtc` | `0x3009_0000` | |
| PWM | OpenCores `pwm` | `0x300A_0000` | 6 通道脉宽调制器，16-bit 精度 |

### SoC 基础设施

| IP | 来源 | 说明 |
|----|------|------|
| AXI 互联 | OpenPULP `axi` | 多主多从总线 |
| DDR3 控制器 | Xilinx MIG | DDR3 接口 |
| **IPC_CTRL** | **自研** | **`0x000C_2000`，核间通信控制器，含 Doorbell + IPI 状态 + 队列控制寄存器**，详见 [`IP/ipc.md`](IP/ipc.md) |
| **SPINLOCK** | **自研** | **`0x000C_3000`，硬件同步锁控制器，含 32 路 test-and-set 锁寄存器**，详见 [`IP/ipc.md`](IP/ipc.md) |

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
| AP (RV64) | `0xC000_0000` | FSI 加载 OpenSBI 后跳转 |