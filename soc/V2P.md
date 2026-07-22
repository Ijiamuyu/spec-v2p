# V2P SoC 整体 SPEC

> 本文档描述 V2P SoC 的全局设计规范。

## 1. SoC 架构

V2P 是一个异构 RISC-V SoC，包含以下子系统：

```
 ┌─────────────────────────────────────────────────────────────┐
 │                         AP 子系统                            │
 │                   RV64 ×4 SMP / VexRiscv-SMP                │
 └───────────────────────────┬─────────────────────────────────┘
                             │
 ┌─────────────────────────────────────────────────────────────┐
 │                        AXI Interconnect                      │
 ├──────────┬───────────┬───────────┬────────────┬─────────────┤
 │   NPU    │    FSI    │   MBOX    │   LSIO     │   Memory    │
 │  RV32    │   RV32    │  核间通信  │  外设集     │             │
 │ Gemmini  │ 安全启动   │           │  SPI/I2C   │  DDR3 1GB   │
 │          │ AES/SHA   │           │  UART/DMA  │  SRAM 704KB │
 │          │ RSA/TRNG  │           │  ETH/SD    │             │
 └──────────┴───────────┴───────────┴────────────┴─────────────┘
```

所有模块通过 **AXI 互联** 连接。

### 子系统角色

| 子系统 | 处理器架构 | 角色 |
|--------|-----------|------|
| **AP** | RV64 ×4 SMP | 应用处理器，运行 Linux SMP / FreeRTOS SMP / RT-Thread SMP |
| **NPU** | RV32 | 神经网络推理加速，Gemmini 8×8 脉动阵列 |
| **FSI** | RV32 | 功能安全岛，信任根，主导安全启动 |
| **LSIO** | 无 | 低速外设集，由 AP/FSI 编程控制 |

## 2. 地址空间

32-bit 地址空间，DDR（1GB）置于顶部。

| 区域 | 地址范围 | 说明 |
|------|---------|------|
| DDR3 | `0xC000_0000 ~ 0xFFFF_FFFF` | 1GB 主存 |
| 核间控制 | `0x4000_0000 ~ 0x4001_0000` | CLINT / PLIC / MBOX |
| LSIO 外设 | `0x3000_0000 ~ 0x3010_0000` | SPI / QSPI / I2C×2 / UART / DMA / GPIO / PWM / ETH / SD / RTC / NPU-DBG / Timer×3 |
| FSI 外设 | `0x2000_0000 ~ 0x2006_0000` | AES / SHA / RSA / TRNG / UART |
| NPU 私有 SRAM | `0x000E_0000 ~ 0x0010_0000` | 128KB，FSI 预加载固件 |
| FSI 私有 SRAM | `0x000C_0000 ~ 0x000E_0000` | 128KB，含 BootROM + 固件 + 数据 |
| SYSTEM SRAM | `0x0001_0000 ~ 0x000C_0000` | 704KB 共享，核间通信与 MBOX 数据区 |
| EEPROM | `0x0000_2000 ~ 0x0000_4000` | 8KB I2C 设备，启动参数 |

> 详细外设寄存器地址见各子系统 SPEC。

### 复位向量

- **FSI (RV32)** → `0x000C_0000`（FSI 私有 SRAM，BootROM 入口）
- **NPU (RV32)** → `0x000E_0000`（NPU 私有 SRAM，FSI 预加载）
- **AP (RV64)** → `0xC000_0000`（DDR，FSI 预加载 OpenSBI）

## 3. 中断路由

| 中断源 | 中断号 | 目标 |
|--------|--------|------|
| LSIO UART | 42 | AP PLIC |
| LSIO SPI | 43 | AP PLIC |
| LSIO QSPI | 49 | AP PLIC |
| LSIO I2C 0 | 44 | AP PLIC |
| LSIO I2C 1 | 54 | AP PLIC |
| LSIO DMA | 45 | AP PLIC |
| LSIO Ethernet | 47 | AP PLIC |
| LSIO SD/MMC | 48 | AP PLIC |
| LSIO PWM | 50 | AP PLIC |
| LSIO Timer 0 | 51 | AP PLIC |
| LSIO Timer 1 | 52 | AP PLIC |
| LSIO Timer 2 | 53 | AP PLIC |
| NPU Debug UART | 46 | AP PLIC |
| AP Timer (CLINT mtimeirq, 每核独立) | - | AP 本地中断 |
| NPU Timer | - | NPU 本地中断 |
| FSI Timer | - | FSI 本地中断 |
| FSI UART | 10 | FSI 本地中断 |
| MBOX 通道 | - | AP / NPU / FSI |

MBOX 中断目标可配置，支持跨子系统门铃通知。

## 4. 互联与总线

- **系统总线**：AXI4 互联，支持多主多从
- **外设总线**：AXI4-Lite / APB 桥接
- **NPU 桥**：TileLink → AXI 桥，连接 Gemmini
- **QSPI**：支持 XIP（eXecute-In-Place），板载 N25Q256 Flash 直接映射地址空间

## 5. 存储层次

| 层次 | 容量 | 属性 |
|------|------|------|
| DDR3 | 1GB | 主存，所有主设备可访问 |
| SYSTEM SRAM | 704KB | 共享，核间通信与数据缓冲 |
| NPU 私有 SRAM | 128KB | 调度核专用 |
| FSI 私有 SRAM | 128KB | 安全域专用 |
| Gemmini Scratchpad | 40×36Kb | NPU 专用 |

## 6. 时钟与复位

SoC 提供单时钟域与全局复位：

- **主时钟**：由 FPGA 平台提供
- **复位**：POR 全局复位，各核独立复位控制
- **FSI 主导启动时序**：FSI 完成初始化后逐个释放其他核复位
- **Timer**：AP 每核独立 CLINT mtimeirq（4 路）+ NPU / FSI 各 1 路专属 Timer，作为操作系统时基与高性能计时；LSIO 集成 3 个通用 Timer 供 AP 应用层使用

## 7. 启动流程

| 阶段 | 执行体 | 动作 |
|------|--------|------|
| 0 | FPGA 位流 | DDR 初始化，预置 BootROM，其余核保持复位 |
| 1 | BootROM (FSI SRAM) | 加载 FSI 完整固件，SHA 自验后跳转 |
| 2 | FSI 完整固件 | 加载 NPU 固件、AP 启动镜像至 DDR，RSA 验签后释放各核 |
| 3 | 各子系统 | AP → Linux；NPU → FreeRTOS；FSI → 安全监控 |

