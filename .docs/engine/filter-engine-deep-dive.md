# filter-engine-deep-dive — RTK 过滤引擎深度解剖

> **研究日期**：2026-06-06
> **研究阶段**：P2.1
> **研究方法**：源码追踪 + 分类对比

## 概述
深入分析 RTK 双轨过滤引擎的设计：Rust-native（语言感知、状态机）、TOML DSL（8 阶段声明式管道）、lazy_static 正则策略和四种压缩策略的实现。

## 核心发现

1. **双轨设计各司其职**：Rust-native 负责复杂解析（JSON/结构化输出），TOML DSL 负责简单行级过滤
2. **四种压缩策略贯穿全部 filter**：Smart Filtering → Grouping → Truncation → Deduplication，每种策略在不同 filter 中各有侧重
3. **lazy_static! 是性能基石**：24 处使用，确保正则一次性编译，避免热路径重复编译
4. **stream.rs 提供统一流式过滤**：LineHandler + LineStreamFilter 模式抽象了行级别处理
5. **truncate.rs 全局截断约束**：4 种截断量级（ERRORS=20, WARNINGS=10, LIST=20, INVENTORY=50）统一管控

## 详细分析

### 1. Rust-native 过滤引擎架构

#### 入口模式

所有 Rust-native filter 遵循统一入口模式：

```rust
pub fn run(args: &[String], verbose: u8) -> Result<i32> {
    let timer = tracking::TimedExecution::start();

    // 1. 执行原始命令
    let output = execute_command("cmd", &cmd_args)?;

    // 2. 应用过滤（可能使用 stream.rs 的流式处理）
    let filtered = filter_output(&output.stdout);

    // 3. 打印过滤结果
    print!("{}", filtered);

    // 4. 处理退出码
    if !output.status.success() {
        exit(output.status.code().unwrap_or(1));
    }

    // 5. 记录追踪
    timer.track("raw cmd", "rtk cmd", &output.stdout, &filtered);
    Ok(0)
}
```

#### 三种过滤级别（filter.rs）

`src/core/filter.rs` 提供基于语言感知的通用过滤：

| 级别 | 行为 | 用于 |
|------|------|------|
| `None` | 原样输出 | 通用 passthrough |
| `Minimal` | 去除空行和纯注释行 | 通用的快速清理 |
| `Aggressive` | 去除所有注释、doc comment、空行 | 仅保留代码骨架（函数签名 + 结构体定义） |

语言检测：基于文件扩展名映射到 `Language` 枚举（13 种语言 + Data/Unknown），支持注释模式匹配。

#### Git filter 状态机（最复杂的 Rust-native 过滤器）

`git.rs`（2847 行）是 RTK 最复杂的 filter，状态机处理多子命令：

```
GitCommand enum → run() → match subcommand
├── Diff   → build_diff_command() → filter_diff_output()
│   ├── compute_diff() → tokenized diff stat
│   └── condense_unified_diff() → io::stdin() 管道处理
├── Log    → build_log_command() → stream 过滤
│   ├── hash + message + relative time + author
│   └── `-vv` 模式下展开 body
├── Status → build_status_command() → "--porcelain -b"
│   ├── 使用 git C 语言环境 (LC_ALL=C) 确保英文解析
│   ├── 按 staged/unstaged/untracked 分组
│   └── 长路径名截断
├── Commit → 执行 → "ok <short_hash>"
├── Push   → 执行 → "ok <branch> [...]"
└── Stash  → list/show/pop/apply/drop + passthrough
```

关键设计：
- `git_cmd_c_locale()` 使用 `LC_ALL=C` 确保 git 输出格式稳定
- `--porcelain -b` 模式避免 git status 的本地化差异
- 每个子命令都有独立的 `filter_*` 函数

### 2. TOML DSL 8 阶段管道

声明式过滤管道（`toml_filter.rs`）：

```
输入命令输出
    │
    ▼
Stage 1: strip_ansi           ──── 移除 \x1b[...m ANSI 序列
    │
    ▼
Stage 2: replace              ──── 链式正则替换（行级别，支持 $1 回溯引用）
    │
    ▼
Stage 3: match_output         ──── 全文正则匹配 → 短路返回预设消息
    │                              （可设 unless 条件防止吞错误）
    ▼
Stage 4: strip/keep_lines     ──── RegexSet 批处理过滤行
    │
    ▼
Stage 5: truncate_lines_at    ──── 每行截断到 N 字符
    │
    ▼
Stage 6: head/tail_lines      ──── 保留前 N 或后 N 行
    │
    ▼
Stage 7: max_lines            ──── 绝对行数上限（最终兜底）
    │
    ▼
Stage 8: on_empty             ──── 结果为空时输出预设消息
    │
    ▼
   打印输出
```

**实现细节**：
- `CompiledFilter` 结构体将 TOML 配置编译为预编译的正则（`Regex`/`RegexSet`）
- `LineFilter` 枚举：`None | Strip(RegexSet) | Keep(RegexSet)` 两种过滤模式
- `RegexSet` 单次扫描匹配所有模式，比逐个 regex 匹配高效

### 3. Stream 流式过滤（stream.rs）

`src/core/stream.rs` 提供行级别流式处理抽象：

```rust
pub trait LineHandler {
    fn handle(&mut self, line: &str) -> FilterMode;
}

pub enum FilterMode { Keep, Skip }

pub struct LineStreamFilter<H: LineHandler> { ... }
```

使用模式：
- 实现 `LineHandler` trait 定义行过滤逻辑
- `LineStreamFilter` 包装后自动处理
- 用于 git log、cargo build 等需要流式输出场景

### 4. 四种压缩策略实现

| 策略 | 技术手段 | 代表 filter | 实现文件 |
|------|----------|-------------|----------|
| **Smart Filtering** | 正则匹配删除冗余行（ANSI码、注释、空行） | `filter.rs`, `strip_lines_matching` | `core/filter.rs`, `core/toml_filter.rs` |
| **Grouping** | 按规则/文件/目录聚合计数 | `ruff_cmd.rs`（规则分组），`ls.rs`（目录树） | `cmds/python/ruff_cmd.rs`, `cmds/system/ls.rs` |
| **Truncation** | 保留 N 行/字符 + "..." 后缀 | `truncate.rs`，`CAP_WARNINGS=10` | `core/truncate.rs` |
| **Deduplication** | 多行合并为 `repeated Nx` | `log_cmd.rs`, `docker logs` | `cmds/system/log_cmd.rs` |

#### Truncation 量级系统（truncate.rs）

```rust
// 全局截断常量（所有 filter 共享）
pub const CAP_ERRORS: usize = 20;      // 错误：最高信号密度
pub const CAP_WARNINGS: usize = 10;    // 警告：密度较低
pub const CAP_LIST: usize = 20;        // 列表：每行一个
pub const CAP_INVENTORY: usize = 50;   // 清单：穷举查询
pub const fn reduced(cap: usize, by: usize) -> usize;  // 安全缩减
```

### 5. lazy_static! 正则策略

RTK 代码库中 24 处 `lazy_static!` 使用：

```
src/cmds/dotnet/binlog.rs          1   — TRX 解析
src/cmds/cloud/aws_cmd.rs          1   — JSON 字段过滤
src/cmds/cloud/psql_cmd.rs         1   — 表格边框剥离
src/cmds/git/gh_cmd.rs             1   — PR URL 解析
src/cmds/git/glab_cmd.rs           1   — MR 信息提取
src/cmds/git/gt_cmd.rs             1   — 栈式 PR 解析
src/cmds/jvm/gradlew_cmd.rs        5   — Gradle 构建行分组
src/cmds/js/playwright_cmd.rs      2   — 测试结果解析
src/cmds/js/next_cmd.rs            2   — Next.js 构建输出
src/cmds/js/vitest_cmd.rs          1   — 测试报告
src/cmds/js/tsc_cmd.rs             1   — TypeScript 错误
src/cmds/ruby/rake_cmd.rs          1   — Rake 测试
src/cmds/python/mypy_cmd.rs        1   — 错误分组
src/cmds/ruby/rspec_cmd.rs         1   — RSpec 测试
src/cmds/system/grep_cmd.rs        1   — Grep 输出格式化
src/cmds/system/ls.rs              1   — 目录树解析
src/cmds/system/log_cmd.rs         1   — 日志去重
src/cmds/rust/runner.rs            1   — 测试 runner
```

**模式**：
```rust
lazy_static! {
    static ref RE: Regex = Regex::new(r"^pattern$").unwrap();
}
```

### 6. 过滤管线对比：简单 vs 中等 vs 复杂

| 维度 | npm_cmd.rs (简单) | ruff_cmd.rs (中等) | git.rs (复杂) |
|------|-------------------|-------------------|---------------|
| 代码量 | 237 行 | 456 行 | 2847 行 |
| 过滤技术 | 命令路由 + passthrough | `--output-format=json` → serde 反序列化 → 按 code 分组 | 状态机 + stream 过滤 + git porcelain 解析 |
| 外部依赖 | `resolved_command` | `serde::Deserialize` | `stream.rs`, `args_utils` |
| 节省率 | 0-70% | 80% | 75-92% |
| 测试 | 1 个子命令列表常量 | JSON 反序列化测试 | 快照测试 + Clap 参数测试 |

## 结论与洞见

1. **双轨策略有效**：Rust-native 处理复杂结构化输出，TOML DSL 覆盖 60+ 长尾工具
2. **Git filter 是架构标杆**：状态机 + locale 隔离 + porcelain 模式 = CLI 过滤器设计典范
3. **Stream 抽象待完善**：LineHandler trait 设计优雅但只在少数 filter 中使用
4. **Truncation 量级系统可配置化**：目前是编译时常量，未来可考虑 config.toml 覆盖
5. **lazy_static 向 OnceLock 迁移**：Rust 1.70+ 已稳定 OnceLock，RTK 可考虑逐步迁移
