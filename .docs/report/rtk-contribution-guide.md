# rtk-contribution-guide — RTK 贡献指南（基于源码研究）

> **研究日期**：2026-06-06
> **研究阶段**：P6.3
> **研究方法**：综合 P0-P5 研究成果

## 1. 新命令过滤模块实现步骤

### 1.1 决策：Rust-native vs TOML DSL

| 场景 | 选择 |
|------|------|
| 输出需要结构化解析（JSON, XML, diff） | Rust-native |
| 需要状态机、跨行聚合 | Rust-native |
| 简单行过滤 + 截断 + 空结果兜底 | TOML DSL |
| 长尾小众工具 | TOML DSL 更高效 |

### 1.2 Rust-native 实现步骤

1. **创建模块文件**
   ```bash
   touch src/cmds/<ecosystem>/<cmd>_cmd.rs
   ```

2. **实现模块结构**
   ```rust
   //! Filters <tool> output.
   
   use crate::core::{tracking, utils::{execute_command, resolved_command}};
   use anyhow::{Context, Result};
   use lazy_static::lazy_static;
   use regex::Regex;
   
   pub fn run(args: &[String], verbose: u8) -> Result<i32> {
       let timer = tracking::TimedExecution::start();
       let output = execute_command("<tool>", args)?;
       let filtered = filter_output(&output.stdout);
       print!("{}", filtered);
       timer.track("<tool>", "rtk <tool>", &output.stdout, &filtered);
       Ok(exit_code_from_output(&output, "<tool>"))
   }
   
   fn filter_output(input: &str) -> String {
       // 实现过滤逻辑
   }
   ```

3. **注册到 main.rs**
   ```rust
   use cmds::<ecosystem>::<cmd>_cmd;
   // 在 Commands 枚举中添加变体
   // 在 run_cli() match 中添加路由
   ```

4. **添加测试**
   ```rust
   #[cfg(test)]
   mod tests {
       use super::*;
       fn count_tokens(s: &str) -> usize { s.split_whitespace().count() }
       
       #[test]
       fn test_output_format() {
           let input = include_str!("../tests/fixtures/<cmd>_raw.txt");
           let output = filter_output(input);
           assert_snapshot!(output);
       }
       
       #[test]
       fn test_token_savings() {
           let input = include_str!("../tests/fixtures/<cmd>_raw.txt");
           let output = filter_output(input);
           let savings = /* ≥60% */;
           assert!(savings >= 60.0, "...");
       }
   }
   ```

5. **创建夹具**
   ```bash
   <tool> args > tests/fixtures/<cmd>_raw.txt
   ```

6. **验证**
   ```bash
   cargo fmt --all && cargo clippy --all-targets && cargo test --all
   ```

### 1.3 TOML DSL 实现步骤

1. **创建 filter 文件**
   ```bash
   touch src/filters/<tool>.toml
   ```

2. **编写 filter 规则**
   ```toml
   [filters.<tool>]
   description = "Short description"
   match_command = "^<tool>\\b"
   strip_ansi = true
   strip_lines_matching = ["^\\s*$", "^\\[.*\\]"]
   head_lines = 30
   max_lines = 40
   on_empty = "<tool>: ok"
   
   [[tests.<tool>]]
   name = "test name"
   input = "real command output"
   expected = "expected filtered output"
   ```

3. **验证**
   ```bash
   cargo test --all  # build.rs 会在构建时验证 TOML
   rtk verify        # 运行时验证
   ```

## 2. 测试策略

### 2.1 Token 节省验证

每个 filter 必须验证 ≥60% Token 节省，使用 `tests/fixtures/*.txt` 中的真实命令输出：

```rust
#[test]
fn test_token_savings() {
    let input = include_str!("../tests/fixtures/cmd_raw.txt");
    let output = filter_cmd(input);
    let savings = 100.0 - (count_tokens(&output) as f64 / count_tokens(input) as f64 * 100.0);
    assert!(savings >= 60.0, "Expected ≥60% savings, got {:.1}%", savings);
}
```

### 2.2 快照测试

使用 `insta` 保护输出格式：

```rust
#[test]
fn test_output_format() {
    let output = filter_cmd(input);
    assert_snapshot!(output);
}
```

### 2.3 夹具创建

```bash
# 真实命令输出
git log -20 > tests/fixtures/git_log_raw.txt
cargo test 2>&1 > tests/fixtures/cargo_test_raw.txt
```

## 3. PR 提交流程

```bash
# 1. 创建分支
git checkout -b feat/my-new-filter

# 2. 实现 + 测试
cargo fmt --all
cargo clippy --all-targets
cargo test --all

# 3. 提交
git add -A
git commit -m "feat(<ecosystem>): add <tool> filter

- <filter 功能描述>
- <关键设计决策>
- closes #<issue>"

# 4. 推送
git push -u origin feat/my-new-filter
```

### PR 检查清单

- [ ] `cargo fmt --all` 通过
- [ ] `cargo clippy --all-targets` 零警告
- [ ] `cargo test --all` 全部通过
- [ ] Token 节省 ≥60%（Rust-native filter）
- [ ] 使用真实命令输出作为夹具
- [ ] 添加了 insta 快照测试
- [ ] TOML filter 包含 `[[tests.*]]`
- [ ] `rtk verify` 全部通过
- [ ] 已注册到 main.rs Commands 枚举
- [ ] 跨平台 shell 转义已处理

## 4. 编码规则速查

| 规则 | 说明 |
|------|------|
| 无 async | 单线程设计，启动 <10ms |
| 无 unwrap() | 使用 `.context()?`，测试用 `.expect()` |
| lazy_static! | 所有正则使用 lazy_static! |
| Fallback | 过滤失败 → 原始输出 |
| Exit code | 子命令失败 → `exit(code)` 传播 |
| 迭代器优先 | 使用 iterator chain 而非手动 loop |
| &str 优先 | 入参用 &str 而非 &String |
| cfg(test) 内联 | 测试写在模块内 |
