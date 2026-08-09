# FSI (Firmware Security Island) 子系统设计

## 概述

FSI 是 V2P SoC 的安全核子系统，包含 1 个 VexRiscv RV32 处理器及配套的密码学加速器、定时器、看门狗、中断控制器等私密外设。FSI 作为 SoC 最高权限域，负责安全启动、固件验签、电源管理、复位分发、调试访问控制及跨域监控。FSI 通过 AXI 互联访问公共外设（LSIO 部分外设）和共享系统资源，与 AP 及 NPU 之间通过 IPC_CTRL / SPINLOCK 进行核间通信。

## 处理器核

| 项目 | 说明 |
|------|------|
| IP | VexRiscv RV32（SpinalHDL）|
| 架构 | RV32IM |
| 角色 | 安全核，SoC 最高权限域 |
| 调试 | 内置 RISC-V Debug Spec v0.13 Plugin；Debug/Test 模式使能，Release 模式 JTAG 硬件断开 |

## 内存布局

### FSI 私密外设

| 地址范围 | 容量 | 说明 |
|----------|------|------|
| `0x0000_0000` – `0x0000_0FFF` | 4KB | eFuse OTP，位流固化，只读不可写，详见 §eFuse OTP |
| `0x0000_1000` – `0x0000_7FFF` | 28KB | BootROM 代码区，独立 BRAM，仅 FSI 可读 |
| `0x000D_0000` – `0x000E_FFFF` | 128KB | FSI 私有 SRAM，固件加载目标 |
| `0x1000_0000` – `0x1000_FFFF` | 64KB | AES-256 |
| `0x1001_0000` – `0x1001_FFFF` | 64KB | SHA-256 |
| `0x1002_0000` – `0x1002_FFFF` | 64KB | RSA-2048 |
| `0x1003_0000` – `0x1003_FFFF` | 64KB | TRNG |
| `0x1004_0000` – `0x1004_0FFF` | 4KB | WDT |
| `0x1004_1000` – `0x1004_1FFF` | 4KB | UART |
| `0x1004_2000` – `0x1004_2FFF` | 4KB | Timer |
| `0x1008_0000` – `0x100B_FFFF` | 256KB | PLIC |
| `0x100C_0000` – `0x100C_FFFF` | 64KB | CLINT |
| `0x100D_0000` – `0x100D_0FFF` | 4KB | PMU，电源管理与复位控制器 |

### FSI 可访问的共享资源

| 地址范围 | 容量 | 说明 |
|----------|------|------|
| `0x0001_0000` – `0x000B_7FFF` | 672KB | SYSTEM SRAM，核间共享数据区 |
| `0x000C_2000` – `0x000C_2FFF` | 4KB | IPC_CTRL，核间通信控制器（Doorbell + IPI）|
| `0x000C_3000` – `0x000C_3FFF` | 4KB | SPINLOCK，硬件同步锁（32 路 test-and-set）|
| `0x3005_8000` – `0x3005_8FFF` | 4KB | AP UART（跨域读取，监控 AP 状态）|
| `0x0010_0000` – `0x0010_FFFF` | 64KB | QSPI0（中断支持）|
| `0x0011_0000` – `0x0011_FFFF` | 64KB | QSPI1（仅 Poll）|
| `0x0012_0000` – `0x0012_FFFF` | 64KB | I2C0（中断支持，挂载 EEPROM 24LC02）|
| `0x0013_0000` – `0x0013_FFFF` | 64KB | I2C1（仅 Poll）|
| `0x0014_0000` – `0x0014_FFFF` | 64KB | Reserved（DMA 已移至 `0x0020_0000`）|
| `0x0015_0000` – `0x0015_FFFF` | 64KB | UART（扩展，AP/FSI 读写；AP 中断使能）|
| `0x0016_0000` – `0x0016_FFFF` | 64KB | GPIO（AP/FSI 读写；无中断）|
| `0x0017_0000` – `0x0017_FFFF` | 64KB | Ethernet MAC（AP/FSI 读写；内部 DMA，AP 中断使能）|
| `0x0018_0000` – `0x0018_FFFF` | 64KB | SD/MMC（AP/FSI 读写；内部 DMA，AP 中断使能）|
| `0x0019_0000` – `0x0019_FFFF` | 64KB | RTC（AP/FSI 读写；AP 中断使能）|
| `0x001A_0000` – `0x001A_FFFF` | 64KB | PWM（AP/FSI 读写；无中断，6 通道）|
| `0x0020_0000` – `0x002F_FFFF` | 1MB | DMA（仅只读寄存器，1MB 对齐）|
| `0xC000_0000` – `0xFFFF_FFFF` | 1GB | DDR3 主存 |

> **AXI 隔离**：FSI 作为安全核具有最高权限，可跨域读取 AP UART（日志监控）。FSI 不可发起 DMA 传输（DMA 寄存器只读）。
>
> **LSIO 访问策略**（`0x0010_0000`–`0x001A_FFFF`）：FSI 本地中断仅侦听 QSPI0 和 I2C0，QSPI1/I2C1 仅 Poll。详见 [`../V2P.md`](../V2P.md) 内存布局中的 LSIO 说明。

## 中断布局

FSI 采用 CLINT + PLIC 双中断控制器架构：

- **CLINT**（`0x100C_0000`）：提供 1 路本地软件中断（MSIP）和 1 路本地定时器中断（mtime）
- **PLIC**（`0x1008_0000`）：将外部设备中断集中路由至 FSI

### CLINT 本地中断

| 中断类型 | 源 | 说明 |
|----------|-----|------|
| 软件中断 (MSIP) | — | 保留未使用，核间通信走 PLIC #10/#11 |
| 定时器中断 (mtime) | — | OS 时基；PLIC #1 提供独立高精度计时 |

### PLIC 外部中断

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | — | Reserved（PLIC ID 0 保留）|
| 1 | FSI Timer | 高精度计时 |
| 2 | AP Core0 WDT 复位 | AP Core0 看门狗最终超时通知 |
| 3 | AP Core1 WDT 复位 | AP Core1 看门狗最终超时通知 |
| 4 | AP Core2 WDT 复位 | AP Core2 看门狗最终超时通知 |
| 5 | AP Core3 WDT 复位 | AP Core3 看门狗最终超时通知 |
| 6 | FSI UART | 控制台 |
| 7 | LSIO QSPI | QSPI0 中断，QSPI1 仅 Poll |
| 8 | LSIO I2C | I2C0 中断（EEPROM），I2C1 仅 Poll |
| 9 | NPU WDT 复位 | NPU 看门狗最终超时通知 |
| 10 | Doorbell DB_AP_FSI | AP→FSI 核间通信 |
| 11 | Doorbell DB_NPU_FSI | NPU→FSI 核间通信 |
| 12 | Doorbell DB_ERR | Doorbell 异常 |

> AES / SHA / RSA / TRNG 纯 MMIO 轮询，无中断需求。

## 看门狗 (WDT)

FSI WDT（SiFive sifive-blocks `wdt`，OpenCores `watch_dog` 同源，`0x1004_0000`）超时直接触发 SoC 全局复位，无需软件干预，是系统的最后防线。

| 项目 | 说明 |
|------|------|
| 超时动作 | 直接触发 SoC 全局复位（`soc_rst_n`）|
| 复位路径 | FSI WDT → PMU → SoC 全局复位 |
| 软件干预 | 无需软件干预，硬件自动完成 |

## 调试 (Debug)

FSI 使用 VexRiscv 内置 RISC-V Debug Spec v0.13 Plugin（halt/step/resume、寄存器/内存访问）。JTAG DTM 通过 BSCANE2 复用 Xilinx JTAG 引脚。

| 项目 | 说明 |
|------|------|
| 使能条件 | Debug/Test 模式使能；**Release 模式 JTAG 硬件断开** |
| JTAG 路径 | BSCANE2（复用 Xilinx JTAG 引脚）→ DTM → DM |
| 访问控制 | PMU `PMU_DBG_CTRL.bit1` gates TAP 选择 |
| 约束 | Release 模式由 PMU 硬件强制禁用，仅 POR 可重新配置，软复位不变 |
| 主机侧 | OpenOCD + GDB |

## eFuse OTP

`0x0000_0000` – `0x0000_0FFF`，BootROM 独立 BRAM 的前 4KB，作为 FPGA 阶段 eFuse 仿真。

| 项目 | 说明 |
|------|------|
| 实现方式 | 独立 BRAM 前 4KB，位流编译时写入初始值 |
| 写操作 | 硬件忽略（返回错误响应），模拟 eFuse 不可编程特性 |
| 读操作 | 正常返回，仅 FSI 可读 |
| 复位 | BRAM 内容在 FPGA 配置期内保持 |
| ASIC 替换 | 替换为工艺 eFuse macro，数字接口不变 |

> 详细存储布局（Lifecycle、根密钥种子、启动参数、分区表等）见 [`boot.md`](../../sw/boot.md)。

## PMU（电源管理与复位控制器）

`0x100D_0000`，仅 FSI 可写。FSI 通过 PMU 统一管理 SoC 复位分发、时钟门控及调试访问控制。

| 功能 | 说明 |
|------|------|
| **复位聚合** | POR / FSI WDT（全芯片复位）、Debug 请求 / SW 写寄存器（各域复位）、AP/NPU WDT → FSI → PMU（AP/NPU 域复位）|
| **复位输出** | `soc_rst_n`（全芯片）、`ap_rst_n`、`npu_rst_n`、`fsi_rst_n`（仅 POR 触发）|
| **时钟门控** | `PMU_CLK_GATE`：AP bit0、NPU bit1、Reserved bit2、Ethernet bit3。FSI 域不可门控 |
| **调试控制** | `PMU_DBG_CTRL`：bit0=AP JTAG（恒 1），bit1=FSI JTAG（Debug/Test=1，Release=0，仅 POR 可改）|

## 外设清单

| IP | 来源 | 地址 | 说明 |
|----|------|------|------|
| AES-256 | OpenCores `aes_core` | `0x1000_0000` | MMIO 轮询，无中断 |
| SHA-256 | OpenCores `sha256` | `0x1001_0000` | MMIO 轮询，无中断 |
| RSA-2048 | OpenCores `basicrsa` | `0x1002_0000` | MMIO 轮询，无中断 |
| TRNG | OpenTitan `entropy_src` | `0x1003_0000` | MMIO 轮询，无中断 |
| WDT | SiFive sifive-blocks `wdt` | `0x1004_0000` | SoC 全局复位最后防线 |
| UART | SiFive sifive-blocks `uart` | `0x1004_1000` | FSI 控制台 |
| Timer (FSI) | OpenCores `timer`（保留） | `0x1004_2000` | 高精度计时，PLIC #1 |
| PLIC (FSI) | OpenHW core-v-plic | `0x1008_0000` | 平台级中断控制器，256KB |
| CLINT (FSI) | OpenPULP | `0x100C_0000` | 核级本地中断控制器 |
| PMU | 自研 | `0x100D_0000` | 电源管理与复位控制器，仅 FSI 可写 |

## 复位与启动

### 复位向量

| 项目 | 说明 |
|------|------|
| 复位地址 | `0x0000_1000`（BootROM 代码区入口，前 4KB 为 eFuse OTP）|
| 引导流程 | FSI 从 BootROM 取指执行，自校验后加载 FSI 固件 |
| 复位信号 | `fsi_rst_n`，仅 POR 触发 |

### 启动流程

FSI 承担 SoC 早期启动的全部职责：

1. **阶段 0**：FPGA 位流加载，固化 eFuse OTP 及 BootROM 至 `0x0000_0000`，DDR 初始化；仅 FSI 退出复位
2. **阶段 1**：FSI (RV32) 从 `0x0000_1000`（BootROM 代码区）取指执行，自校验后加载 FSI 固件至 `0x000D_0000`
3. **阶段 2**：FSI 固件加载 NPU 至 `0x000B_8000`、AP 镜像至 DDR `0xC000_0000`，逐一验签
4. **阶段 3**：FSI 释放 NPU / AP 复位信号

> 固件镜像来源介质（QSPI Flash 等）、验签算法及详细加载流程见 [`boot.md`](../../sw/boot.md)。

### 复位源

FSI 域复位仅可由 POR 触发。FSI 自身不可被其他域复位。

| 复位源 | 说明 |
|--------|------|
| POR | 上电复位，唯一可复位 FSI 域的事件 |
| FSI WDT | 触发 SoC 全局复位，FSI 域随之复位 |

## 核间通信

FSI 通过以下机制与 AP、NPU 通信：

- **IPC_CTRL**（`0x000C_2000`）：含 Doorbell 寄存器，支持 FSI→AP（AP PLIC #25–#28）、FSI→NPU（NPU 中断 #2）、AP→FSI（FSI PLIC #10）、NPU→FSI（FSI PLIC #11）的双向通知
- **SPINLOCK**（`0x000C_3000`）：32 路硬件 test-and-set 锁，用于共享数据结构的互斥访问
- **SYSTEM SRAM**（`0x0001_0000`–`0x000B_7FFF`）：核间共享数据区，配合 SPINLOCK 实现核间通信协议

详见 [`../ip/ipc.md`](../ip/ipc.md)。

## 时钟与电源

| 项目 | 说明 |
|------|------|
| 时钟门控 | FSI 域不可门控（`PMU_CLK_GATE` 无 FSI 位）|
| 复位输出 | PMU 管理，FSI 固件通过 PMU 寄存器控制其他域复位 |
| 调试控制 | PMU `PMU_DBG_CTRL.bit1` 控制 FSI JTAG 使能（Debug/Test=1，Release=0，仅 POR 可改）|
