# hook-installation-test — RTK Hook 安装与端到端测试

> **研究日期**：2026-06-06
> **研究阶段**：P0.3
> **研究方法**：实际操作 + 配置审计

## 概述
验证 RTK Hook 系统的安装状态和命令重写链路的完整性。

## 核心发现

1. **Hook 已安装且激活**：`PreToolUse` hook 针对 `Bash` 工具，自动运行 `rtk hook claude`
2. **RTK.md 已配置**：slim 模式 + `@RTK.md` 在 CLAUDE.md 中引用
3. **命令重写链路完整**：`git status` → `rtk git status`，`cargo test && git push` → `rtk cargo test && rtk git push`
4. **ripgrep 自动转换**：`rg foo` → `rtk grep foo`
5. **`--dry-run` 未实现**：v0.37.2 不支持，v0.42.2 源码中已有
6. **`rtk rewrite` exit=3 而非 0**：v0.37.2 的非标准行为

## 详细分析

### 当前 Hook 配置 (`rtk init --show`)

```
[ok] Hook: rtk hook claude (native binary command)
[ok] RTK.md: C:\Users\hu\.claude\RTK.md (slim mode)
[ok] Global (~/.claude/CLAUDE.md): @RTK.md reference
[ok] Local (./CLAUDE.md): rtk enabled
[ok] settings.json: RTK hook configured
[ok] OpenCode: plugin installed
[--] Cursor hook: not found
```

### settings.json Hook 定义

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "rtk hook claude"
      }]
    }]
  }
}
```

这是 **Auto-Rewrite 模式**：所有 Bash 命令在执行前被 `rtk hook claude` 拦截并重写。

### 命令重写测试

| 原始命令 | 重写结果 | 状态 |
|----------|----------|------|
| `git status` | `rtk git status` | ✅ |
| `ls -la` | `rtk ls -la` | ✅ |
| `cargo test && git push` | `rtk cargo test && rtk git push` | ✅ 复合命令正确拆分 |
| `rg foo` | `rtk grep foo` | ✅ ripgrep → RTK grep |
| `npm install` | (无输出) | ✅ 无匹配，exit=1 |

### 版本差异

| 特性 | v0.37.2 | v0.42.2 (源码) |
|------|---------|---------------|
| `rtk init --dry-run` | ❌ 不支持 | ✅ 已实现 |
| `rtk rewrite` exit code | exit=3 | exit=0 |
| `--agent` 选项 | 6 种 | 8+ 种 (含 Pi, Hermes) |
| `--copilot` 安装 | ✅ 支持 | ✅ 完善 |

### 双重路径覆盖

当前 RTK 通过两条路径并行工作：

```
路径 1 — PreToolUse Hook（透明）
  LLM Bash → hook 拦截 → rtk hook claude → rtk rewrite → 执行

路径 2 — CLAUDE.md @RTK.md（显式）
  LLM 读取 CLAUDE.md → 看到 "所有命令加 rtk 前缀" → 主动使用 rtk
```

这确保了 100% 覆盖率（Hook 捕获漏掉的，CLAUDE.md 提示主动使用）。

## 结论与洞见

- Hook 系统运行正常，无需重新初始化
- v0.37.2 功能基本完备，`--dry-run` 等新特性缺失但不影响使用
- 后续源码分析（P1-P6）以 v0.42.2 为准，可关注 exit code 规范的演进
