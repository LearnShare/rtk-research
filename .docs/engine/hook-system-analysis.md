# hook-system-analysis — RTK Hook 系统与命令重写

> **研究日期**：2026-06-06
> **研究阶段**：P2.2
> **研究方法**：源码分析 + 实验验证

## 概述
分析 RTK Hook 系统的完整链路：从 `rtk init` 安装 → `PreToolUse` 拦截 → `rtk rewrite` 重写 → `rtk hook claude` 执行。涵盖 7+ AI 代理的适配架构、SHA-256 完整性校验和项目信任机制。

## 核心发现

1. **Hook 脚本是薄委托层**：所有重写逻辑在 Rust 二进制中（`src/discover/registry.rs`），Hook 只做 JSON 格式转换
2. **`rtk rewrite` 是单一真相源**：用四个 exit code（0/1/2/3）编码重写结果 + 权限决策
3. **四层安全模型**：操作命令白名单 + SHA-256 完整性校验 + 权限系统 + 项目信任机制
4. **复合命令智能拆分**：`&&` 和 `|` 链拆分为独立重写
5. **Auto-Rewrite vs Suggest 双模式**：全覆盖 vs 非侵入建议

## 详细分析

### 1. 安装流程（`rtk init`）

```
rtk init --global
├─ 1. 生成 hook 脚本内容（嵌入在二进制中）
│     hooks/init.rs → 调用常量 → 写入 ~/.claude/hooks/rtk-rewrite.sh
├─ 2. 计算并存储 SHA-256 校验和
│     hooks/integrity.rs → store_hash() → .rtk-hook.sha256
├─ 3. 写 RTK.md（slime 模式）
│     ~/.claude/RTK.md
├─ 4. 打补丁 settings.json
│     添加 PreToolUse hook: "rtk hook claude"
└─ 5. 配置 @RTK.md 引用
      ~/.claude/CLAUDE.md 添加 @RTK.md
```

### 2. 命令拦截流程

```
LLM Agent: "git status"
        │
        ▼
PreToolUse Hook (settings.json)
  ┌─────────────────────────┐
  │ "matcher": "Bash",      │
  │ "hooks": [{             │
  │   "type": "command",    │
  │   "command": "rtk hook claude"
  │ }]                      │
  └─────────┬───────────────┘
            │ stdin: { "tool_call": { "name": "Bash", "input": { "command": "git status" } } }
            ▼
rtk hook claude  # 处理 Claude Code JSON 格式
            │
            ▼
rtk rewrite "git status"  # 单一真相源
            │
            ▼
registry.rs  # 匹配规则 → 返回 "rtk git status"
            │
            ▼
exit code:
  0 → Hook 允许自动执行 (Allow)
  1 → 无重写，透传 (Passthrough)
  2 → 拒绝执行 (Deny)
  3 → 需要用户确认 (Ask)
```

### 3. `rtk rewrite` exit code 契约

这是整个 Hook 系统的核心设计，4 种退出码编码了重写决策 + 权限决策：

| Exit | Stdout | 含义 | Hook 行为 |
|------|--------|------|-----------|
| 0 | `rtk git status` | 重写 + 允许 | 自动替换命令 |
| 1 | (空) | 无 RTK 等价 | 原样透传 |
| 2 | (空) | 拒绝规则匹配 | 阻止执行 |
| 3 | `rtk git status` | 需要用户确认 | 重写 + 提示确认 |

### 4. 命令重写注册表（registry.rs）

重写规则在 `src/discover/rules.rs` 中定义（~80 条规则），使用 `RULES` 常量和 `lazy_static! RegexSet` 批量匹配：

```rust
// rules.rs 示例模式
lazy_static! {
    static ref REGEX_SET: RegexSet =
        RegexSet::new(RULES.iter().map(|r| r.pattern)).expect("...");
}
```

**重写分类**：

| 分类 | 示例 | 估算节省 |
|------|------|----------|
| Git | `git status` → `rtk git status` | 40-200 tokens |
| Cargo | `cargo test` → `rtk cargo test` | 150-500 tokens |
| Tests | `pytest` → `rtk pytest` | 800 tokens |
| Files | `ls -la` → `rtk ls` | 100 tokens |
| Build | `next build` → `rtk next build` | 300 tokens |
| Network | `curl` → `rtk curl` | 150 tokens |
| Package | `npm install` → `rtk npm install` | 150 tokens |

**复合命令处理**：
- `cargo test && git push` 被 `lexer.rs` 的 `split_on_operators()` 按 `&&`, `|`, `;` 拆分
- 每个段独立重写 → `rtk cargo test && rtk git push`
- 复合命令中 RTK 不支持的段保持原样

### 5. 二级重写路径 — @RTK.md

除了 Hook 重写，还有 `CLAUDE.md` 级别的重写指令：

```
路径 1 — PreToolUse Hook（自动透明）
  Bash 命令 → hook 拦截 → rtk hook claude → rtk rewrite → 自动替换

路径 2 — @RTK.md 提示（手动）
  模型读取 CLAUDE.md → 看到 @RTK.md 指令 → 主动在命令前加 rtk 前缀
```

两条路径互为补充，路径 1 自动捕获漏掉的命令。

### 6. SHA-256 完整性校验（integrity.rs）

```rust
pub enum IntegrityStatus {
    Verified,           // 哈希匹配
    Tampered { ... },   // 哈希不匹配（检测篡改）
    NoBaseline,         // 无存储的基准哈希
    NotInstalled,       // Hook 未安装
    OrphanedHash,       // 哈希存在但 Hook 被删除
}
```

**设计考虑**：
- Hash 文件设为 0o444（只读），防止意外覆盖
- `rtk verify` 运行时执行 `run_verify()` 检查所有 hook
- `is_operational_command()` 在命令执行前调用 `runtime_check()`

### 7. Agent 适配矩阵

| Agent | Hook 类型 | 安装命令 | 透明重写 |
|-------|-----------|----------|----------|
| **Claude Code** | PreToolUse Shell Hook | `rtk init -g` | ✅ |
| **Claude Code (OpenCode)** | TypeScript Plugin | `rtk init -g --opencode` | ✅ |
| **Cursor** | preToolUse JSON | `rtk init -g --agent cursor` | ✅ |
| **VS Code Copilot** | preToolUse JSON | `rtk init --copilot` | ✅ |
| **GitHub Copilot CLI** | preToolUse modifiedArgs | `rtk init --copilot` | ✅ |
| **Gemini CLI** | BeforeTool (Rust binary) | `rtk init -g --gemini` | ✅ |
| **Cline/Roo Code** | Rules 文件 | `rtk init --agent cline` | ❌ 提示级别 |
| **Windsurf** | Rules 文件 | `rtk init --agent windsurf` | ❌ 提示级别 |
| **Codex CLI** | AGENTS.md 指令 | `rtk init --codex` | ❌ 提示级别 |
| **Kilo Code** | Rules 文件 | `rtk init --agent kilocode` | ❌ 提示级别 |
| **OpenClaw** | TypeScript Plugin | 单独安装 | ✅ |

### 8. 权限系统（permissions.rs + trust.rs）

**两层权限控制**：

1. **命令级别（Pre rewrite）**：
   - `check_command(cmd)` 扫描已知危险模式（`>&file`, sudo 滥用等）
   - 返回 `Deny` / `Allow` / `Ask` 决策
   - Deny 规则触发 exit=2，完全阻止重写

2. **项目 TOML 信任级别**：
   - `.rtk/filters.toml` 默认不被信任
   - `rtk trust` → SHA-256 校验 → 存储信任
   - 内容变更 → 信任失效 → 重新审核
   - `RTK_TRUST_PROJECT_FILTERS=1` 环境变量覆盖（CI 场景）

## 结论与洞见

1. **Hook 架构优雅**：薄委托层 + 集中式 rewrite = 易于扩展新 Agent
2. **四重安全加固**：白名单 + SHA-256 + 权限系统 + 信任机制，防御攻击
3. **复合命令拆分是亮点**：shell 操作符感知，不是简单的字符串替换
4. **exit code 编码是创新**：用退出码编码 "可执行 + 需要确认" 两维信息
5. **v0.42.2 的权限系统已成熟**：相比 v0.37.2，新增 --auto-patch/--no-patch 和 Copilot 集成
