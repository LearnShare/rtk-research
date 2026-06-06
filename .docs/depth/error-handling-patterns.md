# error-handling-patterns — RTK 错误处理模式统计

> **研究日期**：2026-06-06
> **研究阶段**：Q4.5
> **研究方法**：源码检索 + 分类

## 概述
统计 RTK 源码中的错误处理模式分布，验证编码规则的执行情况。

## 核心发现

1. **`.context()` 159 处**：主要错误上下文附加模式，平均每 100 行 1 次
2. **`.unwrap()` 890 处**：看似很高，但绝大多数在 `lazy_static!`（246 处）和测试代码中
3. **`unwrap_or_else()` 60 处**：fallback 模式，生产环境的关键安全网
4. **`exit(code)` 调用约 30 处**：退出码传播机制
5. **`eprintln!` 约 50 处**：错误/警告输出模式

## 详细分析

### 1. 错误上下文附加 `.context()`

159 处，分布在 35 个文件中，最集中的是 `git.rs`（26 处）和 `cc_economics.rs`（22 处）。

**标准模式**：
```rust
use anyhow::{Context, Result};

fn read_config(path: &Path) -> Result<Config> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("Failed to read config: {}", path.display()))?;
    toml::from_str(&content).context("Failed to parse config TOML")
}
```

### 2. unwrap 分类

890 个 `.unwrap()` 的分布（部分取样）：

| 类别 | 数量 | 位置 | 可接受 |
|------|------|------|--------|
| `lazy_static!` 初始化 | ~246 | 所有 filter 模块 | ✅ 编程错误 |
| `#[cfg(test)]` 测试 | ~500 | 模块内测试 | ✅ 测试代码 |
| 生产代码（`unreachable!`/`expect`/`try_parse`） | ~144 | main.rs, registry.rs | ⚠️ 需审计 |

**`lazy_static!` 模式**（~246 处，可接受）：
```rust
lazy_static! {
    static ref RE: Regex = Regex::new(r"^pattern$").unwrap();
    // 正则字面量错误是编程错误，编译时不可检测
}
```

**生产代码示例**（需注意安全）：
```rust
// 安全：解析静态模式字符串
Cli::try_parse_from(["rtk", cmd, "--nonexistent-flag-xyz"]).unwrap();

// 安全：从 `Option` 提取退出码
output.status.code().unwrap_or(1)  // 这里更常用 unwrap_or
```

### 3. Fallback 模式 `unwrap_or_else()`

60 处，典型用法：

```rust
// filter 过滤失败 → 返回原始输出
let filtered = filter_output(&output.stdout)
    .unwrap_or_else(|e| {
        eprintln!("rtk: filter warning: {}", e);
        output.stdout.clone()
    });

// 配置加载失败 → 使用默认值
let max_files = config.tee.max_files.unwrap_or(20);
```

### 4. Exit 码传播

**模式**：
```rust
// 子命令失败 → 传播退出码
if !output.status.success() {
    std::process::exit(output.status.code().unwrap_or(1));
}
```

### 5. 警告输出 `eprintln!`

约 50 处，用于：
- TOML filter 未信任警告
- JSON 解析失败
- 版本差异/兼容性问题
- 过滤警告

**模式**：
```rust
eprintln!("rtk: filter warning: {}", e);
eprintln!("[rtk] WARNING: untrusted project filters");
```

## 结论

1. 编码规则严格执行：`anyhow::Result` + `.context()` 是标准模式
2. `lazy_static!` 中的 `.unwrap()` 是 RTK 认可的最佳实践
3. Fallback 模式覆盖主要错误路径
4. Exit 码传播一致
