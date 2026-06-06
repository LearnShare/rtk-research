# stream-module — 流式处理模块分析

> **研究日期**：2026-06-06
> **研究阶段**：Q0.3
> **研究方法**：源码分析

## 概述
分析 `src/core/stream.rs` 的流式过滤架构：`StreamFilter` Trait + `LineHandler`（行级）+ `BlockHandler`（块级）两种实现策略。

## 核心发现

1. **双模式流式过滤**：`LineHandler`（行级逐行处理）和 `BlockHandler`（块级聚合），根据数据类型选择
2. **`StreamFilter` Trait 统一接口**：`feed_line()` → `flush()` → `on_exit()` 生命周期
3. **行级默认保留**：`LineHandler` 的 `should_skip()` 默认返回 false（保留所有行）
4. **块级负责聚合**：`BlockHandler` 管理块的开始/延续/结束边界
5. **仅在 git 和 cargo filter 中使用**：采用度有限，大部分 filter 使用自包含的过滤函数

## 详细分析

### 1. StreamFilter 核心 Trait

```rust
pub trait StreamFilter {
    fn feed_line(&mut self, line: &str) -> Option<String>;  // 处理每一行
    fn flush(&mut self) -> String;                           // 刷新缓冲
    fn on_exit(&mut self, exit_code: i32, raw: &str) -> Option<String>;  // 退出时摘要
}
```

生命周期：`feed_line()` 按行调用 → `flush()` 输出剩余缓冲 → `on_exit()` 总结

### 2. 行级处理：LineHandler

```rust
pub trait LineHandler {
    fn should_skip(&mut self, _line: &str) -> bool { false }  // 跳过噪声行
    fn observe_line(&mut self, _line: &str) {}                 // 跟踪状态
    fn format_summary(&self, exit_code: i32, raw: &str) -> Option<String>;
}

pub struct LineStreamFilter<H: LineHandler> {
    handler: H,
}
```

- 每行经过 `should_skip` → 跳过或保留
- `observe_line` 收集统计（如错误计数）
- `on_exit` 时 `format_summary` 生成总结

### 3. 块级处理：BlockHandler

```rust
pub trait BlockHandler {
    fn should_skip(&mut self, line: &str) -> bool;    // 跳过行
    fn is_block_start(&mut self, line: &str) -> bool; // 检测块开始
    fn is_block_continuation(&mut self, line: &str, block: &[String]) -> bool; // 检测块延续
    fn format_summary(&self, exit_code: i32, raw: &str) -> Option<String>;
}

pub struct BlockStreamFilter<H: BlockHandler> {
    in_block: bool,
    current_block: Vec<String>,
    blocks_emitted: usize,
}
```

状态机：
```
行输入 → should_skip?
  ├─ true → 跳过
  └─ false → is_block_start?
       ├─ true → 开始新块（发出前一个块）
       └─ false → in_block? → is_block_continuation?
            ├─ true → 追加到当前块
            └─ false → 结束并发出块
```

### 4. Test Helper: RegexBlockFilter

`src/core/stream.rs` 中提供了一个专为测试设计的块过滤器：

```rust
#[cfg(test)]
pub struct RegexBlockFilter {
    start_re: Regex,       // 块开始正则
    skip_prefixes: Vec<String>,  // 跳过前缀
    tool_name: String,     // 工具名（用于摘要）
    block_count: usize,    // 已发出块数
}
```

支持链式配置：`RegexBlockFilter::new("cargo", r"^error\[").skip_prefix("warning:")`

### 5. 使用位置

| 位置 | 使用模式 | 用途 |
|------|----------|------|
| `src/cmds/git/git.rs` | `LineHandler` | git log 流式过滤 |
| `src/cmds/rust/cargo_cmd.rs` | `BlockHandler` + `RegexBlockFilter` | cargo build 错误块聚合 |
| 测试 | `RegexBlockFilter` | 块过滤行为验证 |

### 6. 与 TOML filter 的关系

```
stream.rs（内部流式过滤）
  → 用于 git log、cargo build 等实时输出
  → 在命令执行过程中逐行处理
  
TOML DSL（后处理过滤）
  → 用于 ipconfig、make、brew 等
  → 命令执行完成后处理完整输出
```

两者互补：流式用于实时，TOML DSL 用于批处理。

## 结论与洞见

1. **设计简洁但采用度低**：两个抽象 Trait + 两种实现，但只有 git 和 cargo 使用
2. **`RegexBlockFilter` 在测试中更有用**：单元测试中模拟过滤行为，`#[cfg(test)]` 条件编译
3. **与 TOML DSL 互补**：流式处理实时输出，TOML 处理完整输出
4. **`on_exit` 是 hidden gem**：命令退出时生成摘要，可用于总结测试结果
