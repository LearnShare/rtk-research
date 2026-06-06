# rust-filter-workshop — Rust Filter 实操：netstat

> **研究日期**：2026-06-06
> **研究阶段**：Q0.6
> **研究方法**：动手实践（完整贡献流程）

## 概述
从零开始为 `netstat -an` 实现一个 Rust-native filter，完整经历模块创建 → 注册到 main.rs → 编译 → 测试 → 实测全流程。

## 核心发现

1. **完整流程约 30 分钟**：含设计、编码、注册、编译调试、测试
2. **需要修改 4 处**：新建 `_cmd.rs` + main.rs import + Commands 枚举 + routing + is_operational_command
3. **`automod` 自动发现模块**：将文件放入 `src/cmds/system/` 即自动注册为子模块
4. **`tracking::TimedExecution` 是标准入口模式**：每次 filter 调用都需记录追踪
5. **`warnings = "deny"` 要求严格**：所有警告都是错误，需清理未使用变量等

## 详细分析

### Filter 设计与效果

**目标**：`netstat -an` 默认输出 160+ 行，按状态分组压缩

**输入示例**（164 行）：
```
Active Connections

  Proto  Local Address          Foreign Address        State
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING
  TCP    10.0.1.63:22000        10.0.4.36:22000        ESTABLISHED
  ...
```

**输出效果**（~15 行，节省 ~90%）：
```
ESTABLISHED (45):
   TCP  10.0.1.63:22000 → 10.0.4.36:22000
   TCP  10.0.1.63:49411 → 98.66.133.185:443
   ... +40 more

LISTENING (42):
   TCP  0.0.0.0:80 → 0.0.0.0:0
   TCP  0.0.0.0:135 → 0.0.0.0:0
   ... +37 more

TIME_WAIT (25):
   TCP  127.0.0.1:80 → 127.0.0.1:56718
   ... +20 more
```

**Token 节省**：
- 原始：~164 行 ~2000 tokens
- 过滤后：~15 行 ~150 tokens
- 节省率：~92%

### 实现步骤

#### 步骤 1：创建模块文件

```rust
// src/cmds/system/netstat_cmd.rs

use crate::core::tracking;
use anyhow::{Context, Result};
use std::collections::BTreeMap;
use std::process::{Command, Stdio};

pub fn run(args: &[String], _verbose: u8) -> Result<i32> {
    let timer = tracking::TimedExecution::start();
    let output = Command::new("netstat")
        .args(args)
        .stdout(Stdio::piped())
        .stderr(Stdio::inherit())
        .output()?;

    let raw = String::from_utf8_lossy(&output.stdout).to_string();
    let filtered = filter_netstat_output(&raw);
    print!("{}", filtered);

    timer.track("netstat", "rtk netstat", &raw, &filtered);
    Ok(output.status.code().unwrap_or(1))
}
```

#### 步骤 2：过滤逻辑

按状态（LISTENING/ESTABLISHED/TIME_WAIT）使用 `BTreeMap` 分组：

```rust
fn filter_netstat_output(raw: &str) -> String {
    // 1. 解析每一行，筛选 TCP/UDP 连接
    // 2. 按状态分组
    // 3. 每个状态最多显示 5 条
    // 4. 输出分组标题 + 连接列表
}
```

#### 步骤 3：注册到 main.rs

需修改 4 处：

```
1. Import:
   use cmds::system::{..., netstat_cmd};

2. Commands 枚举:
   Netstat {
       #[arg(trailing_var_arg = true, allow_hyphen_values = true)]
       args: Vec<String>,
   }

3. Routing (run_cli):
   Commands::Netstat { args } => netstat_cmd::run(&args, cli.verbose)?,

4. 安全白名单 (is_operational_command):
   | Commands::Netstat { .. }
```

#### 步骤 4：测试

```bash
$ cargo test -- netstat_cmd
6 passed, 2072 filtered out
```

#### 步骤 5：实测

```bash
$ rtk netstat -an
ESTABLISHED (45):   # 164 行 → 15 行
LISTENING (42):
TIME_WAIT (25):
```

### 遇到的问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 编译报错 `mismatched closing delimiter` | `matches!` 宏缺少闭合 `)` | 检查 `is_operational_command` 语法 |
| `warnings = "deny"` 导致编译失败 | `verbose` 参数未使用 | 加 `_` 前缀：`_verbose` |
| 使用 `runner::run_filtered` 无输出 | `resolved_command("netstat")` 在 Windows 上路径问题 | 改用 `Command::new("netstat")` 直接 |
| 测试需考虑 Windows CRLF | `\r\n` 行尾 | `trim()` + `split_whitespace()` 自动处理 |

### 与 TOML filter 对比

| 维度 | TOML filter (ipconfig) | Rust filter (netstat) |
|------|----------------------|----------------------|
| 代码量 | ~15 行 TOML | ~100 行 Rust |
| 实现时间 | ~10 分钟 | ~30 分钟 |
| 注册步骤 | 仅创建文件 | 4 处修改 + 编译 |
| 过滤能力 | 行级正则 | 状态机/聚合/分组 |
| 复杂度 | 低 | 中 |
| 适合场景 | 行过滤/截断 | 结构化输出/聚合 |
| 贡献门槛 | 低（无 Rust 要求） | 中（Rust 基础） |

## 结论与洞见

1. **Rust filter 开发流程标准化**：所有 `_cmd.rs` 遵循相同模式，易于参考
2. **`automod` 降低配置负担**：只需要将文件放在正确目录即可，无需手动 `mod`
3. **warning-as-error 是好实践**：确保代码质量，但初期调试时需注意
4. **`Command::new` 比 `resolved_command` 更直接**：对于已知系统命令（netstat）可以直接使用
5. **6 个测试覆盖所有功能路径**：正常解析、有标题、空输入、无 TCP、分组、限制
