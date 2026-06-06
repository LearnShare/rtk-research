# parser-module — 解析器基础设施分析

> **研究日期**：2026-06-06
> **研究阶段**：Q0.2
> **研究方法**：源码分析

## 概述
分析 `src/parser/` 模块的设计：提供统一的工具输出解析基础设施，支持三级降解（Full → Degraded → Passthrough）的安全降级策略。

## 核心发现

1. **三级降解策略是安全核心**：Full（完整 JSON 解析）→ Degraded（部分数据+警告）→ Passthrough（原始截断），确保不返回假数据
2. **JSON 提取器 `extract_json_object` 是关键工具**：处理带前缀的工具输出（pnpm banner、dotenv 消息），大括号平衡算法可处理嵌套 JSON
3. **`ParseResult<T>` 泛型枚举**：优雅编码三级状态，支持 map / unwrap / tier 查询
4. **`OutputParser` Trait**：统一解析接口，允许按 tier 上限限制解析深度
5. **`TokenFormatter` Trait**：与 parser 互补，将结构化类型格式化为 Token 高效文本

## 详细分析

### 1. 模块结构

```
src/parser/
├── mod.rs         ← ParseResult<T> 枚举 + OutputParser Trait + JSON 提取器
├── types.rs       ← 标准类型（TestResult, DependencyState）
└── formatter.rs   ← TokenFormatter Trait + FormatMode
```

### 2. 三级降解设计

```rust
pub enum ParseResult<T> {
    Full(T),                  // Tier 1: 完整 JSON 解析
    Degraded(T, Vec<String>), // Tier 2: 部分数据 + 警告
    Passthrough(String),      // Tier 3: 截断原始输出 + [RTK:PASSTHROUGH] 标记
}
```

**降级路径**：Tier 1 → Tier 2 → Tier 3。每个 filter 的实现者决定降级条件。

`parse_with_tier()` 允许限制最大解析层：
```rust
fn parse_with_tier(input: &str, max_tier: u8) -> ParseResult<Self::Output>
```

### 3. JSON 提取算法

`extract_json_object()` 从带有工具 banner 的输出中提取第一个完整 JSON 对象：

```
输入（pnpm + JSON）:
Scope: all 6 workspace projects
{"numTotalTests": 13, "numPassedTests": 13}

提取策略:
1. 查找 "numTotalTests" 标记（vitest 特有）
2. 回退到第一个独立的 '{'
3. 大括号平衡向前扫描到匹配的 '}'
4. 处理转义字符和字符串内大括号
```

测试覆盖了：纯净 JSON、pnpm 前缀、dotenv 前缀、嵌套大括号、无 JSON 输入、字符串内大括号。

### 4. 标准类型

```rust
TestResult {
    total, passed, failed, skipped: usize,
    duration_ms: Option<u64>,
    failures: Vec<TestFailure>   // test_name, file_path, error_message, stack_trace
}

DependencyState {
    total_packages, outdated_count: usize,
    dependencies: Vec<Dependency>   // name, current_version, latest_version, ...
}
```

### 5. TokenFormatter

三种格式模式对应 CLI 的详细程度：

| 模式 | 触发 | 输出风格 |
|------|------|----------|
| `Compact` | 默认 | `PASS (13) FAIL (0)` |
| `Verbose` | `-v` | 含详细错误信息 |
| `Ultra` | `--ultra-compact` | 符号和缩略语 |

### 6. 测试覆盖

13 个测试 + JSON 提取器 5 个专项测试：
- `ParseResult` 的 tier/is_ok/unwrap/map/warnings
- `truncate_output` 多字节/emoji 安全
- `extract_json_object` 各种前缀场景
- JSON 内字符串大括号（防止误匹配）

## 结论与洞见

1. **三级降解策略安全性高**：优先返回降级数据而非错误，适合 LLM 场景
2. **JSON 提取器解决真实问题**：vitest/pnpm/dotenv 的输出前缀在测试框架中常见
3. **TokenFormatter 与 OutputParser 分离**：解析（parser）和格式化（formatter）关注点分离
4. **仅 vitest/playwright 使用了 parser**：大部分 filter 仍用直接解析，parser 模块的采用度不高
