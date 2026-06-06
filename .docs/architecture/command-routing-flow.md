# command-routing-flow — RTK 入口与命令路由系统

> **研究日期**：2026-06-06
> **研究阶段**：P1.1
> **研究方法**：静态源码分析

## 概述
分析 RTK 入口文件 `src/main.rs`（3275 行）的架构设计，包括 CLI 解析、命令分发、fallback 链路和安全白名单机制。

## 核心发现

1. **双层路由架构**：Clap 解析（主路径）+ `run_fallback()`（兜底路径），确保 100% 命令可达
2. **安全白名单**：`is_operational_command()` 默认拒绝，新命令需显式加入才通过完整性校验
3. **TOML 过滤优先级**：fallback 中自动查找 TOML filter，比 bare passthrough 优先
4. **Meta 命令保护**：16 个 RTK 专属命令永不 fallback，确保参数错误能可见
5. **SIGPIPE 安全**：Unix 上重置 SIGPIPE 为默认，避免 panic=abort 时 SIGABRT

## 详细分析

### 1. Cli 结构体与全局 Flags

```
[Cli]
├── command: Commands    # 子命令枚举（主路由）
├── verbose: u8          # -v / -vv / -vvv（全局）
├── ultra_compact: bool  # --ultra-compact（ASCII 图标模式）
└── skip_env: bool       # --skip-env（跳过环境验证）
```

全局 flags 通过 `global = true` 机制传递到所有子命令。每个命令处理器接收 `cli.verbose` 和 `cli.ultra_compact` 参数。

### 2. Commands 枚举架构

Commands 枚举包含 **50+ 变体**，按功能分为 6 层：

| 层级 | 类型 | 示例 | 子枚举 |
|------|------|------|--------|
| 文件操作 | 直接路由 | `Ls`, `Tree`, `Read`, `Find`, `Wc` | 无 |
| Git 生态 | 嵌套子命令 | `Git` → `GitCommands` | `Diff/Log/Status/...` (11) |
| 语言生态 | 嵌套子命令 | `Cargo` → `CargoCommands` | `Build/Test/Clippy/...` (6) |
| 容器/云 | 嵌套子命令 | `Docker` → `DockerCommands` | `Ps/Images/Compose/...` |
| 分析统计 | 直接路由 | `Gain`, `Discover`, `Session` | 无 |
| Hook/配置 | 直接路由 | `Init`, `Rewrite`, `Hook`, `Verify` | 无 |

每个需要子命令处理的外部工具拥有独立的 `Other(Vec<OsString>)` 变体作为 passthrough。

### 3. run_fallback() — 三层回退链路

当 Clap 解析失败时，`run_fallback()` 提供三层回退：

```
run_fallback(parse_error)
│
├─ 空 args → 显示 Clap 错误并退出
├─ 是 RTK_META_COMMANDS → 显示 Clap 错误（永不 fallback）
│
├─ 路径 1: TOML filter 匹配
│   ├─ resolve_command(args[0])
│   ├─ execute → capture stdout/stderr
│   ├─ apply_filter(toml_filter, output)
│   ├─ 失败时 tee_and_hint() 保存原始输出
│   └─ timer.track() 记录到 SQLite
│
├─ 路径 2: No TOML match → passthrough
│   ├─ Stdio::inherit (流式输出)
│   └─ timer.track_passthrough()
│
└─ 路径 3: Command not found
    └─ eprintln !("[rtk: {}]", e); Ok(127)
```

关键设计决策：
- TOML 查找时使用 `file_name()` 提取 basename，支持绝对路径
- `RTK_NO_TOML=1` 可跳过 TOML 引擎
- `tee` 机制在命令失败时保存原始输出到 `~/.local/share/rtk/tee/`

### 4. is_operational_command() — 安全白名单

```rust
/// SECURITY: whitelist pattern — new commands are NOT integrity-checked
/// until explicitly added here. A forgotten command fails open (no check)
/// rather than creating false confidence about what's protected.
fn is_operational_command(cmd: &Commands) -> bool {
    matches!(cmd, Commands::Ls { .. } | Commands::Tree { .. } | ...)
}
```

**设计哲学**：fail-open（忘记添加 = 不校验），不产生虚假安全感。白名单保护 **42 个命令变体**，排除了 meta 命令（init/gain/verify/config 等）。

### 5. 执行流水线

```
main()
├─ SIGPIPE reset (unix only)
└─ run_cli()
    ├─ telemetry::maybe_ping()           # 每日一次
    ├─ Cli::try_parse()                  # Phase 1: PARSE
    │   └─ Err → run_fallback()
    ├─ hook_check::maybe_warn()          # 检查 hook 过时
    ├─ integrity::runtime_check()         # 操作命令完整性校验
    ├─ match command                     # Phase 2: ROUTE
    │   ├─ Git → git::run()
    │   ├─ Cargo → cargo_cmd::run()
    │   ├─ Gain → analytics::gain::run()
    │   └─ ...
    └─ Ok(exit_code) / Err(e)
```

### 6. RTK_META_COMMANDS 列表

16 个 RTK 专属命令，不应该 fallback 到 raw 执行：

```
gain, discover, learn, init, config, proxy, run, hook,
hook-audit, pipe, cc-economics, verify, trust, untrust, session, rewrite
```

这些命令在测试中验证：传入非法参数时 Clap 应返回解析错误，而非 fallback 到 shell。

### 7. 测试覆盖

main.rs 包含 ~500 行测试代码，覆盖：
- Git commit 各种参数格式（`-m`, `--message`, `-am`）
- 全局 options（`--no-pager`, `--no-optional-locks`）
- Meta 命令 fallback 防护
- shell_split 令牌化
- proxy 信号处理
- 集成测试（`#[ignore]`，需预编译二进制）

## 结论与洞见

1. **路由设计优雅**：Clap derive 模式 + `Other(Vec<OsString>)` 确保所有子命令可达
2. **fallback 是安全网**：TOML filter 查找 + passthrough 两级兜底，不丢命令
3. **安全优先**：白名单 fail-open 策略保守但合理
4. **可观测性强**：每个执行路径都有 timer.track() 记录
5. **3275 行 main.rs 偏长**：可考虑将各 Commands 子枚举拆分到独立模块
