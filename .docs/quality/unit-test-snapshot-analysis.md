# unit-test-snapshot-analysis — RTK 单元测试与快照测试分析

> **研究日期**：2026-06-06
> **研究阶段**：P3.1
> **研究方法**：源码分析 + 运行测试

## 概述
分析 RTK 的测试体系：2065 个测试全部通过，0 failed，7 ignored。涵盖单元测试、快照测试（insta）、Token 节省验证和 TOML 内联测试。

## 核心发现

1. **2065/2065 全绿**：7 ignored（集成测试），0 failures
2. **16 个测试夹具**：`tests/fixtures/` 下包含 gradlew, dotnet, glab, golangci 的真实命令输出
3. **内联测试覆盖 TOML filter**：`rtk verify` 独立运行，`build.rs` 构建时也验证
4. **无快照目录**：`src/**/snapshots/` 为空（insta 快照尚未创建/接受）
5. **`#[ignore]` 标记集成测试**：需要已安装的 RTK 二进制，`cargo test --ignored` 运行

## 详细分析

### 1. 测试全量运行结果

```
cargo test --all
────────────────────────────────
2065 passed, 7 ignored (1 suite, 30.97s)
```

**0 失败**，7 个 ignored 测试为集成测试（需要预编译二进制）。

### 2. 测试分布

| 类别 | 位置 | 数量 | 说明 |
|------|------|------|------|
| main.rs Clap 测试 | `src/main.rs` | ~50 | 参数解析、shell_split、Meta 命令 |
| 命令过滤测试 | `src/cmds/*/_cmd.rs` | ~800 | 各 filter 的内联测试 |
| Core 基础设施测试 | `src/core/*.rs` | ~300 | config/tracking/utils/truncate |
| Hook 系统测试 | `src/hooks/*.rs` | ~400 | rewrite/registry/rules/trust |
| 分析/发现测试 | `src/analytics/`, `src/discover/` | ~400 | gain/discover/learn |
| TOML 内联测试 | `src/filters/*.toml` | 145 | `rtk verify` 独立运行 |
| 集成测试 | `#[ignore]` | 7 | 需要已安装 RTK 二进制 |

### 3. 测试夹具清单

```
tests/fixtures/
├── dotnet/
│   ├── build_failed.txt
│   ├── format_changes.json
│   ├── format_empty.json
│   ├── format_success.json
│   └── test_failed.txt
├── glab_ci_trace_raw.txt
├── glab_issue_list_raw.json
├── glab_mr_list_raw.json
├── glab_release_list_raw.txt
├── glab_release_view_raw.txt
├── golangci_v2_json.txt
├── gradlew_build_failed_raw.txt
├── gradlew_build_raw.txt
├── gradlew_connected_raw.txt
├── gradlew_lint_raw.txt
├── gradlew_test_failed_raw.txt
└── gradlew_test_raw.txt
```

夹具覆盖 3 个生态系统（gradlew, dotnet, glab, golangci），均为真实命令输出的捕获结果。

### 4. 测试模式示例

**模式 1：Clap 参数解析（main.rs）**
```rust
#[test]
fn test_git_commit_single_message() {
    let cli = Cli::try_parse_from(["rtk", "git", "commit", "-m", "fix: typo"]).unwrap();
    match cli.command {
        Commands::Git { command: GitCommands::Commit { args }, .. } => {
            assert_eq!(args, vec!["-m", "fix: typo"]);
        }
        _ => panic!("Expected Git Commit command"),
    }
}
```

**模式 2：Token 节省验证（典型 filter）**
```rust
#[cfg(test)]
mod tests {
    use super::*;
    fn count_tokens(s: &str) -> usize { s.split_whitespace().count() }

    #[test]
    fn test_token_savings() {
        let input = include_str!("../tests/fixtures/cmd_raw.txt");
        let output = filter_cmd(input);
        let savings = 100.0 - (count_tokens(&output) as f64 / count_tokens(input) as f64 * 100.0);
        assert!(savings >= 60.0, "...");
    }
}
```

**模式 3：集成测试（#[ignore]）**
```rust
#[test]
#[ignore]
fn test_real_git_log() {
    let output = std::process::Command::new("rtk")
        .args(&["git", "log", "-10"])
        .output()?;
    assert!(output.status.success());
    assert!(stdout.len() < 5000, "Output too large");
}
```

**模式 4：Shell 转义跨平台测试**
```rust
#[cfg(target_os = "windows")]
const EXPECTED_SHELL: &str = "cmd.exe";
#[cfg(target_os = "macos")]
const EXPECTED_SHELL: &str = "zsh";
```

### 5. TOML 内联测试（rtk verify）

运行时测试套件 `rtk verify` 输出：
```
PASS  native binary hook registered in settings.json
145/145 tests passed
```

145 个 TOML filter 内联测试全部通过，覆盖了所有内置 filter。

### 6. 测试质量评估

| 维度 | 评价 | 证据 |
|------|------|------|
| 覆盖率 | 好 | 2065 测试，所有核心模块有测试 |
| 数据真实性 | 好 | 16+ 个真实命令输出夹具 |
| Clap 参数测试 | 优秀 | 覆盖各种参数组合和边缘情况 |
| Token 节省断言 | 好 | ≥60% 硬约束，但部分 filter 没有（如 npm） |
| 跨平台测试 | 中等 | 使用 `#[cfg]` 但主要在 macOS/Linux 运行 |
| 快照测试 | 待完善 | insta 在代码中有引用但 snapshots 目录为空 |

## 结论与洞见

1. **测试规模良好**：2065 测试量级对于 50+ 源文件的项目是健康的
2. **Token 节省断言是独特价值**：将商业指标编码为测试约束
3. **TOML 内联测试框架创新**：非 Rust 开发者也能为 filter 写测试
4. **夹具覆盖可扩展**：当前覆盖 4 个生态系统，但 git/cargo/js 等核心 filter 使用内联数据
5. **`--require-all` 适合 CI**：强制每个 filter 都有测试
