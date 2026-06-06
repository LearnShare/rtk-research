# core-infrastructure — RTK Core 基础设施深度分析

> **研究日期**：2026-06-06
> **研究阶段**：P1.3
> **研究方法**：深度源码阅读

## 概述
分析 `src/core/` 目录下 14 个组件的设计，涵盖配置、追踪、过滤引擎、TOML DSL、错误恢复和遥测系统。

## 核心发现

1. **双轨过滤引擎**：Rust-native（`filter.rs`，语言感知）+ TOML DSL（`toml_filter.rs`，8 阶段管道）
2. **SQLite 追踪系统**：项目级 GLOB 匹配、90 天自动清理、TimedExecution RAII 计时器
3. **Tee 恢复机制**：失败命令保存原始输出到磁盘，防止 LLM 上下文丢失
4. **Telemetry 仅 23h 一次**：SHA-256 匿名化 + GDPR consent gate + 编译时可选
5. **单线程无 async 约束**：所有代码通过 `unsafe_code = "deny"` + `warnings = "deny"` 强制执行

## 详细分析

### 1. config.rs — 配置系统

三级配置加载：
```
config.toml  (~/.config/rtk/ 或 %APPDATA%/rtk/)
├── [tracking]     enabled, history_days (默认90), database_path
├── [display]      colors, emoji, max_width (默认120)
├── [filters]      ignore_dirs, ignore_files
├── [tee]          enabled, mode ("failures"|"all"), max_files, max_file_size
├── [telemetry]    enabled, consent_given, consent_date
├── [hooks]        exclude_commands, transparent_prefixes
└── [limits]       grep_max_results, status_max_files, passthrough_max_chars
```

关键设计：
- `HooksConfig.exclude_commands` — 用户可配置不被重写的命令（跨 `rtk init -g` 持久化）
- `transparent_prefixes` — Docker exec / direnv 等包装命令的透明前缀匹配
- `Config::load()` 默认创建带注释的模板文件

### 2. tracking.rs — 追踪系统

**数据库**：`~/.local/share/rtk/tracking.db`（SQLite bundled）

**Schema**（commands 表）：
```
| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | 自增 |
| command | TEXT | 原始命令 |
| rewritten_cmd | TEXT | RTK 重写后的命令 |
| input_tokens | INTEGER | 原始输出 token 数 |
| output_tokens | INTEGER | 过滤后 token 数 |
| saved_tokens | INTEGER | 节省的 token 数 |
| savings_pct | REAL | 节省百分比 |
| execution_time_ms | INTEGER | 执行耗时 |
| project_path | TEXT | 项目路径（GLOB 匹配） |
| timestamp | TEXT | ISO 8601 时间戳 |
```

**关键模式**：
- `TimedExecution::start()` — RAII 计时器，记录 start time
- `.track()/.track_passthrough()` — 记录到 SQLite
- 90 天轮换：`DELETE FROM commands WHERE timestamp < datetime('now', '-90 days')`
- 项目级过滤：`project_path GLOB 'C:\works\cc-test\*'`

### 3. utils.rs — 工具函数

| 函数 | 作用 | 实现 |
|------|------|------|
| `strip_ansi(text)` | 移除 ANSI 转义码 | lazy_static regex `\x1b\[[0-9;]*[a-zA-Z]` |
| `truncate(s, max_len)` | 截断字符串加... | char-aware，最小 3 字符 |
| `format_tokens(n)` | Token 数格式化 | K/M 后缀（"1.2M", "59.2K"） |
| `execute_command(cmd, args)` | 执行命令 | `Command::new().output()` + context |
| `resolved_command(name)` | 解析命令路径 | `which` crate → fallback to name |
| `exit_code_from_status` | 提取退出码 | Unix/Windows 兼容 |

### 4. filter.rs — 语言感知过滤引擎

```rust
pub enum FilterLevel { None, Minimal, Aggressive }
pub enum Language { Rust, Python, JavaScript, TypeScript, Go, C, Cpp, Java, Ruby, Shell, Data, Unknown }
pub trait FilterStrategy { fn filter(&self, content: &str, lang: &Language) -> String; }
```

**三级过滤**：
- `None`：原样输出
- `Minimal`：去除空行和纯注释行
- `Aggressive`：去除所有注释、doc comment、多余空行

**语言检测**：基于文件扩展名映射到 `Language` 枚举。

### 5. toml_filter.rs — TOML DSL 过滤引擎

**8 阶段管道**（按序执行）：
```
1. strip_ansi           → 移除 ANSI 转义码
2. replace              → 正则替换（链式，行级别）
3. match_output         → 全文匹配 → 短路返回消息
4. strip/keep_lines     → 按正则过滤行
5. truncate_lines_at    → 行截断到 N 字符
6. head/tail_lines      → 保留前 N 或后 N 行
7. max_lines            → 绝对行数上限
8. on_empty             → 结果为空时的兜底消息
```

**查找优先级链**：
```
.rtk/filters.toml (项目级 → 可提交 Git)
  → ~/.config/rtk/filters.toml (用户全局)
    → BUILTIN_TOML (嵌入式，60+ filters)
      → passthrough (无匹配)
```

**数据结构**：
```rust
struct TomlFilterDef {
    description: String,
    match_command: String,          // 正则匹配命令
    strip_ansi: bool,
    filter_stderr: bool,            // 捕获 stderr 一起过滤
    strip_lines_matching: Vec<Regex>,
    keep_lines_matching: Vec<Regex>,
    replace: Vec<ReplaceRule>,      // { pattern, replacement }
    match_output: Vec<MatchOutputRule>, // { pattern, message, unless }
    truncate_lines_at: Option<usize>,
    max_lines: Option<usize>,
    tail_lines: Option<usize>,
    on_empty: Option<String>,
}
```

环境变量控制：
- `RTK_NO_TOML=1` — 跳过 TOML 引擎
- `RTK_TOML_DEBUG=1` — 打印匹配信息和行数到 stderr

### 6. tee.rs — 失败输出恢复

**触发条件**：命令退出码非零 + 输出 >= 500 字节

**工作流程**：
1. 生成唯一文件名：`{timestamp}_{command_slug}_{hex}.log`
2. 写入 `~/.local/share/rtk/tee/`
3. 限制文件数（默认 20）和文件大小（默认 1MB）
4. 输出 hint 消息：`[rtk: raw logs saved to ~/...tee/filename.log]`

**配置覆盖**：
- `RTK_TEE_DIR` 环境变量
- `config.toml` → `[tee]` section

### 7. telemetry.rs — 匿名遥测

**安全保证**：
- 编译时可选（`RTK_TELEMETRY_URL` 环境变量在构建时嵌入）
- GDPR 要求显式同意确认（`consent_given: true`）
- SHA-256 匿名化机器 ID
- ping 间隔 23 小时（非 24h，避免固定时间聚合）
- 所有网络错误静默忽略（fire-and-forget）

**opt-out 检查顺序**：
1. 编译时 URL 未设置 → 跳过
2. `RTK_TELEMETRY_DISABLED=1` → 跳过
3. 配置 `consent_given != true` → 跳过
4. 配置 `enabled = false` → 跳过
5. 当日已 ping → 跳过

### 8. 其余组件速览

| 组件 | 功能 |
|------|------|
| `constants.rs` | 路径常量（DATA_DIR, CONFIG_TOML, HISTORY_DB） |
| `display_helpers.rs` | PeriodStats Trait 统一 DayStats/WeekStats/MonthStats 的表格渲染 |
| `runner.rs` | `rtk err` 和 `rtk test` 的通用执行器 |
| `stream.rs` | 流式命令输出处理 |
| `truncate.rs` | 智能截断（保持结构完整性） |
| `args_utils.rs` | 参数解析辅助 |
| `telemetry_cmd.rs` | `rtk telemetry` 子命令处理器 |

## 结论与洞见

1. **Core 是精心设计的共享基础设施**：14 个组件各司其职，无循环依赖
2. **TOML filter 是最被低估的组件**：60+ 内置 filter + 8 阶段管道 + 优先级链 = 强大的声明式扩展能力
3. **Tee 是防御性设计亮点**：LLM 世界的独特需求 — 输出丢失意味着需要重新执行命令
4. **Telemetry 的隐私保护是标杆**：编译时可选 + 显式同意 + SHA-256 + fire-and-forget
5. **Tracking 的 GLOB 匹配是实用创新**：解决 worktree 和不同项目下的追踪问题
