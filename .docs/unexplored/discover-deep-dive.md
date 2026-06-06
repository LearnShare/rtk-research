# discover-deep-dive — 发现引擎深度分析

> **研究日期**：2026-06-06
> **研究阶段**：Q0.4
> **研究方法**：源码分析

## 概述
分析 `src/discover/` 模块的完整架构：74 条命令重写规则、shell 令牌化算法、会话历史扫描引擎和发现报告生成。

## 核心发现

1. **74 条重写规则**（`rules.rs`）：覆盖 Git/GitHub/Cargo/npm/pnpm/Files/Infra/Build/Test 等 12+ 类别
2. **`rtk discover` 是逆向发现工具**：扫描 Claude Code 历史，找出本应通过 RTK 但未使用的命令
3. **`rtk rewrite` 使用相同 registry**：`discover/registry.rs` 是 RTK 命令重写的单一真相源
4. **复合命令拆分**：`lexer.rs` 的 `split_on_operators()` 按 `&&`, `|`, `;` 拆分命令链
5. **18 个忽略前缀 + 10 个精确忽略**：shell 内建命令（`cd`, `echo`, `for`, `if`）不重写

## 详细分析

### 1. 模块结构

```
src/discover/
├── mod.rs          ← run() 入口：扫描 → 分类 → 聚合 → 报告
├── lexer.rs        ← shell 令牌化：tokenize(), split_on_operators()
├── rules.rs        ← 74 条重写规则 + 忽略列表
├── registry.rs     ← rewrite_command() 单一真相源
├── provider.rs     ← ClaudeProvider: 读取会话历史文件
└── report.rs       ← 报告格式 + RtkStatus 枚举
```

### 2. 74 条重写规则分类

| 类别 | 规则数 | 示例 |
|------|--------|------|
| Git 系列 | 8 | `git status`, `gh pr`, `glab mr`, `gt log` |
| Cargo/Rust | 8 | `cargo build`, `cargo test`, `cargo clippy` |
| JS 系列 | 14 | `npm run`, `pnpm install`, `npx tsc`, `vitest` |
| Python | 5 | `ruff check`, `pytest`, `mypy`, `pip list` |
| Go | 5 | `go test`, `go build`, `golangci-lint run` |
| Ruby | 3 | `rspec`, `rubocop`, `rake test` |
| .NET | 4 | `dotnet build`, `dotnet test` |
| Files/System | 8 | `cat`, `rg`, `ls`, `tree`, `find`, `env` |
| Build/Infra | 6 | `docker ps`, `kubectl get`, `terraform plan` |
| 其他 | 13 | `curl`, `diff`, `log`, `json`, `make` |

每条规则定义：

```rust
pub struct RtkRule {
    pattern: &'static str,          // 匹配正则
    rtk_cmd: &'static str,          // RTK 命令前缀
    rewrite_prefixes: &'static [&'static str],  // 可重写的原始命令前缀
    category: &'static str,         // 分类标签
    savings_pct: f64,               // 预估节省率
    subcmd_savings: &[(&str, f64)], // 子命令粒度节省
    subcmd_status: &[(&str, RtkStatus)],  // 子命令 RTK 状态
}
```

### 3. 忽略列表

```rust
pub const IGNORED_PREFIXES: &[&str] = &[
    "cd ", "echo ", "printf ", "export ", "source ", ". ",
    "sed ", "python3 -c", "node -e", "ruby -e",
    "rtk ", "pwd", "then", "else", "do", "for", "while", "if", "case",
];

pub const IGNORED_EXACT: &[&str] = &[
    "cd", "echo", "true", "false", "wait", "pwd", "bash", "sh", "fi", "done",
];
```

`rtk ` 在忽略前缀中 — 已使用 RTK 的命令不计入"发现"结果。

### 4. 令牌化算法（lexer.rs）

```rust
pub fn tokenize(input: &str) -> Vec<ParsedToken>  // 完整令牌化
pub fn shell_split(input: &str) -> Vec<String>     // 引号感知的简单拆分
pub fn split_on_operators(input: &str) -> Vec<String>  // && | ; 拆分
pub fn contains_unattestable_construct(cmd: &str) -> bool  // 检测不可审核结构
```

**split_on_operators 拆分算法**：
```
"cargo test && git push"
  → ["cargo test ", " git push"]
  → 分别重写为 "rtk cargo test" + "rtk git push"
  → 组合为 "rtk cargo test && rtk git push"
```

**不可审核构造**（`contains_unattestable_construct`）：
- 子 shell：`$(...)`
- 反引号：`` `...` ``
- 复杂重定向：`>&`, `<>`
- 原始命令直接 passthrough，防止注入

### 5. registry.rewrite_command — 重写核心

```rust
pub fn rewrite_command(cmd: &str, excluded: &[String], transparent_prefixes: &[String]) -> Option<String>
```

流程：
1. 检查 `exclude_commands`（用户配置的不重写命令）
2. 检测透明前缀（`docker exec mycontainer` → 剥离并保存前缀）
3. `split_on_operators()` 拆分复合命令
4. 每段 `classify_command()` → `Supported/Unsupported/Ignored`
5. 支持的命令用 `rtk_cmd` 替换原始前缀（支持 `subcmd_savings` 精度控制）
6. 重组合复合命令 → `"rtk cargo test && rtk git push"`

### 6. discover run() 扫描流程

```
discover::run()
    │
    ├─ ClaudeProvider.discover_sessions()
    │   ├─ ~/.claude/projects/*/ 扫描目录
    │   ├─ 按项目过滤 + 时间窗过滤
    │   └─ 返回 session 文件路径列表
    │
    ├─ provider.extract_commands(session_path)
    │   ├─ 读取 session JSONL 文件
    │   ├─ 提取 tool_call → Bash + tool_result
    │   └─ 计算 output_len, is_error
    │
    ├─ 对每条命令:
    │   ├─ split_command_chain() 拆分 &&
    │   ├─ strip_disabled_prefix() 检查 RTK_DISABLED
    │   ├─ classify_command() 匹配规则
    │   ├─ 支持的 → supported_map 聚集
    │   ├─ 不支持的 → unsupported_map 聚集
    │   └─ 已 rtk 或忽略 → 跳过
    │
    └─ 输出报告:
        ├─ 已发现 N 个支持但未使用 RTK 的命令
        ├─ 预估可节省 N tokens
        └─ 按节省量排序的建议列表
```

### 7. 分类推理

```rust
pub enum Classification {
    Supported { rtk_equivalent, category, estimated_savings_pct, status },
    Unsupported { base_command },
    Ignored,
}
```

Token 预算是动态计算的：
- 如果有 `output_len`（来自 session 记录）：`output_len / 4` 估算
- 如果没有：基于类别的平均值（`category_avg_tokens()`）

### 8. Agent 集成状态检测

```rust
pub struct AgentIntegrationStatus {
    pub cursor_hook_installed: bool,
    pub hermes_plugin_installed: bool,
    pub copilot_hook_installed: bool,
}
```

`report.rs` 中的 `detect()` 检查各种 agent 的 hook 文件是否存在。

## 结论与洞见

1. **74 条规则覆盖全面**：所有主要生态系统的常用命令都有对应规则
2. **复合命令拆分是关键能力**：`&&`, `|`, `;` 的正确处理使 RTK 不丢失链式命令
3. **不可审核构造的静默 passthrough 是安全设计**：复杂 shell 语法不尝试解析
4. **discover 和 rewrite 共享 registry**：单一的 `rewrite_command()` 函数服务 Hook 重写和发现扫描
5. **Token 预算是启发式**：没有真实 output_len 时使用类别平均值，可能不准确
