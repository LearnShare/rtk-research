# command-injection-audit — RTK 命令注入面分析

> **研究日期**：2026-06-06
> **研究阶段**：Q1.1
> **研究方法**：源码审计 + 攻击建模

## 概述
系统审计 RTK 的命令注入攻击面，包括 `permissions.rs` 规则引擎、shell 令牌化安全、以及 `resolved_command` 的路径解析风险。

## 核心发现

1. **四层防御纵深**：命令白名单 + 权限规则（Deny/Ask/Allow）+ 不可审核构造检测 + SHA-256 完整性
2. **`permissions.rs` 是核心防御**：998 行规则引擎，42 个测试，覆盖命令匹配、复合命令拆分、逃逸检测
3. **复合命令分段权限**：每个 `&&`/`|`/`;` 段独立检查，防止单段绕过
4. **不可审核构造完全阻止自动允许**：`$()`, `` ` ``, `>&file` 触发的命令永远 Ask
5. **引号感知拆分**：引号内的 `&&` 不会触发命令拆分

## 详细分析

### 1. 命令执行路径安全分析

RTK 的 3 条命令执行路径：

| 路径 | 代码 | 安全级别 | 风险 |
|------|------|----------|------|
| 1. `resolved_command()` | `core/utils.rs` | 🟢 安全 | 使用 `which` crate 查找可执行文件，拒绝 shell 注入 |
| 2. `Command::new("netstat")` | 直接创建 | 🟢 安全 | 固定命令名，args 参数化 |
| 3. `run fallback → sh -c` | `main.rs run_fallback()` | 🟡 需注意 | 仅在有 TOML match 时使用，否则 `Stdio::inherit` |
| 4. `rtk run -c "cmd"` | `main.rs Run` | 🟡 用户可控 | 使用 `sh -c` 或 `cmd /C` 执行，用户显式调用 |

### 2. permissions.rs 引擎

**规则来源**：从 Claude Code settings.json 的 `permissions.deny/ask/allow` 中读取 `Bash(...)` 模式。

**优先级**：`Deny > Ask > Allow > Default (ask)`

**匹配算法**（`command_matches_pattern`）：
```
1. "*" → 全匹配
2. "prefix*" → 前缀匹配（词边界保护）
3. "*suffix" → 后缀匹配
4. "pre*suf" → GLOB 匹配
5. "exact" → 精确/前缀匹配
```

**词边界保护**：`"git push --force"` 不匹配 `"git push --forceful"`（防止意外拒绝）。

**词边界保护 —— 反例已修复**：`"sudo:*"` 不匹配 `"sudoedit"`（Bug #1，已修复）。

### 3. 复合命令安全

`split_compound_command()` 使用 `lexer.rs` 的 `split_for_permissions()` 拆分：

```rust
// 输入: "git status && rm -rf /"
// 拆分: ["git status ", " rm -rf /"]
// 检查: 每段独立匹配拒绝规则
// 结果: Deny（rm -rf / 匹配拒绝规则）
```

**关键设计**——#1213 修复：
- Allow 要求**每个**非空段都独立匹配 allow 规则
- 单段匹配不能提升整个链为 Allow
- 引用内 `&&` 不拆分（`echo "a && b"` 作为单段）

### 4. 不可审核构造检测

`contains_unattestable_construct()` 在 `lexer.rs` 中检测：

| 构造 | 检测 | 处理 |
|------|------|------|
| `$(cmd)` 子 shell | `$(` | → Ask（永不 Auto-Allow） |
| `` `cmd` `` 反引号 | `` ` `` | → Ask |
| `>&file` 文件重定向 | `>&` | → Ask |
| 换行注入 `cmd\nhidden` | `\n` | → 拆分检查每段 |

### 5. 换行与后台命令逃逸

```rust
// 换行隐藏命令: "git status\nrm -rf ~" 
// → 拆分为 "git status" 和 "rm -rf ~"
// → "rm -rf ~" 匹配 deny 规则 → Deny

// 后台命令: "git status & rm -rf ~"
// → 拆分为 "git status " 和 " rm -rf ~"
// → "rm -rf ~" 匹配 deny 规则 → Deny
```

### 6. 遗留风险

| 风险 | 严重性 | 说明 | 建议 |
|------|--------|------|------|
| `run_fallback` 在 TOML match 路径使用 `resolved_command` | 🟡 中 | 命令名来自用户输入，但 `which` 限制可执行文件 | 增加命令名校验 |
| `rtk run -c` 直接 `sh -c` | 🟡 中 | 用户显式要求执行任意命令 | 已在 CLI 中明确 |
| Windows `cmd /C` 环境差异 | 🟢 低 | 与 POSIX sh 引号语义不同 | 跨平台测试覆盖 |
| `permissions.rs` 规则来自用户配置 | 🟢 低 | 用户自己管理权限文件 | 当前设计合理 |
| Cursor/Gemini 规则加载路径不同 | 🟢 低 | 文件读取路径不同但逻辑相同 | 一致性已验证 |

## 结论与洞见

1. **权限引擎是 RTK 安全核心**：998 行代码 + 42 个测试，覆盖主要攻击面
2. **复合命令分段是正确设计**：#1213 修复后所有段必须独立匹配，无单段绕过
3. **不可审核构造作为安全门**：永远不允许自动执行包含 `$()`/`` ` ``/> 的命令
4. **RTK 的安全模型保守**：默认 Ask（不自动允许），需用户显式配置 Allow
5. **测试覆盖完整**：逃逸测试包括换行、子 shell、引号内操作符、复合链
