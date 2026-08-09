# V2P QEMU 仿真

## 版本

| 版本 | 日期 | 说明 |
|------|------|------|
| **v0.2** | 2026-08-08 | 地址修正 + 外设扩展规划：PLIC/CLINT/UART 修正为 `0x3000_0000` 系列；规划 WDT×4、Timer×4、LSIO 全外设（sifive-blocks 替换） |
| v0.1 | 2026-07-26 | AP 最小系统：4 核 sifive-u54 + CLINT + PLIC + UART + SYSTEM SRAM + DDR（旧地址 `0x4000_0000`，已废弃） |

## 系统架构

```
┌──────────────────────────────────────────────────────────┐
│  V2P AP QEMU Machine (v2p-ap)                            │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │  Hart 0  │  │  Hart 1  │  │  Hart 2  │  │  Hart 3  │ │
│  │ sifive-  │  │ sifive-  │  │ sifive-  │  │ sifive-  │ │
│  │   u54    │  │   u54    │  │   u54    │  │   u54    │ │
│  │ RV64 MSU │  │ RV64 MSU │  │ RV64 MSU │  │ RV64 MSU │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
│       │              │              │              │      │
│  ┌────┴──────────────┴──────────────┴──────────────┴──┐  │
│  │                   AXI Interconnect                  │  │
│  └──┬────────┬──────────┬──────────┬──────────┬───────┘  │
│     │        │          │          │          │          │
│  ┌──┴──┐ ┌──┴──┐  ┌───┴───┐  ┌──┴───┐  ┌───┴──────┐   │
│  │CLINT│ │PLIC │  │ UART  │  │System│  │   DDR    │   │
│  │     │ │     │  │sifive │  │ SRAM │  │  (1GB)   │   │
│  │0x300│ │0x300│  │0x3005 │  │0x0001│  │0xC000    │   │
│  │4_0000│ │0_0000│  │_8000  │  │_0000 │  │_0000     │   │
│  └─────┘ └─────┘  └───────┘  └──────┘  └──────────┘   │
└──────────────────────────────────────────────────────────┘
```

## 内存映射

### 当前已实现

| 地址 | 大小 | 组件 |
|------|------|------|
| `0x0001_0000` | 672KB | SYSTEM SRAM（核间共享） |
| `0x3000_0000` | 256KB | PLIC |
| `0x3004_0000` | 64KB | CLINT |
| `0x3005_8000` | 4KB | UART（当前 16550，规划替换 sifive uart） |
| `0xC000_0000` | 1GB | DDR3（复位向量） |

### 外设扩展规划（待实施，对应 IP 替换方案）

| 地址 | 大小 | 组件 | PLIC# | QEMU 模型 |
|------|------|------|:---:|-----------|
| `0x3005_0000` | 4KB | WDT Core0 | #11 | `opencores-wdt`（自研） |
| `0x3005_1000` | 4KB | WDT Core1 | #12 | 同上 |
| `0x3005_2000` | 4KB | WDT Core2 | #13 | 同上 |
| `0x3005_3000` | 4KB | WDT Core3 | #14 | 同上 |
| `0x3005_4000` | 4KB | Timer Core0 | #16 | `opencores-timer`（自研） |
| `0x3005_5000` | 4KB | Timer Core1 | #17 | 同上 |
| `0x3005_6000` | 4KB | Timer Core2 | #18 | 同上 |
| `0x3005_7000` | 4KB | Timer Core3 | #19 | 同上 |
| `0x0010_0000` | 64KB | QSPI0 | #3 | `sifive_spi` + `m25p80(n25q256a)` |
| `0x0011_0000` | 64KB | QSPI1 | #4 | `sifive_spi` |
| `0x0012_0000` | 64KB | I2C0 | #5 | `ocores-i2c`（自研）+ `at24c(24LC02)` |
| `0x0013_0000` | 64KB | I2C1 | #6 | `ocores-i2c`（自研） |
| `0x0015_0000` | 64KB | UART 扩展 | #2 | `sifive_uart` |
| `0x0016_0000` | 64KB | GPIO | — | `sifive_gpio` |
| `0x0017_0000` | 64KB | Ethernet MAC | #9 | `opencores_eth` |
| `0x0018_0000` | 64KB | SD/MMC | #10 | `cadence_sdhci` + SD 卡 |
| `0x0019_0000` | 64KB | RTC | #15 | `goldfish_rtc` 或 自研 |
| `0x001A_0000` | 64KB | PWM | — | `sifive_pwm` |
| `0x0020_0000` | 1MB | DMA | #7 | `sifive_pdma`（1MB 对齐）|

> 地址/中断编号与 `spec/soc/V2P.md`、`spec/soc/ap/ap.md` 完全一致。未列出区域保持 unimp 占位。
> PLIC 目标上下文规划：`"MS,MS,MS,MS"`（每核 M+S，匹配 core-v-plic 双模式）。

## 编译

```bash
# 清理输出目录
rm -rf out/qemu && mkdir -p out/qemu

# 配置（仅 riscv64）
cd out/qemu
../../qemu/configure --target-list=riscv64-softmmu --disable-werror

# 编译
ninja qemu-system-riscv64
```

产物：`out/qemu/qemu-system-riscv64`

## 使用

### 裸机程序

```bash
# 编译裸机程序（复位向量 0xC000_0000，RV64 裸机）
riscv64-unknown-elf-gcc -march=rv64imafdc -mabi=lp64d -mcmodel=medany \
    -T v2p.ld -nostdlib -o firmware.elf start.S main.c

# 运行
out/qemu/qemu-system-riscv64 -M v2p-ap -m 1G \
    -bios firmware.elf -nographic
```

### OpenSBI + Linux（后续）

```bash
out/qemu/qemu-system-riscv64 -M v2p-ap -m 1G \
    -bios fw_dynamic.bin \
    -kernel Image -append "console=ttyS0" \
    -nographic
```

### 查看支持的机器

```bash
out/qemu/qemu-system-riscv64 -M help | grep v2p
# 输出: v2p-ap   RISC-V V2P AP (Application Processor) machine
```

### GDB 调试

QEMU 启动 GDB server，监听 `localhost:1234`：

```bash
# 终端1：启动 QEMU（-s 开启 GDB server，-S 暂停等待 GDB 连接）
out/qemu/qemu-system-riscv64 -M v2p-ap -m 1G \
    -bios firmware.elf -nographic -s -S
```

```bash
# 终端2：启动 GDB 并连接
riscv64-unknown-elf-gdb firmware.elf \
    -ex "target extended-remote :1234"
```

### 多核调试

V2P AP 为 4 核 SMP，GDB 连接后每个 hart 为一个线程：

```
(gdb) info threads
  Id   Target Id                 Frame
* 1    Thread 1 (RV64 hart 0)    _start () at start.S:5
  2    Thread 2 (RV64 hart 1)    _park () at start.S:12
  3    Thread 3 (RV64 hart 2)    _park () at start.S:12
  4    Thread 4 (RV64 hart 3)    _park () at start.S:12

(gdb) thread 2                    # 切换到 hart1
(gdb) set scheduler-locking on   # 单步仅当前 hart
```

### 常用 GDB 命令

| 命令 | 说明 |
|------|------|
| `c` | 继续运行 |
| `si` | 单步执行一条指令 |
| `b *0xC0000000` | 在复位向量设断点 |
| `b main` | 在 main 函数设断点 |
| `info reg` | 查看所有寄存器 |
| `p/x $mstatus` | 查看 M 模式状态寄存器 |
| `x/4i $pc` | 反汇编当前 PC 开始 4 条指令 |
| `monitor info mtree` | QEMU 地址空间树 |
| `monitor info qtree` | QEMU 设备树 |

## 源码

| 文件 | 说明 |
|------|------|
| `qemu/include/hw/riscv/v2p_ap.h` | SoC 设计常量 |
| `qemu/hw/riscv/v2p_ap.c` | 机器实现 |
| `qemu/hw/riscv/Kconfig` | 构建配置 |
| `qemu/hw/riscv/meson.build` | 编译规则 |
| `qemu/hw/timer/opencores_timer.c` | OpenCores Timer（自研） |
| `qemu/hw/watchdog/opencores_wdt.c` | OpenCores WDT（自研） |
| `qemu/hw/i2c/ocores_i2c.c` | I2C 控制器（自研，compatible `sifive,fu540-c000-i2c`） |

> 复用主线模型：`hw/ssi/sifive_spi.c`、`hw/char/sifive_uart.c`、`hw/gpio/sifive_gpio.c`、`hw/timer/sifive_pwm.c`、`hw/dma/sifive_pdma.c`、`hw/sd/cadence_sdhci.c`、`hw/net/opencores_eth.c`、`hw/nvram/eeprom_at24c.c`、`hw/block/m25p80.c`。
