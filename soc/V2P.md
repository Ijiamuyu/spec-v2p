# V2P SoC 硬件设计

## 内存布局

> 未列出的地址范围为 Reserved。访问 Reserved 区域将产生错误响应。

| 地址范围 | 容量 | 归属 | 说明 |
|----------|------|------|------|
| `0x0000_0000` – `0x0000_7FFF` | 32KB | FSI | BootROM，独立 BRAM，位流固化，仅FSI 可读 |
| `0x0000_8000` – `0x0000_FFFF` | 32KB | — | Reserved（预留 BootROM 扩展）|
| `0x0001_0000` – `0x000B_7FFF` | 672KB | 全局 | SYSTEM SRAM，核间共享数据区，所有主设备可访问 |
| `0x000B_8000` – `0x000C_1FFF` | 40KB | NPU | NPU TCM（Coral ITCM 8KB + DTCM 32KB）|
| `0x000C_2000` – `0x000C_FFFF` | 56KB | — | Reserved |
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
| `0x2009_0000` – `0x2FFF_FFFF` | ~256MB | — | Reserved |
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
| `0x300A_0000` – `0x3FFF_FFFF` | ~256MB | — | Reserved |
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
| `0x4002_8000` – `0x4002_8FFF` | 4KB | AP | UART（AP Dedicated）|
| `0x4002_9000` – `0xBFFF_FFFF` | ~2GB | — | Reserved |
| `0xC000_0000` – `0xFFFF_FFFF` | 1GB | 全局 | DDR3 主存 |

> FSI / AP / NPU 专有区域由 AXI 访问控制硬编码隔离；公共外设 AP 与 FSI 均可访问。FSI 作为安全核具有最高 AXI 权限，可跨越隔离读写 AP 专有区 UART（用于 Log 监控及安全审计）。NPU 对外设访问由内部实现决定，暂不涉及 SoC 层控制。
>
> LSIO 上挂载 2 路 QSPI（QSPI0/1）和 2 路 I2C（I2C0/1）。EEPROM 挂载在 I2C0，存放启动参数。AP 可通过 PLIC 获取全部 LSIO 中断；FSI 通过本地中断侦听 QSPI0 和 I2C0，QSPI1 和 I2C1 仅可 Poll 模式访问。
>
> LSIO DMA 控制器仅 AP 用于数据传输（包括 QSPI0/1、I2C0/1）。Ethernet MAC 和 SD/MMC 使用自身内部 DMA。FSI 可读写 LSIO DMA 配置及状态寄存器，但不得发起 DMA 传输。
>
> 编址粒度说明：FSI 外设（`0x2000_0000`–`0x2008_FFFF`）和公共外设（`0x3000_0000`–`0x3009_FFFF`）以 64KB 为步进分配地址空间，AP 私密外设（`0x4002_0000`–`0x4002_8FFF`）以 4KB 为步进紧凑排布。此差异由各 AXI 从口地址解码的地址掩码配置决定。

## 中断布局

### AP 本地中断 (CLINT)

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0–3 | 软件中断 | 每核 1 个，核间通信（AP 核间）|
| 4–7 | 定时器中断 | 每核 1 个，CLINT mtime 作操作系统时基 |

> AP 私有 Timer（OpenCores `timer`，PLIC #15–18）用于高精度计时及额外定时中断，与 CLINT mtime 互补。

### AP 外部中断 (PLIC)

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | AP UART | Log 输出 |
| 1 | LSIO UART | 扩展口 |
| 2 | LSIO QSPI0 | |
| 3 | LSIO QSPI1 | |
| 4 | LSIO I2C0 | |
| 5 | LSIO I2C1 | |
| 6 | LSIO DMA | |
| 7 | LSIO GPIO | |
| 8 | Ethernet | |
| 9 | SD/MMC | |
| 10 | AP Core0 WDT | 每核独立 |
| 11 | AP Core1 WDT | 每核独立 |
| 12 | AP Core2 WDT | 每核独立 |
| 13 | AP Core3 WDT | 每核独立 |
| 14 | LSIO RTC | |
| 15 | AP Core0 Timer | 每核独立 |
| 16 | AP Core1 Timer | 每核独立 |
| 17 | AP Core2 Timer | 每核独立 |
| 18 | AP Core3 Timer | 每核独立 |
| 19 | FSI 软件中断 | FSI → AP 核间通信 |
| 20 | NPU 软件中断 | NPU → AP 核间通信 |

### FSI 本地中断

FSI 采用 CLINT + PLIC 双中断控制器架构：

- **CLINT**（`0x2008_0000`）：提供 1 路本地软件中断（MSIP）和 1 路本地定时器中断（mtime）
- **PLIC**（`0x2007_0000`）：将外部设备中断集中路由至 FSI

#### CLINT 本地中断

| 中断类型 | 源 | 说明 |
|----------|-----|------|
| 软件中断 (MSIP) | AP / NPU | CLINT 仅提供 1 路 MSIP 寄存器；多核间软件中断（AP→FSI、NPU→FSI）经 PLIC #9/#10 路由 |
| 定时器中断 (mtime) | — | 作为 FSI 操作系统时基；OpenCores `timer`（`0x2006_0000`）经 PLIC #0 提供独立高精度计时 |

#### PLIC 外部中断

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | FSI Timer | OpenCores `timer`（`0x2006_0000`），高精度计时 |
| 1 | AP Core0 WDT 复位 | AP Core0 WDT 最终超时触发 FSI 复位请求 |
| 2 | AP Core1 WDT 复位 | AP Core1 WDT 最终超时触发 FSI 复位请求 |
| 3 | AP Core2 WDT 复位 | AP Core2 WDT 最终超时触发 FSI 复位请求 |
| 4 | AP Core3 WDT 复位 | AP Core3 WDT 最终超时触发 FSI 复位请求 |
| 5 | FSI UART | FSI 控制台 |
| 6 | LSIO QSPI | LSIO QSPI0 中断，QSPI1 仅 Poll |
| 7 | LSIO I2C | LSIO I2C0 中断（EEPROM 所在），I2C1 仅 Poll |
| 8 | NPU WDT 复位 | NPU 内置 WDT 最终超时，触发 FSI 中断 |
| 9 | AP 软件中断 | AP → FSI 核间通信（经 PLIC 路由） |
| 10 | NPU 软件中断 | NPU → FSI 核间通信（经 PLIC 路由） |

> AES / SHA / RSA / TRNG 纯 MMIO 轮询，无中断需求。

### NPU 本地中断

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | NPU Timer | Coral 内部定时器 |
| 1 | AP 软件中断 | AP → NPU 核间通信 |
| 2 | FSI 软件中断 | FSI → NPU 核间通信 |

## 看门狗 (WDT)

### 设计原则

- 每个子系统（AP 各核、NPU、FSI）均有独立 WDT
- AP / NPU WDT 支持两阶段超时：首次超时向本级发 PLIC 中断预警（AP 各核对应 PLIC #10–#13，可配置使能）；最终超时后向 FSI 发送复位请求
- FSI 在中断处理程序中判定复位来源，通过软件可控方式复位目标子系统
- FSI 自身 WDT 超时后，直接触发 **SoC 级全局复位**（最后防线）

### 复位流程

| 触发源 | 硬件动作 |
|--------|----------|
| AP Core0–3 WDT 最终超时 | 首次超时经 PLIC #10–#13 向对应 AP 核发中断预警（AP 可喂狗恢复）；最终超时后向 FSI 发出复位请求 → 触发 FSI 中断 #1–#4 → FSI 固件判定并复位对应 Core 或整个 AP |
| NPU WDT 超时 | 向 FSI 发出复位请求 → 触发 FSI 中断 #8 → FSI 固件判定并复位 NPU |
| FSI WDT 超时 | 直接触发 SoC 级全局复位，无需软件干预 |

## 挂载 IP

### 处理器核

| IP | 归属 | 说明 |
|----|------|------|
| VexRiscv-SMP ×4 | AP | SpinalHDL，RV64IMAFDC，Sv39 MMU，缓存一致性 |
| Coral NPU | NPU | Google-Verisilicon，RV32IMF_Zve32x，标量+向量+矩阵 MAC；内置 WDT |
| VexRiscv RV32 | FSI | SpinalHDL，RV32IM，安全核 |

### FSI 外设

| IP | 来源 | 地址 |
|----|------|------|
| AES-256 | OpenCores `aes_core` | `0x2000_0000` |
| SHA-256 | OpenCores `sha256` | `0x2001_0000` |
| RSA-2048 | OpenCores `basicrsa` | `0x2002_0000` |
| TRNG | OpenTitan `entropy_src` | `0x2003_0000` |
| WDT | OpenCores `watch_dog` | `0x2004_0000` |
| UART | OpenCores `uart16550` | `0x2005_0000` |
| Timer (FSI) | OpenCores `timer` | `0x2006_0000` |
| PLIC (FSI) | OpenPULP | `0x2007_0000` |
| CLINT (FSI) | OpenPULP | `0x2008_0000` |

### AP 外设

| IP | 来源 | 地址 |
|----|------|------|
| PLIC | OpenPULP | `0x4000_0000` |
| CLINT | OpenPULP | `0x4001_0000` |
| WDT (Core0) | OpenCores `watch_dog` | `0x4002_0000` |
| WDT (Core1) | OpenCores `watch_dog` | `0x4002_1000` |
| WDT (Core2) | OpenCores `watch_dog` | `0x4002_2000` |
| WDT (Core3) | OpenCores `watch_dog` | `0x4002_3000` |
| Timer (Core0) | OpenCores `timer` | `0x4002_4000` |
| Timer (Core1) | OpenCores `timer` | `0x4002_5000` |
| Timer (Core2) | OpenCores `timer` | `0x4002_6000` |
| Timer (Core3) | OpenCores `timer` | `0x4002_7000` |
| UART (AP Dedicated) | OpenCores `uart16550` | `0x4002_8000` | AP 主 Log 输出口；FSI 可跨域读取以监控 AP 状态 |

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

### SoC 基础设施

| IP | 来源 | 说明 |
|----|------|------|
| AXI 互联 | OpenPULP `axi` | 多主多从总线 |
| DDR3 控制器 | Xilinx MIG | DDR3 接口 |

## 启动

| 阶段 | 硬件动作 |
|------|----------|
| 0 | FPGA 位流加载，固化 BootROM 至 `0x0000_0000`，DDR 初始化；仅 FSI 退出复位 |
| 1 | FSI (RV32) 从 BootROM 取指执行，自校验后加载 FSI 固件至 `0x000D_0000` |
| 2 | FSI 固件加载 NPU 至 `0x000B_8000`、AP 镜像至 DDR `0xC000_0000`，逐一验签 |
| 3 | FSI 释放 NPU / AP 复位信号 |

> 固件镜像来源介质（QSPI Flash 等）、验签算法及详细加载流程见 [`boot.md`](boot.md)。

## 复位向量

| 核 | 复位向量 | 说明 |
|----|----------|------|
| FSI (RV32) | `0x0000_0000` | BootROM 入口 |
| NPU (RV32) | `0x000B_8000` | FSI 加载后跳转 |
| AP (RV64) | `0xC000_0000` | FSI 加载 OpenSBI 后跳转 |