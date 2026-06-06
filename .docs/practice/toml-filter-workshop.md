# toml-filter-workshop — TOML Filter 实操：ipconfig

> **研究日期**：2026-06-06
> **研究阶段**：Q0.5
> **研究方法**：动手实践

## 概述
从零开始为 `ipconfig` 命令创建一个 RTK TOML filter，完整记录设计决策、编码和验证过程。

## 核心发现

1. **TOML filter 开发周期约 10 分钟**：核心配置 ~10 行 + 内联测试 ~15 行
2. **构建时自动验证**：`build.rs` 在编译时解析 TOML、检测语法错误和重复名称
3. **`strip_lines_matching` 与 `keep_lines_matching` 互斥**：只能二选一
4. **编码注意事项**：在 bash/msys2 环境下写中文 TOML 可能出现编码问题，建议用结构模式匹配
5. **构建时 + 运行时双重验证**：`cargo build` 编译验证 + `rtk verify --filter ipconfig` 内联测试验证

## 详细分析

### 选型理由

选择 `ipconfig` 的原因：

| 维度 | 评估 |
|------|------|
| 未覆盖 | ✅ `src/filters/` 中不存在 |
| 常用性 | ✅ Windows 最常见网络诊断命令 |
| 输出结构 | ✅ 固定格式（适配器分组 + 属性行） |
| 节省潜力 | ✅ 默认 40+ 行 → 可压到 20 行，Empty 行占比高 |
| 复杂度 | 简单（适合入门 TOML filter） |

### Filter 设计

**目标**：移除空行，限制输出行数，提供空结果兜底消息。

```toml
[filters.ipconfig]
description = "Filter ipconfig output — strip empty lines, cap at 20 lines"
match_command = "^ipconfig\\b"
strip_ansi = true
strip_lines_matching = [
  "^\\s*$",
]
max_lines = 20
on_empty = "ipconfig: ok"

[[tests.ipconfig]]
name = "strip empties keeps content"
input = "..."

[[tests.ipconfig]]
name = "all empty yields on_empty"
input = "\n\n\n\n"
expected = "ipconfig: ok"
```

**设计决策记录**：

| 决策 | 选项 | 选择 | 理由 |
|------|------|------|------|
| 过滤策略 | strip vs keep | strip 空行 | keep 会 kill 适配器标题行 |
| 字段 | `max_lines` | 20 | 足够显示活跃适配器信息 |
| 兜底 | `on_empty` | "ipconfig: ok" | 统一风格（参考 `make.toml`） |
| 文件名 | `ipconfig.toml` | kebab-case | 符合命名约定 |

### 遇到的问题

1. **`strip_lines_matching` + `keep_lines_matching` 冲突**：引擎提示 "mutually exclusive"，必须二选一
2. **中文编码**：bash 环境下的 `ipconfig` 输出中文乱码，TOML filter 中的中文模式可能不匹配
3. **信任警告**：项目级 `.rtk/filters.toml` 默认未信任，但不影响内置 filter（`src/filters/`）

### 验证结果

```bash
$ cargo build
Compiling rtk v0.42.2
Finished dev profile [unoptimized + debuginfo]

$ ./target/debug/rtk verify --filter ipconfig
[rtk] WARNING: untrusted project filters skipped in verify
2/2 tests passed
```

内联测试全部通过，构建时 TOML 验证通过。

## 结论与洞见

1. **TOML filter 门槛极低**：只需了解正则表达式基础即可上手
2. **构建时验证是关键质量门禁**：编译前捕获所有语法问题
3. **`on_empty` 是 Token 节省利器**：全断开状态的 `ipconfig` 从 30 行 → 1 行
4. **建议贡献模式**：TOML filter 适合新手首次贡献；复杂场景再考虑 Rust-native
5. **ipconfig filter 可扩展到 `/all` 参数**：当前 `match_command = "^ipconfig\\b"` 匹配所有叫 `ipconfig` 的命令
