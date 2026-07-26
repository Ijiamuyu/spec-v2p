# V2P License Header 规范

所有 V2P 项目新创建的代码文件必须包含标准 License 头，以声明版权与许可证信息。

## 许可证

V2P 项目采用 **GNU General Public License v2**（项目根目录 `LICENSE` 文件），SPDX 标识 `GPL-2.0-only`。
QEMU 相关代码遵循 QEMU 上游许可证（GPL-2.0-only），见 QEMU `COPYING` 文件。

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

## 模板

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

1. 从对应模板复制 header，追加通用声明（注释符对齐）
2. 将 `<简要描述该文件功能>` 替换为实际描述（英文，简洁）
3. 如有多个作者/贡献者，可增加 `Copyright (C) <year> <Author>` 行
4. **不可修改** `SPDX-License-Identifier` 和 GPL 声明正文
5. 自动生成的文件仅标注 `SPDX-License-Identifier: GPL-2.0-only` 即可
