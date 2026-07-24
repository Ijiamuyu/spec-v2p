# BootROM 设计

BootROM 独立 BRAM（`0x0000_0000`–`0x0000_7FFF`）分为 eFuse OTP 仿真区和代码区，仅 FSI 可访问。eFuse OTP 为内存映射（`0x0000_0000`–`0x0000_0FFF`），启动参数、密钥、校验值均固化于其中。
系统上电后 FSI 从 BootROM 代码区启动，自校验通过后加载 FSI 固件。

## 地址布局

```
eFuse OTP      0x0000_0000 ~ 0x0000_0FFF (4KB, 位流固化, 只读不可写)
BootROM        0x0000_1000 ~ 0x0000_7FFF (28KB, 独立 BRAM, FSI 只读)
SYSTEM SRAM    0x0001_0000 ~ 0x000B_7FFF (672KB, 核间共享)
NPU TCM        0x000B_8000 ~ 0x000C_1FFF (40KB, Coral ITCM 8KB + DTCM 32KB)
FSI 私有 SRAM  0x000D_0000 ~ 0x000E_FFFF (128KB, 固件加载目标)
```

## 启动流程

| 阶段 | 操作 |
|------|------|
| 0 | FPGA 位流固化 eFuse OTP 及 BootROM，DDR 初始化，仅 FSI 退出复位 |
| 1 | 初始化 UART、SPI；自校验：SHA-256(BootROM) vs eFuse OTP 哈希，不匹配 → 停机 |
| 2 | lifecycle=Debug/Test → waiting 倒计时（默认 2s），`q`/`Q` → 终端；Release → 跳过 |
| 3 | 读 KEY[1:0] + eFuse OTP 确定启动源；加载 FSI 固件至 `0x000D_0000`（加密则 AES 解密）；SHA-256 校验通过则跳转（`skip_verify=1` 跳过）；校验失败 → Debug/Test 回退终端，Release 打印错误后停机 |
| 4 | FSI 固件接管：lifecycle=Debug/Test → waiting 倒计时（默认 1s），`q`/`Q` → 终端；读 KEY + eFuse OTP 确定 AP/NPU 介质，逐镜像加载： |
|   | **NPU** → `0x000B_8000`：SHA-256 → RSA 验签（`skip_verify=1` 跳过）；失败 → Debug/Test 回退终端，Release 打印错误 |
|   | **AP** → DDR：RTOS 裸机 或 OpenSBI → U-Boot → Linux，SHA-256 → RSA 验签（`skip_verify=1` 跳过）；失败 → Debug/Test 回退终端，Release 打印错误 |
|   | 全部通过 → 释放各核复位 |
| 5 | FSI 守护：运行时安全监控、心跳、WDT |

- 启动源优先级：SPI Flash（默认）→ UART XMODEM（回退）；KEY[1:0]=`01`（SD）时 BootROM 忽略回退 SPI。FSI 固件启动后支持 SD。
- KEY[1:0] 在阶段 3 锁存，通过共享 SRAM 传递 FSI 固件复用，无需重读 GPIO。
- Release 模式下 waiting=0，无交互入口。
- 等待时长通过 eFuse OTP 配置（`bootrom_wait` / `fsi_wait`），`0x0000` 跳过。

## UART 终端模式

waiting 期间 `q`/`Q` 进入交互终端（Release 不可用）：

| 命令 | 说明 |
|------|------|
| `help` | 命令列表 |
| `boot spi` / `boot sd` | 从 SPI Flash / SD 加载启动（`boot sd` 仅 FSI 固件） |
| `loadx <addr>` | XMODEM 下载至 addr（CRC 校验） |
| `go <addr>` | 跳转执行 |
| `md <addr> <len>` / `mw <addr> <val>` | 读写内存（`mw!` 跳过写保护，仅 Debug） |
| `crc <addr> <len>` / `sha256 <addr> <len>` | CRC32 / SHA-256 |
| `reset` / `info` | 软复位 / 版本号、密钥摘要、生命周期 |

## SoC 硬件自检

Debug/Test 模式下可用，分两级：

| 级别 | 执行体 | 范围 |
|------|--------|------|
| 一级 | BootROM | UART、SPI Flash、SHA-256 IP |
| 二级 | FSI 固件 | DDR、SRAM、SD、加密 IP、以太网等 |

- 一级失败 → 停机打印错误码；二级结果通过核间通道上报 AP。
- 均为硬件基础功能测试，具体命令留待后续阶段。

## SoC 生命周期

eFuse OTP 偏移 `0x000` 处的 8 字节 `Lifecycle` 唯一决定，位流编译时写入，上电即生效，单向不可逆：

```
Debug → Test → Release
```

| 状态 | 值 | UART 终端 | `mw!` |
|------|----|:---------:|:-----:|
| Debug | `0x00` | ✅ | ✅ |
| Test | `0x01` | ✅ | ❌ |
| Release | `0x02` | ❌ | ❌ |

FPGA 阶段通过更新位流切换；ASIC 阶段通过 eFuse 编程器写入，不可逆。

## 加密校验

默认 MMIO 调用 FSI 子系统硬件 IP（随位流就绪）；异常或 eFuse OTP `crypto_mode=0x01` 时降级软件实现：

| IP | 来源 | 用途 | 软件降级开销 |
|----|------|------|:---:|
| SHA-256 | OpenCores `sha256` | 自校验 + 固件哈希 | ~2KB |
| AES-256 | OpenCores `aes_core` | 固件解密 | ~4KB |
| RSA-2048 | OpenCores `basicrsa` | 固件验签 | ~8KB |

## 代码规格

| 项目 | 规格 |
|------|------|
| 指令集 | RV32IM（Zicsr） |
| 大小 | ≤ 28KB（硬件优先 ~5KB；全软件 ~17KB，余量充裕） |
| 外设 | SPI Flash、UART、GPIO（KEY） |

## eFuse OTP 存储布局 (4KB, `0x0000_0000–0x0000_0FFF`)

| 偏移 | 字段 | 大小 | 说明 |
|:----:|------|:----:|------|
| `0x000` | Lifecycle | 8B | `0x00` Debug / `0x01` Test / `0x02` Release |
| `0x008` | 根密钥种子 | 256B | AES/SHA/RSA 派生密钥种子 |
| `0x108` | 公钥哈希 | 32B | FSI 固件签名公钥 SHA-256 摘要 |
| `0x128` | BootROM 哈希 | 32B | BootROM 代码区自校验预期值 |
| `0x148` | 芯片 ID | 16B | 唯一标识 |
| `0x158` | skip_verify | 1B | `0x01` 跳过校验（仅 Debug/Test） |
| `0x159` | crypto_mode | 1B | `0x00` 硬件优先 / `0x01` 强制软件 |
| `0x15A` | bootrom_wait | 2B | 倒计时 ms，默认 2000 |
| `0x15C` | fsi_wait | 2B | 同上，默认 1000 |
| `0x15E` | FSI 固件分区表 | 64B | FSI 固件偏移、大小、SHA-256 预期值 |
| `0x19E` | AP 启动介质 | 1B | `0xFF` 跟随 KEY / `0x00` SPI / `0x01` SD / `0x02` UART |
| `0x19F` | NPU 启动介质 | 1B | 同上 |
| `0x1A0` | AP 分区表 | 64B | AP 镜像偏移、大小、哈希 |
| `0x1E0` | NPU 分区表 | 64B | NPU 镜像偏移、大小、哈希 |
| `0x220–0xFFF` | Reserved | ~3.5KB | — |