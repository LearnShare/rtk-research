# testing-strategy — RTK 测试体系全景分析

> **研究日期**：2026-06-06
> **研究阶段**：P1.5
> **研究方法**：源码分析 + 代码审查

## 概述
分析 RTK 的完整测试策略：快照测试、Token 节省验证、测试夹具和集成测试框架。

## 核心发现

1. **三层测试体系**：单元测试（内联）+ 快照测试（`insta`）+ 集成测试（`#[ignore]`）
2. **Token 节省硬约束**：所有 filter 必须验证 ≥60% token 节省，否则 release block
3. **真实夹具强制**：使用 `include_str!` 加载真实命令输出，禁止合成数据
4. **TOML 内联测试**：每个 TOML filter 自带 `[[tests.*]]` 定义，`build.rs` + `rtk verify` 双重验证
5. **跨平台测试隔离**：`#[cfg(target_os = "...")]` 标注平台特定行为

## 详细分析

### 测试层次架构

```
┌─────────────────────────────────────────┐
│  Integration Tests (#[ignore])           │  真实命令执行
│  tests/integration_test.rs               │  需要安装 rtk 二进制
│  cargo test --ignored                    │
├─────────────────────────────────────────┤
│  Snapshot Tests (insta)                  │  输出格式回归
│  assert_snapshot!(output)                │  自动管理 .snap 文件
│  cargo insta review                      │
├─────────────────────────────────────────┤
│  Unit Tests (#[cfg(test)] mod tests)     │  函数级行为验证
│  内联在各 *_cmd.rs / *.rs 文件中          │  count_tokens() + savings assertion
│  cargo test --all                        │
└─────────────────────────────────────────┘
```

### 快照测试模式

使用 `insta` crate 管理期望输出：

```
src/cmds/git/
├── git.rs              ← #[cfg(test)] mod tests { ... }
└── snapshots/
    └── git__tests__test_git_log_format.snap   ← 自动生成
```

**核心模式**：
```rust
#[test]
fn test_git_log_format() {
    let input = include_str!("../tests/fixtures/git_log_raw.txt");
    let output = filter_git_log(input);
    assert_snapshot!(output);  // 自动创建/比较快照
}
```

**工作流程**：
```
cargo test → 创建新快照
cargo insta review → 交互式审查
cargo insta accept → 接受变更
```

### Token 节省验证

每个 filter 测试必须包含：

```rust
fn count_tokens(text: &str) -> usize {
    text.split_whitespace().count()
}

#[test]
fn test_token_savings() {
    let input = include_str!("...");
    let output = filter(input);
    let savings = 100.0 - (count_tokens(&output) as f64 / count_tokens(input) as f64 * 100.0);
    assert!(savings >= 60.0, "Expected ≥60% savings, got {:.1}%", savings);
}
```

**各 filter 节省目标**：

| Filter | 目标 | 测试夹具 |
|--------|------|----------|
| `git log` | 80%+ | `tests/fixtures/git_log_raw.txt` |
| `cargo test` | 90%+ | `tests/fixtures/cargo_test_raw.txt` |
| `gh pr view` | 87%+ | `tests/fixtures/gh_pr_view_raw.txt` |
| `pnpm list` | 70%+ | `tests/fixtures/pnpm_list_raw.txt` |
| `docker ps` | 60%+ | `tests/fixtures/docker_ps_raw.txt` |

### 测试夹具策略

```
tests/fixtures/
├── git_log_raw.txt        ← $ git log -20
├── cargo_test_raw.txt     ← $ cargo test 2>&1
├── gh_pr_view_raw.txt     ← $ gh pr view 123
├── pnpm_list_raw.txt      ← $ pnpm list
├── dotnet/                ← dotnet 专用夹具
└── ...
```

**铁律**：必须使用 `真实命令输出`，合成数据被禁止。理由：
- 真实输出包含 ANSI 码、emoji、多字节字符
- 合成数据无法反映实际过滤效果
- 节省率断言依赖真实数据才能有效

### TOML 内联测试

每个 TOML filter 文件包含测试定义：

```toml
[[tests.make]]
name = "empty output"
input = "make: Nothing to be done for 'all'."
expected = "make: ok"

[[tests.make]]
name = "error output"
input = "make: *** [target] Error 1\n..."
expected = "make: *** [target] Error 1"
```

**双重验证路径**：
1. `build.rs` — 构建时解析 `[[tests.*]]`，运行并验证
2. `rtk verify` — 运行时运行所有 TOML filter 的内联测试

### main.rs 测试

3275 行的 main.rs 中有 ~500 行测试：
- Clap 参数解析验证（`test_git_commit_*` 系列）
- Meta 命令 fallback 防护测试
- shell_split 令牌化测试（支持单引号/双引号）
- 集成测试（`#[ignore]`，需要 `cargo build` 产物）

### 测试命令清单

```bash
cargo test --all              # 全部单元测试 + 快照测试
cargo test --ignored          # 集成测试（需要已安装 rtk）
cargo insta review            # 审查快照变更
cargo insta accept            # 接受快照变更
bash scripts/test-all.sh      # 冒烟测试（需要已安装 rtk）
rtk verify                    # TOML filter 内联测试 + hook 完整性
```

### 测试反模式（来自 .claude/rules/cli-testing.md）

❌ 使用硬编码合成数据
❌ 跳过跨平台测试
❌ 忽略性能回归
❌ 接受 <60% token 节省

## 结论与洞见

1. **快照测试是主要策略**：insta 提供了低摩擦的输出回归测试
2. **Token 节省断言是创新**：将商业目标（降低成本）直接编码为测试约束
3. **真实夹具确保有效性**：禁止合成数据避免 "实验室条件" 偏差
4. **TOML 内联测试降低贡献门槛**：无需 Rust 知识即可为 TOML filter 添加测试
5. **集成测试标记 `#[ignore]`**：正确区分 CI 快速反馈和完整验证
