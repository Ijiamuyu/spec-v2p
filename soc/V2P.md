# V2P SoC 硬件设计

## 内存布局

### 全局共享

| 区域 | 地址范围 | 容量 | 说明 |
|------|----------|------|------|
| DDR3 | `0xC000_0000` – `0xFFFF_FFFF` | 1GB | 主存，所有主设备可访问 |
| SYSTEM SRAM | `0x0000_A000` – `0x000B_1FFF` | 672KB | 核间共享数据区 |
| EEPROM | `0x0000_0000` – `0x0000_1FFF` | 8KB | I2C 设备，启动参数 |

### FSI 专有

| 区域 | 地址范围 | 容量 | 说明 |
|------|----------|------|------|
| BootROM | `0x0000_2000` – `0x0000_9FFF` | 32KB | 独立 BRAM，位流固化，仅 FSI 可读 |
| FSI 私有 SRAM | `0x000C_0000` – `0x000D_FFFF` | 128KB | FSI 固件加载目标 |
| FSI 外设 | `0x2000_0000` – `0x2006_0000` | — | AES / SHA / RSA / TRNG / UART |

### AP 专有

| 区域 | 地址范围 | 容量 | 说明 |
|------|----------|------|------|
| PLIC | `0x4000_0000` – `0x4000_FFFF` | 64KB | 外部中断路由 |
| CLINT | `0x4001_0000` – `0x4001_FFFF` | 64KB | 核间中断 + mtime 定时器 |

### NPU 专有

| 区域 | 地址范围 | 容量 | 说明 |
|------|----------|------|------|
| NPU TCM | `0x000E_0000` – `0x000E_9FFF` | 40KB | Coral 内部 ITCM 8KB + DTCM 32KB，FSI 写固件用（NPU 内部直连，仅加载暴露到总线） |

### 公共外设

| 区域 | 地址范围 | 容量 | 说明 |
|------|----------|------|------|
| LSIO | `0x3000_0000` – `0x3005_FFFF` | — | 低速外设：SPI / I2C / UART / DMA / GPIO / WDT |
| 其他 | `0x3006_0000` – `0x300B_FFFF` | — | Ethernet / SD/MMC / RTC / NPU Debug UART / Timer×3 |

> FSI / AP / NPU 专有区域由 AXI 访问控制硬编码隔离；公共外设 AP 与 FSI 均可访问。

## 中断布局

### AP 本地中断 (CLINT)

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0–3 | 软件中断 | 每核 1 个 |
| 4–7 | 定时器中断 | 每核 1 个 |

### AP 外部中断 (PLIC)

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 42 | LSIO UART | AP 控制台 |
| 43 | LSIO SPI | |
| 44 | LSIO I2C | EEPROM 访问 |
| 45 | LSIO DMA | |
| 46 | NPU Debug UART | |
| 47 | Ethernet | |
| 48 | SD/MMC | |
| 49 | LSIO WDT | |
| 50 | LSIO RTC | |
| 51–53 | LSIO Timer 0–2 | |

### FSI 本地中断

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 5 | FSI Timer | 专属定时器 |
| 10 | FSI UART | FSI 控制台 |

> AES / SHA / RSA / TRNG 纯 MMIO 轮询，无中断需求。

### NPU 本地中断

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 5 | NPU Timer | Coral 内部定时器 |

> 核间通信中断待定。

## 挂载 IP

### 处理器核

| IP | 归属 | 说明 |
|----|------|------|
| VexRiscv-SMP ×4 | AP | SpinalHDL，RV64IMAFDC，Sv39 MMU，缓存一致性 |
| Coral NPU | NPU | Google-Verisilicon，RV32IMF_Zve32x，标量+向量+矩阵 MAC |
| VexRiscv RV32 | FSI | SpinalHDL，RV32IM，安全核 |

### FSI 外设

| IP | 来源 | 地址 |
|----|------|------|
| AES-256 | OpenCores `aes_core` | `0x2000_0000` |
| SHA-256 | OpenCores `sha256` | `0x2001_0000` |
| RSA-2048 | OpenCores `basicrsa` | `0x2002_0000` |
| TRNG | OpenTitan `entropy_src` | `0x2003_0000` |
| UART | OpenCores `uart16550` | `0x2005_0000` |

### AP 外设

| IP | 来源 | 地址 |
|----|------|------|
| PLIC | OpenPULP | `0x4000_0000` |
| CLINT | OpenPULP | `0x4001_0000` |

### 公共外设

| IP | 来源 | 地址 |
|----|------|------|
| SPI | OpenCores `simple_spi` | `0x3000_0000` |
| I2C | OpenCores `i2c_master_top` | `0x3001_0000` |
| DMA | OpenCores `wb_dma` | `0x3002_0000` |
| UART (AP) | OpenCores `uart16550` | `0x3003_0000` |
| GPIO | OpenCores `gpio` | `0x3004_0000` |
| WDT | OpenCores `watch_dog` | `0x3005_0000` |
| Ethernet MAC | OpenCores `ethmac` | `0x3006_0000` |
| SD/MMC | OpenCores `sd_controller` | `0x3007_0000` |
| RTC | OpenCores `rtc` | `0x3008_0000` |
| UART (NPU Debug) | OpenCores `uart16550` | `0x3009_0000` |
| Timer ×3 | OpenCores `timer` | `0x300B_0000` |

### SoC 基础设施

| IP | 来源 | 说明 |
|----|------|------|
| AXI 互联 | OpenPULP `axi` | 多主多从总线 |
| DDR3 控制器 | Xilinx MIG | DDR3 接口 |

## 启动

| 阶段 | 硬件动作 |
|------|----------|
| 0 | FPGA 位流加载，固化 BootROM 至 `0x0000_2000`，DDR 初始化；仅 FSI 退出复位 |
| 1 | FSI (RV32) 从 BootROM 取指执行，自校验后加载 FSI 固件至 `0x000C_0000` |
| 2 | FSI 固件加载 NPU 至 `0x000E_0000`、AP 镜像至 DDR `0xC000_0000`，逐一验签 |
| 3 | FSI 释放 NPU / AP 复位信号 |

> 详见 [`boot.md`](boot.md)。

## 复位向量

| 核 | 复位向量 | 说明 |
|----|----------|------|
| FSI (RV32) | `0x0000_2000` | BootROM 入口 |
| NPU (RV32) | `0x000E_0000` | FSI 加载后跳转 |
| AP (RV64) | `0xC000_0000` | FSI 加载 OpenSBI 后跳转 |