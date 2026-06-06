# command-filter-comparison — RTK 命令过滤模块对比分析

> **研究日期**：2026-06-06
> **研究阶段**：P1.4
> **研究方法**：分类对比阅读

## 概述
对比分析 9 个生态系统、30+ 命令过滤模块的设计模式、过滤策略和代码量。

## 核心发现

1. **统一模块结构**：所有 `*_cmd.rs` 遵循 imports → Args → lazy_static! → run() → filter_*() → tests
2. **三种过滤实现级别**：简单 passthrough（npm）< 行级过滤（ls）< JSON 结构化过滤（ruff/cargo）< 完整状态机（git）
3. **TOML 过滤覆盖最广**：60+ 内置 TOML filter 覆盖了 Rust-native 未处理的边缘工具
4. **lazy_static! 是性能关键**：24 处使用，确保正则只编译一次
5. **git 是最复杂的 filter**：2847 行（vs npm 237 行），包含完整的 diff/stat 解析

## 详细分析

### 共通模块结构

```rust
// 1. 模块文档
//! Filters X output.

// 2. 导入
use crate::core::{utils, runner, tracking};
use anyhow::Result;
use lazy_static::lazy_static;
use regex::Regex;

// 3. 类型定义
pub struct Args { ... }

// 4. 静态正则
lazy_static! { static ref RE: Regex = ...; }

// 5. 公共入口
pub fn run(args: &[String], verbose: u8) -> Result<i32> { ... }

// 6. 私有过滤函数
fn filter_*(input: &str) -> String { ... }

// 7. 内联测试
#[cfg(test)] mod tests { ... }
```

### 生态系统对比

| 生态 | 模块 | 代码量 | 过滤策略 | 实现方式 | 预估节省 |
|------|------|--------|----------|----------|----------|
| **git** | `git.rs` | 2847 行 | 状态机解析 diff/stat，压缩所有子命令 | Rust-native | 75-92% |
| **rust** | `cargo_cmd.rs` | 2216 行 | JSON 解析 test results，仅保存 failures | Rust-native | 90%+ |
| **go** | `go_cmd.rs` | 1134 行 | JSON streaming parse，errors-only | Rust-native | 90% |
| **docker/k8s** | `container.rs` | 982 行 | 表格压缩，关键字段保留 | Rust-native | 60-80% |
| **system/ls** | `ls.rs` | 715 行 | 树形重组，目录级聚合 | Rust-native | 80% |
| **python/ruff** | `ruff_cmd.rs` | 456 行 | JSON 结构化解析，规则分组 | Rust-native | 80% |
| **js/npm** | `npm_cmd.rs` | 237 行 | 智能路由 + passthrough | Rust-native | 0-70% |
| **filters/** | 60+ TOML | ~10-30 行/个 | 行级正则过滤 + 截断 | TOML DSL | 60-90% |

### TOML vs Rust-native 决策标准

| 场景 | 选择 | 原因 |
|------|------|------|
| 输出需要结构化解析（JSON/diff） | Rust-native | TOML 只能做行级正则 |
| 需要按文件/规则分组聚合 | Rust-native | 需要状态机 |
| 简单行级过滤 + 截断 | TOML DSL | 声明式，无需编译 |
| 长尾小众工具 | TOML DSL | 60+ 内置覆盖 |
| 用户自定义项目规则 | `.rtk/filters.toml` | 可提交 Git |

### 过滤策略矩阵

| 策略 | 代表命令 | 技术手段 |
|------|----------|----------|
| **Smart Filtering** | `rtk git status` | 去除 "nothing to commit" 之外的冗余输出 |
| **Grouping** | `rtk ruff check` | 按规则 code 分组：`F401 x3, E501 x2` |
| **Truncation** | `rtk cargo build` | 只保留 error/warning，丢弃 "Compiling" 行 |
| **Deduplication** | `rtk docker logs` | 重复行折叠：`line repeated 42x` |

### Passthrough 机制

所有 filter 模块通过 `Other(Vec<OsString>)` 变体实现：
```rust
enum CargoCommands {
    Build { args },
    Test { args },
    ...
    Other(Vec<OsString>),  // catch-all passthrough
}
```

Passthrough 仍然追踪但不过滤：
- stdout/stderr 通过 `Stdio::inherit()` 流式输出
- `timer.track_passthrough()` 记录为 0% 节省
- 用户不受影响 — 命令行为完全不变

## 结论与洞见

1. **代码量跨度极大**：237 行 (npm) 到 2847 行 (git)，反映过滤复杂度差距
2. **JSON 是结构化过滤的关键**：ruff、cargo test、go test 都利用 `--json` 或 `--output-format=json` 标志
3. **TOML 填充了长尾**：60+ TOML filter 覆盖了 Rust-native 不值得实现的工具
4. **passthrough 是安全网**：每个子命令枚举都保留 `Other` 变体
