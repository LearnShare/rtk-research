# learn-module — CLI 纠正检测模块分析

> **研究日期**：2026-06-06
> **研究阶段**：Q0.1
> **研究方法**：源码分析

## 概述
分析 `src/learn/` 模块的设计：通过分析 Claude Code 会话历史中的错误命令序列，自动检测和纠正 CLI 使用错误。

## 核心发现

1. **轻量级算法**：基于模式匹配（regex）+ 局部窗口搜索 + Jaccard 相似度，无外部依赖
2. **三层过滤**：排除用户拒绝操作 → 排除 TDD 编译循环 → 排除路径探索变体
3. **6 种错误分类**：UnknownFlag / CommandNotFound / WrongSyntax / WrongPath / MissingArg / PermissionDenied + Other
4. **置信度系统**：基础分 0.5（相同 base command）+ 0.5（参数 Jaccard 相似度）+ 0.2 成功奖励
5. **与 discover 模块共享数据源**：使用相同的 `ClaudeProvider` 读取会话历史

## 详细分析

### 1. 模块架构

```
learn::run()
    │
    ├─ ClaudeProvider.discover_sessions()    ← 扫描 Claude Code 历史
    ├─ provider.extract_commands()           ← 提取命令 + 输出 + 错误标记
    │
    ├─ detector::find_corrections()          ← 核心检测算法
    │   ├─ is_command_error()               ← 过滤非错误 + 用户拒绝
    │   ├─ classify_error()                 ← 6 种错误分类
    │   ├─ is_tdd_cycle_error()             ← 排除编译错误
    │   ├─ differs_only_by_path()           ← 排除路径探索
    │   └─ command_similarity()             ← Jaccard 相似度
    │
    ├─ detector::deduplicate_corrections()  ← 合并相同规则
    │
    ├─ report::format_console_report()      ← 终端输出
    └─ report::write_rules_file()           ← Markdown 规则文件
```

### 2. 错误分类正则

| 类型 | 正则 | 示例匹配 |
|------|------|----------|
| UnknownFlag | `unexpected argument\|unknown (option\|flag)\|...` | `error: unexpected argument '--foo'` |
| CommandNotFound | `command not found\|not recognized as an internal\|...` | `bash: foobar: command not found` |
| WrongPath | `no such file or directory\|cannot find the path\|...` | `cat: foo.txt: No such file or directory` |
| MissingArg | `requires a value\|requires an argument\|...` | `error: --output requires a value` |
| PermissionDenied | `permission denied\|access denied\|not permitted` | `permission denied: /etc/shadow` |
| Other | 默认匹配 | `something went wrong` |

### 3. 检测算法

**滑动窗口搜索**：`CORRECTION_WINDOW = 3` — 仅在错误命令后的 3 条命令内搜索纠正

```
时间序列：
  cmd[0]: git commit --ammend        (is_error=true)  ← 错误
  cmd[1]: ls -la                      (is_error=false) ← 无关
  cmd[2]: pwd                         (is_error=false) ← 无关
  cmd[3]: git commit --amend          (is_error=false) ← 太远，命中窗口边界
```

**匹配条件**（全部满足）：
1. `command_similarity(wrong, right) >= 0.5`（相同 base command）
2. 不是 TDD 编译错误
3. 不是仅路径不同的探索行为
4. `confidence >= MIN_CONFIDENCE (0.6)`

### 4. 相似度算法

```rust
command_similarity(a, b) =
    base_a == base_b ?
        0.5 + |args_a ∩ args_b| / |args_a ∪ args_b| * 0.5
    :
        0.0
```

如果纠正后的命令成功（非错误），confidence 额外 +0.2（上限 1.0）。

### 5. 去重逻辑

按 `(base_command, error_type_str, diff_token)` 三元组分组：

```rust
fn extract_diff_token(wrong, right) → "removed --foo" / "added --bar" / "--ammend → --amend"
```

每组保留置信度最高的示例，统计 `occurrences`。

### 6. 测试覆盖

13 个测试覆盖：
- `is_command_error` 边界条件（需 error=true + 排除 user rejection）
- 5 种错误分类的匹配
- `extract_base_command` 前缀剥离
- `command_similarity` 相同/不同 base + 参数重叠
- `find_corrections` 完整流程：窗口限制、TDD 排除、路径探索排除、置信度门槛
- `deduplicate_corrections` 合并相同 + 保留不同

## 结论与洞见

1. **设计实用但轻量**：没有使用 ML/AI，纯规则引擎，500 行代码完成任务
2. **Jaccard 参数相似度有局限性**：`--ammend vs --amend` 被视为 0% 参数相似度（完全不同的字符串），但不是通过 base command 0.5 + 匹配成功奖励 0.2 = 0.7 通过门槛
3. **路径探索排除可以更精确**：当前用 `similarity > 0.9` 近似判断，可能误判
4. **数据源耦合**：learn 模块依赖 discover::provider::ClaudeProvider，独立测试时需 mock
