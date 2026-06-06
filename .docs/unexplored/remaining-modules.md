# remaining-modules — 剩余组件速览

> **研究日期**：2026-06-06
> **研究阶段**：Q3
> **研究方法**：源码分析

## 概述
覆盖 v1 和 Q0 中未深入触及的剩余模块：args_utils、runner、ccusage、hooks 剩余组件、analytics 剩余组件。

## args_utils — 参数还原工具

```rust
// 解决 Clap 的 trailing_var_arg 吃掉 "--" 的问题
pub fn restore_double_dash(parsed_args: &[String]) -> Vec<String>;
```

**问题**：Clap 的 `trailing_var_arg = true` 在处理 `git diff -- file` 时会吃掉 `--`，导致子进程收到 `["file"]` 而不是 `["--", "file"]`。

**解决方案**：对比 `parsed_args` 和 `raw_args`（`std::env::args()`）中的 `--` 数量差异，还原被吞掉的 `--` 令牌。

---

## runner — 通用命令执行骨架

提供统一的命令执行 + 过滤骨架，被 `wc_cmd.rs` 等 filter 使用：

```rust
pub fn run_filtered<F>(cmd, tool_name, args_display, filter_fn, opts) -> Result<i32>
pub fn run_filtered_with_exit<F>(cmd, tool_name, args_display, filter_fn, opts) -> Result<i32>
pub fn run_passthrough(tool, args, verbose) -> Result<i32>
```

**RunOptions**：
- `tee_label` — 失败时保存原始输出
- `filter_stdout_only` — 仅过滤 stdout
- `skip_filter_on_failure` — 失败时跳过过滤
- `inherit_stdin` — 转发 stdin 给子进程
- `no_trailing_newline` — 去除尾部换行

---

## ccause — Claude Code 经济学

`src/analytics/ccusage.rs` 提供与 `npx ccusage` 工具的集成：

```rust
pub struct CcusageMetrics {
    input_tokens: u64,
    output_tokens: u64,
    cache_creation_tokens: u64,
    cache_read_tokens: u64,
    total_tokens: u64,
    total_cost: f64,
}

pub fn fetch_ccusage_data(granularity, days_back) -> Result<Vec<CcusagePeriod>>;
pub fn fetch_ccusage_total(days_back) -> Result<CcusageMetrics>;
```

**功能**：调用 `npx ccusage` 获取 Claude Code API 使用数据，与 RTK 节省数据合并计算 ROI。

---

## hooks 剩余组件

| 组件 | 路径 | 职责 |
|------|------|------|
| `hook_cmd.rs` | `src/hooks/hook_cmd.rs` | 处理 Claude/Cursor/Copilot/Gemini 的 Hook JSON 输入 |
| `hook_check.rs` | `src/hooks/hook_check.rs` | 每日检查 Hook 是否过期/缺失 |
| `verify_cmd.rs` | `src/hooks/verify_cmd.rs` | `rtk verify` 运行时验证内联测试 |
| `hook_audit_cmd.rs` | `src/hooks/hook_audit_cmd.rs` | Hook 重写审计 |
| `permissions.rs` | `src/hooks/permissions.rs` | 权限规则引擎（Q1 已分析） |
| `constants.rs` | `src/hooks/constants.rs` | Agent 配置路径常量 |

**hook_cmd 流程**（Claude Code JSON 处理器）：
```
stdin → {"tool_call":{"name":"Bash","input":{"command":"git status"}}}
  → 提取 command 字段
  → rtk rewrite "git status"  (子进程)
  → exit code 0/1/2/3
  → 返回 {"response":{"command":"rtk git status"}} 或空
```

---

## analytics 剩余组件

| 组件 | 路径 | 职责 |
|------|------|------|
| `gain.rs` | `src/analytics/gain.rs` | v1 P2.3 已深入分析 |
| `cc_economics.rs` | `src/analytics/cc_economics.rs` | 结合 ccusage + RTK 节省计算 ROI |
| `ccusage.rs` | `src/analytics/ccusage.rs` | 本表已覆盖 |
| `session_cmd.rs` | `src/analytics/session_cmd.rs` | `rtk session` 命令 |

**cc_economics 计算模型**：
```
RTK 节省 = Σ(output_tokens_before - output_tokens_after)
CC 消费 = Σ(ccusage 报告的 input + output + cache)
ROI = RTK 节省 / CC 总消费
```

---

## cmds 剩余边缘 fiter

| 组件 | 路径 | 行数 | 特点 |
|------|------|------|------|
| `gradlew_cmd.rs` | `src/cmds/jvm/gradlew_cmd.rs` | 351 | 5 个 lazy_static!（最多） |
| `binlog.rs` | `src/cmds/dotnet/binlog.rs` | — | MSBuild 二进制日志解析 |
| `dotnet_trx.rs` | `src/cmds/dotnet/dotnet_trx.rs` | — | TRX 测试结果 XML 解析 |
| `psql_cmd.rs` | `src/cmds/cloud/psql_cmd.rs` | — | PostgreSQL 表格边框剥离 |

**gradlew 特殊之处**：Android Gradle wrapper 输出包含彩色 ANSI 码 + 丰富进度，filter 需要 5 个正则覆盖各种构建行格式。
