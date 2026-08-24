# SiFive I2C — sifive-blocks i2c 控制器

## 概述

I2C 控制器采用 **SiFive sifive-blocks `i2c`**（OpenCores `i2c_master_top`（Richard Herveille）的 fork），compatible `sifive,fu540-c000-i2c`，Linux 驱动为 `i2c-ocores`。该 IP 经 SiFive **FU540-C000 / FU740-C000** 实硅验证。

### 属性

| 项目 | 说明 |
|------|------|
| 实例 | I2C0（`0x0012_0000`）、I2C1（`0x0013_0000`）|
| 范围 | 各 64KB（`0x0012_0000–0x0012_FFFF` / `0x0013_0000–0x0013_FFFF`）|
| 总线 | AXI4-Lite 从口 |
| 寄存器步进 | **32-bit**（`reg-shift=2`），仅低 8 位有效，高 24 位保留（读 0）|
| 访问权限 | AP / FSI 均可读写 |
| 中断 | I2C0 → AP PLIC #5 / FSI PLIC #8；I2C1 → AP PLIC #6（FSI 仅 Poll，无中断通路）|
| 挂载设备 | I2C0 挂 EEPROM 24C64（从地址 `0xA0`，64Kbit）|

---

## 寄存器映射

| 偏移 | 名称 | 访问 | 复位值 | 说明 |
|:----:|------|:----:|:------:|------|
| `0x00` | `PRERlo` | RW | `0xFF` | 预分频低字节 |
| `0x04` | `PRERhi` | RW | `0xFF` | 预分频高字节 |
| `0x08` | `CTR` | RW | `0x00` | 控制寄存器 |
| `0x0C` | `TXR`/`RXR` | W/R | `0x00` | 发送/接收数据 |
| `0x10` | `CR`/`SR` | W/R | `0x00` | 命令（写）/ 状态（读）|

> 仅低 8 位有效，高 24 位保留（读 0）。

---

## 寄存器位定义

### CTR（`0x08`）— 控制寄存器

| bit | 名称 | 复位 | 说明 |
|:---:|------|:---:|------|
| 7 | `EN` | 0 | 核使能 |
| 6 | `IEN` | 0 | 中断使能 |
| 5:0 | — | 0 | 保留 |

### CR（`0x10`，写）— 命令寄存器

| bit | 名称 | 说明 |
|:---:|------|------|
| 7 | `STA` | 产生 START |
| 6 | `STO` | 产生 STOP |
| 5 | `RD` | 读 |
| 4 | `WR` | 写 |
| 3 | `ACK` | 应答（0=发 ACK，1=发 NACK）|
| 2:1 | — | 保留 |
| 0 | `IACK` | 中断应答 |

### SR（`0x10`，读）— 状态寄存器

| bit | 名称 | 说明 |
|:---:|------|------|
| 7 | `NACK` | 收到 NACK（0=ACK，1=NACK）|
| 6 | `BUSY` | 总线忙 |
| 5 | `ARBLOST` | 仲裁丢失 |
| 4:2 | — | 保留 |
| 1 | `TIP` | 传输进行中 |
| 0 | `IF` | 中断标志 |

---

## 命令组合值（写 CR）

| 命令 | 值 | 说明 |
|------|:---:|------|
| `START` | `0x91` | STA + WR + IACK |
| `STOP` | `0x41` | STO + IACK |
| `READ` | `0x21` | RD + IACK |
| `WRITE` | `0x11` | WR + IACK |
| `READ_NACK` | `0x29` | RD + ACK + IACK（末字节读后发 NACK）|
| `IACK` | `0x01` | 仅清中断标志 |

---

## 预分频

$$PRER = \frac{f_{clk}}{5 \times f_{scl}} - 1$$

- `f_clk`：I2C 输入时钟（FPGA 上由时钟树提供）。
- `f_scl`：目标 SCL 频率（100K / 400K / 1M / 3.4M）。
- 例：`f_clk=50MHz`、`f_scl=100KHz` → `PRER = 50e6/(5*100e3) - 1 = 99`（`PRERlo=0x63`，`PRERhi=0x00`）。

---

## 传输流程

### 写（master_write）

```
START → 发 dev_addr|W → 发内部地址（reg_width 字节）→ 发数据… → STOP
```

### 读（master_read）

```
START → 发 dev_addr|W → 发内部地址（reg_width 字节）→ START(重复) → 发 dev_addr|R → 收数据…（末字节 READ_NACK）→ STOP
```

- `dev_addr` 为 8-bit 地址（含 R/W 位）：写用 `dev_addr & ~1`，读用 `dev_addr | 1`。
- 24C64 内部地址为 2 字节（`reg_width=2`）。

---

## 中断

- `CTR.IEN=1` 使能中断；传输完成置 `SR.IF`，若 `IEN` 置位则拉高中断线到 PLIC。
- 软件在 `irq_handler` 中写 `CR.IACK` 清 `IF` 并推进状态机。

---

## 参考

- Linux 驱动：`drivers/i2c/busses/i2c-ocores.c`
- 设备树 compatible：`sifive,fu540-c000-i2c`（`reg-shift=<2>`）
- 实硅参考：SiFive FU540-C000 / FU740-C000