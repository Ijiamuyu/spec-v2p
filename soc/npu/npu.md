# NPU (Neural Processing Unit) 子系统设计

## 概述

NPU 子系统是 V2P SoC 的神经网络加速域，包含 1 个 Coral NPU (RV32) 处理器及内置的定时器和看门狗。NPU 通过 AXI 互联访问共享系统资源，与 AP 及 FSI 之间通过 IPC_CTRL / SPINLOCK 进行核间通信。

## 处理器核

| 项目 | 说明 |
|------|------|
| IP | Coral NPU（Google-Verisilicon）|
| 架构 | RV32IMF_Zve32x（标量 + 向量 + 矩阵 MAC）|
| 内置外设 | WDT |
| 内存 | ITCM 8KB + DTCM 32KB（合计 40KB TCM）|

## 内存布局

### NPU 私密内存

| 地址范围 | 容量 | 说明 |
|----------|------|------|
| `0x000B_8000` – `0x000B_9FFF` | 8KB | Coral ITCM（指令 TCM）|
| `0x000B_A000` – `0x000C_1FFF` | 32KB | Coral DTCM（数据 TCM）|

> NPU TCM 位于 `0x000B_8000`–`0x000C_1FFF`，合计 40KB，供 Coral NPU 存放实时性要求高的指令和数据。

### NPU 可访问的共享资源

| 地址范围 | 容量 | 说明 |
|----------|------|------|
| `0x0001_0000` – `0x000B_7FFF` | 672KB | SYSTEM SRAM，核间共享数据区 |
| `0x000C_2000` – `0x000C_2FFF` | 4KB | IPC_CTRL，核间通信控制器（Doorbell + IPI）|
| `0x000C_3000` – `0x000C_3FFF` | 4KB | SPINLOCK，硬件同步锁（32 路 test-and-set）|
| `0xC000_0000` – `0xFFFF_FFFF` | 1GB | DDR3 主存 |

## 中断布局

NPU 使用本地中断控制器，支持 4 路中断。

| 中断号 | 源 | 说明 |
|--------|-----|------|
| 0 | NPU Timer | Coral 内部定时器 |
| 1 | Doorbell DB_APx_NPU（线或） | IPC_CTRL AP→NPU 核间通信，4 路 AP Core 门铃（DB_AP0_NPU–DB_AP3_NPU）线或共享此中断 |
| 2 | Doorbell DB_FSI_NPU | IPC_CTRL FSI→NPU 核间通信 |
| 3 | Reserved | — |

## 看门狗 (WDT)

NPU 内置 WDT，支持两阶段超时机制：

| 阶段 | 超时事件 | 动作 |
|------|----------|------|
| 第一次超时 | WDT 超时预警 | 向 NPU 本地中断发送预警，软件可刷新 WDT 阻止最终复位 |
| 最终超时 | 软件未及时刷新 | 向 FSI 发送复位请求（FSI PLIC #9），FSI 固件收到中断后复位 NPU 域 |

> **复位路径**：NPU WDT → 本地中断预警（软件可恢复）→ FSI PLIC #9 → FSI 固件决策 → PMU `npu_rst_n` 输出。

## 外设清单

| IP | 来源 | 说明 |
|----|------|------|
| Coral NPU Core | Google-Verisilicon | RV32IMF_Zve32x，标量+向量+矩阵 MAC |
| ITCM | — | 8KB，指令紧耦合内存，`0x000B_8000` |
| DTCM | — | 32KB，数据紧耦合内存，`0x000B_A000` |
| NPU Timer | Coral 内部 | 本地中断 #0 |
| NPU WDT | Coral 内部 | 两阶段看门狗 |

## 复位与启动

### 复位向量

| 项目 | 说明 |
|------|------|
| 复位地址 | `0x000B_8000`（ITCM 起始地址）|
| 引导流程 | FSI 加载 NPU 固件至 TCM，然后释放 NPU 复位信号 |
| 复位信号 | `npu_rst_n`，由 PMU 控制 |

### 启动流程

NPU 自身不参与早期启动。由 FSI 安全核完成以下步骤后释放 NPU 复位：

1. FSI BootROM 自校验并加载 FSI 固件
2. FSI 固件验签并加载 NPU 镜像至 `0x000B_8000`（ITCM）
3. FSI 通过 PMU 释放 `npu_rst_n`，NPU 从 `0x000B_8000` 开始执行

### 复位源

NPU 域复位可由以下事件触发：

| 复位源 | 说明 |
|--------|------|
| POR | 上电复位，全芯片复位 |
| NPU WDT 最终超时 | 经 FSI 中断 #9 → FSI 固件 → PMU 输出 `npu_rst_n` |
| FSI SW 写 PMU 寄存器 | FSI 固件主动复位 NPU 域 |

## 核间通信

NPU 通过以下机制与 AP、FSI 通信：

- **IPC_CTRL**（`0x000C_2000`）：含 Doorbell 寄存器，支持 NPU→AP（AP PLIC #21–#24）、NPU→FSI（FSI PLIC #11）、AP→NPU（NPU 中断 #1）、FSI→NPU（NPU 中断 #2）的双向通知
- **SPINLOCK**（`0x000C_3000`）：32 路硬件 test-and-set 锁，用于共享数据结构的互斥访问
- **SYSTEM SRAM**（`0x0001_0000`–`0x000B_7FFF`）：核间共享数据区，配合 SPINLOCK 实现核间通信协议

详见 [`../ip/ipc.md`](../ip/ipc.md)。

## 时钟与电源

| 项目 | 说明 |
|------|------|
| 时钟门控 | PMU `PMU_CLK_GATE.bit1` 控制 NPU 域时钟开关 |
| 复位输出 | PMU 输出 `npu_rst_n` 控制 NPU 域复位 |

> NPU 域时钟门控由 FSI 通过 PMU 管理。NPU 不可自行门控自身时钟。
