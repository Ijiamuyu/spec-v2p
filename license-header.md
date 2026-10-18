# V2P License 声明规范

V2P 项目的许可信息在**仓级**声明：每个仓库根目录放 `LICENSE` 全文，并在 `README.md`
用一小节注明 SPDX 标识与适用范围。软件代码（C/C++、Rust、Python、Shell）的源文件
**建议**再加文件头，便于单文件独立分发时明确许可；RTL、构建/链接脚本与自动生成文件
不要求。

## 许可证

V2P 项目采用 **GNU General Public License v2**（项目根目录 `LICENSE` 文件），SPDX 标识 `GPL-2.0-only`。
QEMU 相关代码遵循 QEMU 上游许可证（GPL-2.0-only），见 QEMU `COPYING` 文件。

## 声明方式

| 层次 | 位置 | 要求 |
|------|------|------|
| **仓级** | 根目录 `LICENSE` + `README.md` 许可小节 | **必须**，每个仓都要有 |
| **文件级** | 软件代码源文件头部（见下方模板） | **建议**（C/C++、Rust、Python、Shell） |
| 其他 | RTL、Makefile、链接脚本、自动生成文件 | 仓级声明即可，不加文件头 |

仓级 README 小节的写法（照抄，按实际仓替换「范围」表内容）：

```markdown
## 许可

本仓采用 **GPL-2.0-only**（SPDX 标识），全文见根目录 [`LICENSE`](LICENSE)。
许可在**仓级**声明（`LICENSE` + 本节），不逐文件加头部注释。

| 范围 | 许可 / 来源 |
|------|------------|
| 本仓代码 | GPL-2.0-only |
| 三方目录（如 `third_party/`） | 不入库，按各自上游许可 |
```

## 通用声明

以下 GPL 声明为所有模板共用，各模板仅展示注释风格和 header 部分差异：

```
This program is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License version 2 as
published by the Free Software Foundation.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License along
with this program; if not, see <http://www.gnu.org/licenses/>.
```

## 软件代码文件头模板（建议）

以下模板给软件代码源文件使用，属建议而非必须；使用时不得修改 `SPDX-License-Identifier`
与 GPL 声明正文。

### C / C++（.c / .h）

```c
/*
 * SPDX-License-Identifier: GPL-2.0-only
 *
 * V2P - <简要描述该文件功能>
 *
 * Copyright (C) 2026 V2P Project
 *
 * <通用声明，行首加 " * ">
 */
```

### Rust（.rs）

```rust
// SPDX-License-Identifier: GPL-2.0-only
//
// V2P - <简要描述该文件功能>
//
// Copyright (C) 2026 V2P Project
//
// <通用声明，行首加 "// ">
```

### Python（.py）

```python
# SPDX-License-Identifier: GPL-2.0-only
#
# V2P - <简要描述该文件功能>
#
# Copyright (C) 2026 V2P Project
#
# <通用声明，行首加 "# ">
```

### Shell（.sh）

```bash
# SPDX-License-Identifier: GPL-2.0-only
#
# V2P - <简要描述该文件功能>
#
# Copyright (C) 2026 V2P Project
#
# <通用声明，行首加 "# ">
```

### 配置文件（Makefile / meson.build / TOML / YAML）

格式同 Shell 模板，使用对应注释符（`#`）。

## 使用说明

1. **仓级（必须）**：确保根目录有 `LICENSE`，且在 `README.md` 写「许可」小节（见上方写法）
2. **文件级（软件代码，建议）**：从对应模板复制 header，追加通用声明（注释符对齐）；
   将 `<简要描述该文件功能>` 替换为实际描述（英文，简洁）
3. 如有多个作者/贡献者，可增加 `Copyright (C) <year> <Author>` 行
4. **不可修改** `SPDX-License-Identifier` 和 GPL 声明正文
5. 自动生成的文件：仅标注 `SPDX-License-Identifier: GPL-2.0-only` 即可（放在生成模板里）
6. 配置文件模板（Makefile / TOML / YAML 等）仅在需要时使用，非必须
