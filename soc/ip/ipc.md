# IPC — 核间通信与同步控制器

## 概述

IPC 是 SoC 中负责 AP / FSI / NPU 之间核间通信与同步的硬件模块，包含两个子模块：

| 子模块 | 功能 | 地址 |
|--------|------|------|
| **IPC_CTRL** | Doorbell 中断通知 + 消息队列数据传输 | `0x000C_2000` |
| **SPINLOCK** | 硬件同步锁（test-and-set 互斥） | `0x000C_3000` |

### 属性

| 项目 | IPC_CTRL | SPINLOCK |
|------|----------|----------|
| 范围 | 4KB（`0x000C_2000–0x000C_2FFF`） | 4KB（`0x000C_3000–0x000C_3FFF`） |
| 总线 | AXI-Lite 从口 | AXI-Lite 从口 |
| 访问权限 | AP / FSI / NPU 均可读写 | AP / FSI / NPU 均可读写 |
| 实现 | 自研 RTL（约 400 行） | 自研 RTL（约 100–150 行） |
| 时钟 | 与 AXI 总线同频 | 与 AXI 总线同频 |

---

## IPC_CTRL — 核间通信控制器

IPC_CTRL 集成了 Doorbell 寄存器、IPI 状态寄存器、以及 6 对消息队列的读写指针和队列配置寄存器。

### 1.1 门铃寄存器（`0x000–0x03F`）

每个门铃寄存器为 **Write-1-Trigger** 类型：
- **写操作**：写入任意值（推荐 `0x01`），硬件立即产生一个时钟周期宽度的正脉冲，连接到目标中断控制器的对应中断请求线
- **读操作**：始终返回 `0x0000_0000`

| 偏移 | 寄存器名 | 发送方 | 接收方 | 连接中断 |
|:----:|----------|--------|--------|---------|
| `0x000` | `DB_AP0_NPU` | AP Core0 → NPU | NPU | NPU 中断 #1（与 DB_AP1-3_NPU 线或） |
| `0x004` | `DB_AP1_NPU` | AP Core1 → NPU | NPU | ↳ 同上 |
| `0x008` | `DB_AP2_NPU` | AP Core2 → NPU | NPU | ↳ 同上 |
| `0x00C` | `DB_AP3_NPU` | AP Core3 → NPU | NPU | ↳ 同上 |
| `0x010` | `DB_NPU_AP0` | NPU → AP Core0 | AP Core0 | AP PLIC #21 |
| `0x014` | `DB_NPU_AP1` | NPU → AP Core1 | AP Core1 | AP PLIC #22 |
| `0x018` | `DB_NPU_AP2` | NPU → AP Core2 | AP Core2 | AP PLIC #23 |
| `0x01C` | `DB_NPU_AP3` | NPU → AP Core3 | AP Core3 | AP PLIC #24 |
| `0x020` | `DB_FSI_AP0` | FSI → AP Core0 | AP Core0 | AP PLIC #25 |
| `0x024` | `DB_FSI_AP1` | FSI → AP Core1 | AP Core1 | AP PLIC #26 |
| `0x028` | `DB_FSI_AP2` | FSI → AP Core2 | AP Core2 | AP PLIC #27 |
| `0x02C` | `DB_FSI_AP3` | FSI → AP Core3 | AP Core3 | AP PLIC #28 |
| `0x030` | `DB_AP_FSI` | AP（任意核）→ FSI | FSI | FSI PLIC #10 |
| `0x034` | `DB_FSI_NPU` | FSI → NPU | NPU | NPU 中断 #2 |
| `0x038` | `DB_NPU_FSI` | NPU → FSI | FSI | FSI PLIC #11 |
| `0x03C` | `DB_ERR` | 任意主设备 | FSI | FSI PLIC #12（Doorbell 异常上报） |

> **定向设计**：NPU→AP 和 FSI→AP 各分配 4 个独立门铃，分别对应 AP Core0–3。发送方写入对应的门铃寄存器，只有目标核被中断唤醒。
>
> **`DB_ERR` 触发条件**：当主设备写入 `0x040–0xFFC` 范围内的未定义偏移地址时，门铃模块返回 SLVERR 并同时触发 `DB_ERR` 中断脉冲。

### 1.2 IPI 状态与使能寄存器（`0x040–0x047`）

| 偏移 | 寄存器 | 宽度 | 复位值 | 访问 | 说明 |
|:----:|--------|:----:|:------:|:----:|------|
| `0x040` | `IPI_STATUS` | 32b | 0 | RO | Doorbell pending 状态，只读。每位对应一个门铃寄存器：bit0=DB_AP0_NPU, bit1=DB_AP1_NPU, ..., bit15=DB_ERR。写操作无效 |
| `0x044` | `IPI_ENABLE` | 32b | 0xFFFF | R/W | 门铃中断掩码。1=使能，0=屏蔽。复位后全部使能 |

> `IPI_STATUS` 的位映射与门铃寄存器偏移按顺序一一对应：bitN 对应偏移 `0x000 + N×4` 的门铃寄存器。

### 1.3 队列指针寄存器（`0x080–0x0AF`）

管理 6 对无锁环形消息队列。每个队列有独立的写指针（WPTR，生产者写入）和读指针（RPTR，消费者写入），两者独立无互斥。

队列容量由 `QUEUE_*_SIZE` 寄存器指定（见 1.4 节），必须为 2 的幂。队列满/空判定：
- **空**：`wptr == rptr`
- **满**：`(wptr - rptr) == size`

| 偏移 | 寄存器 | 宽度 | 复位值 | 可写方 | 说明 |
|:----:|--------|:----:|:------:|--------|------|
| `0x080` | `QUEUE_AP2NPU_WPTR` | 32b | 0 | AP | AP→NPU 队列写指针 |
| `0x084` | `QUEUE_AP2NPU_RPTR` | 32b | 0 | NPU | AP→NPU 队列读指针 |
| `0x088` | `QUEUE_NPU2AP_WPTR` | 32b | 0 | NPU | NPU→AP 队列写指针 |
| `0x08C` | `QUEUE_NPU2AP_RPTR` | 32b | 0 | AP | NPU→AP 队列读指针 |
| `0x090` | `QUEUE_AP2FSI_WPTR` | 32b | 0 | AP | AP→FSI 队列写指针 |
| `0x094` | `QUEUE_AP2FSI_RPTR` | 32b | 0 | FSI | AP→FSI 队列读指针 |
| `0x098` | `QUEUE_FSI2AP_WPTR` | 32b | 0 | FSI | FSI→AP 队列写指针 |
| `0x09C` | `QUEUE_FSI2AP_RPTR` | 32b | 0 | AP | FSI→AP 队列读指针 |
| `0x0A0` | `QUEUE_FSI2NPU_WPTR` | 32b | 0 | FSI | FSI→NPU 队列写指针 |
| `0x0A4` | `QUEUE_FSI2NPU_RPTR` | 32b | 0 | NPU | FSI→NPU 队列读指针 |
| `0x0A8` | `QUEUE_NPU2FSI_WPTR` | 32b | 0 | NPU | NPU→FSI 队列写指针 |
| `0x0AC` | `QUEUE_NPU2FSI_RPTR` | 32b | 0 | FSI | NPU→FSI 队列读指针 |

> **访问控制**：硬件不实施写方限制——"可写方"仅为协议约定。FSI 作为安全核可强制恢复异常指针值。

### 1.4 队列配置寄存器（`0x0C0–0x0EF`）

由 FSI 在系统初始化时统一配置。基地址必须对齐到队列容量边界。

| 偏移 | 寄存器 | 宽度 | 复位值 | 说明 |
|:----:|--------|:----:|:------:|------|
| `0x0C0` | `QUEUE_AP2NPU_ADDR` | 32b | 0 | AP→NPU 队列基地址（可指向 SYSTEM SRAM 或 DDR） |
| `0x0C4` | `QUEUE_AP2NPU_SIZE` | 32b | 0 | AP→NPU 队列容量（字节，必须为 2 的幂） |
| `0x0C8` | `QUEUE_NPU2AP_ADDR` | 32b | 0 | NPU→AP 队列基地址 |
| `0x0CC` | `QUEUE_NPU2AP_SIZE` | 32b | 0 | NPU→AP 队列容量 |
| `0x0D0` | `QUEUE_AP2FSI_ADDR` | 32b | 0 | AP→FSI 队列基地址 |
| `0x0D4` | `QUEUE_AP2FSI_SIZE` | 32b | 0 | AP→FSI 队列容量 |
| `0x0D8` | `QUEUE_FSI2AP_ADDR` | 32b | 0 | FSI→AP 队列基地址 |
| `0x0DC` | `QUEUE_FSI2AP_SIZE` | 32b | 0 | FSI→AP 队列容量 |
| `0x0E0` | `QUEUE_FSI2NPU_ADDR` | 32b | 0 | FSI→NPU 队列基地址 |
| `0x0E4` | `QUEUE_FSI2NPU_SIZE` | 32b | 0 | FSI→NPU 队列容量 |
| `0x0E8` | `QUEUE_NPU2FSI_ADDR` | 32b | 0 | NPU→FSI 队列基地址 |
| `0x0EC` | `QUEUE_NPU2FSI_SIZE` | 32b | 0 | NPU→FSI 队列容量 |
| `0x0F0–0xFFC` | Reserved | — | — | 预留（保留给更多队列对或扩展寄存器）|

---

## SPINLOCK — 硬件同步锁

SPINLOCK 提供 test-and-set 语义的锁寄存器阵列，用于共享资源的原子互斥：

- **读操作**：原子完成"判断空闲→置位→返回"，返回 1 表示获取锁成功，0 表示失败
- **写 `0x0000_0001`**：释放锁（bit0 清零）

> 物理基地址 `0x000C_3000`。下文偏移均相对该基地址。

### 2.1 锁寄存器阵列（`0x000–0x07C`）

32 个独立锁寄存器，每锁 4 字节，共 128 字节。

| 偏移 | 寄存器 | 宽度 | 复位值 | 访问 | 说明 |
|:----:|--------|:----:|:------:|:----:|------|
| `0x000` | `LOCK00` | 32b | 0 | R/W | 锁 #0 |
| `0x004` | `LOCK01` | 32b | 0 | R/W | 锁 #1 |
| `0x008` | `LOCK02` | 32b | 0 | R/W | 锁 #2 |
| `0x00C` | `LOCK03` | 32b | 0 | R/W | 锁 #3 |
| ... | ... | 32b | 0 | R/W | ... |
| `0x07C` | `LOCK31` | 32b | 0 | R/W | 锁 #31 |
| `0x080–0xFFC` | Reserved | — | — | — | 预留扩展（最多 1024 锁）|

### 2.2 锁寄存器语义

每个 `LOCKn` 寄存器的 **bit0** 为锁状态，bit[31:1] 读出恒为 0、写入忽略：

| 操作 | 行为 |
|------|------|
| **读**（原子 test-and-set） | 若 bit0=0（空闲）→ 硬件置 bit0=1，返回 `0x0000_0001` ✅ **获取成功** |
| | 若 bit0=1（已占用）→ 硬件不变，返回 `0x0000_0000` ❌ **获取失败** |
| **写 `0x0000_0001`** | bit0 清零，释放锁 |
| **写 `0x0000_0000`** | 忽略 |
| 写 bit[31:1] 非零 | 忽略 |

> **原子性保证**：读事务在 AXI-Lite 从口内部完成"判断→置位→返回"三个动作，对总线表现为单次原子操作。多个主设备同时读同一 LOCKn 时，AXI 互联仲裁确保仅一个赢得置位，其余返回 0。

> SPINLOCK 无中断输出。

---

## 中断汇总

| 源 | 中断脉冲输出 | 目标 |
|----|-------------|------|
| `DB_AP0_NPU`–`DB_AP3_NPU` | 4 路独立脉冲 → 线或 | NPU 本地中断 #1 |
| `DB_NPU_AP0`–`DB_NPU_AP3` | 4 路独立脉冲 | AP PLIC #21–#24 |
| `DB_FSI_AP0`–`DB_FSI_AP3` | 4 路独立脉冲 | AP PLIC #25–#28 |
| `DB_AP_FSI` | 1 路脉冲 | FSI PLIC #10 |
| `DB_FSI_NPU` | 1 路脉冲 | NPU 本地中断 #2 |
| `DB_NPU_FSI` | 1 路脉冲 | FSI PLIC #11 |
| `DB_ERR` | 1 路脉冲 | FSI PLIC #12 |
| SPINLOCK | 无中断 | — |

---

## 复位行为

| 复位事件 | IPC_CTRL 门铃 | IPC_CTRL IPI 寄存器 | IPC_CTRL 队列指针 | IPC_CTRL 队列配置 | SPINLOCK 锁状态 |
|----------|:-------------:|:-------------------:|:-----------------:|:-----------------:|:---------------:|
| AP 单核复位 | 不影响 | 保留 | 保留 | 保留 | 不影响 |
| FSI 复位 | 不影响 | 保留 | 保留 | 保留 | 不影响 |
| NPU 复位 | 不影响 | 保留 | 保留 | 保留 | 不影响 |
| SoC 全局复位 | 归零 | 归零 | 归零 | 归零 | 全部清零 |

> 子系统独立复位不自动清除队列指针和锁状态——硬件无法可靠判定持锁者或通信中间状态。FSI 作为安全监控核，在检测到 AP/NPU 复位事件后遍历锁状态并强制释放孤儿锁，同时清理残留队列指针。