# V2P QEMU 仿真

## 版本

| 版本 | 日期 | 说明 |
|------|------|------|
| **v0.1** | 2026-07-26 | AP 最小系统：4 核 sifive-u54 + CLINT + PLIC + UART + SYSTEM SRAM + DDR |

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
│  │     │ │     │  │16550  │  │ SRAM │  │  (1GB)   │   │
│  │0x400│ │0x400│  │0x4002 │  │0x0001│  │0xC000    │   │
│  │_1000│ │_0000│  │_8000  │  │_0000 │  │_0000     │   │
│  └─────┘ └─────┘  └───────┘  └──────┘  └──────────┘   │
└──────────────────────────────────────────────────────────┘
```

## 内存映射

| 地址 | 大小 | 组件 |
|------|------|------|
| `0x0001_0000` | 672KB | SYSTEM SRAM（核间共享） |
| `0x4000_0000` | 64KB | PLIC |
| `0x4001_0000` | 64KB | CLINT |
| `0x4002_8000` | 4KB | UART (16550) |
| `0xC000_0000` | 1GB | DDR3（复位向量） |

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
# 输出: v2p-ap   RISC-V V2P AP (Application Processor) minimal system
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
