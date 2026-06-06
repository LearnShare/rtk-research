# toml-dsl-language-design — RTK TOML DSL 语言设计分析

> **研究日期**：2026-06-06
> **研究阶段**：P2.4
> **研究方法**：源码分析 + 语法设计研究

## 概述
分析 RTK 的 TOML DSL 过滤语言设计：8 阶段管道、build.rs 嵌入式 TOML、60+ 内置 filter 覆盖、用户可扩展的优先级链。

## 核心发现

1. **构建时嵌入式**：`build.rs` 合并 `src/filters/*.toml` 为单个 TOML 文件，编译进二进制
2. **8 阶段声明式管道**：从 ANSI 剥离到空结果兜底，全声明式配置
3. **三层优先级链**：项目级 → 用户级 → 内置级，支持分场景覆盖
4. **内联测试框架**：每个 filter 自带 `[[tests.*]]` 定义，构建时 + 运行时双重验证
5. **60+ 内置 filter 覆盖长尾**：涵盖 brew/df/du/gcc/helm/jq/make/ps/rsync/ssh 等运维工具

## 详细分析

### 1. 构建时嵌入式机制（build.rs）

```
src/filters/
├── ansible-playbook.toml
├── basedpyright.toml
├── biome.toml
├── brew-install.toml
├── ...
└── yamllint.toml         (60+ 个 .toml 文件)
        │
        ▼
    build.rs
    1. read_dir → filter *.toml → sort alphabetically
    2. concat → "schema_version = 1\n\n# --- file.toml ---\n..."
    3. TOML parse validate
    4. 检查重复 filter 名称
    5. write → OUT_DIR/builtin_filters.toml
        │
        ▼
    src/core/toml_filter.rs
    const BUILTIN_TOML: &str = include_str!(concat!(env!("OUT_DIR"), "/builtin_filters.toml"));
```

**构建时验证**：TOML 解析失败 → 编译失败，防止无效配置进入二进制。

### 2. 查找优先级链

```
rtk make install
    │
    ▼
1. .rtk/filters.toml (项目级)
   ├── match_command = "^make\\b"
   ├── 匹配 → 使用项目级规则
   ├── 未匹配 → 继续
   └── ⚠️ 需要 rtk trust 才能启用
    │
    ▼
2. ~/.config/rtk/filters.toml (用户级)
   ├── 匹配 → 使用用户级规则
   └── 未匹配 → 继续
    │
    ▼
3. BUILTIN_TOML (嵌入式，60+ 内置)
   ├── 匹配 → 使用内置规则
   └── 未匹配 → 继续
    │
    ▼
4. Passthrough (原样透传)
```

项目级 filter 覆盖内置 filter 时有警告：
```
[rtk] warning: filter 'make' is shadowing a built-in filter
```

### 3. TOML Filter 完整 Schema

```toml
[filters.my-tool]
description = "Short description of what this filter does"
match_command = "^my-tool\\b"              # (必填) 匹配命令的正则

# Stage 1: ANSI 剥离
strip_ansi = true                          # (可选, 默认 false)

# 合并 stderr 一起过滤
filter_stderr = true                       # (可选, 默认 false)

# Stage 2: 行级正则替换（链式，顺序执行）
replace = [
  { pattern = "some pattern", replacement = "replacement" },
]

# Stage 3: 全文匹配短路
match_output = [
  { pattern = "nothing to be done", message = "my-tool: ok" },
  { pattern = "error pattern", message = "my-tool: error", unless = "warning" },
]

# Stage 4: 行过滤（二选一或都不设）
strip_lines_matching = [
  "^\\s*$",                               # 空行
  "^\\[\\d+\\/\\d+\\]",                   # 进度行
]
keep_lines_matching = [
  "^error",
  "^warning",
]

# Stage 5: 行截断
truncate_lines_at = 80                     # (可选) 每行最多 80 字符

# Stage 6: 头/尾保留
head_lines = 30                           # (可选) 保留前 30 行
tail_lines = 10                           # (可选) 保留后 10 行

# Stage 7: 绝对行数上限
max_lines = 40                            # (可选) 最多 40 行

# Stage 8: 空结果兜底
on_empty = "my-tool: ok (no output)"      # (可选)

# ── 内联测试 ──
[[tests.my-tool]]
name = "empty output case"
input = "my-tool: Nothing to be done."
expected = "my-tool: ok"

[[tests.my-tool]]
name = "error output"
input = "my-tool: error: file not found"
expected = "my-tool: error: file not found"
```

### 4. 管道实现引擎核心类型

```rust
enum LineFilter {
    None,                              // 不过滤行
    Strip(RegexSet),                   // 匹配的行被删除
    Keep(RegexSet),                    // 只保留匹配的行
}

struct CompiledFilter {
    match_regex: Regex,                 // 命令匹配
    strip_ansi: bool,
    replace: Vec<CompiledReplaceRule>,  // 链式替换
    match_output: Vec<CompiledMatchOutputRule>,  // 短路匹配
    line_filter: LineFilter,           // 行过滤
    truncate_lines_at: Option<usize>,
    head_lines: Option<usize>,
    tail_lines: Option<usize>,
    max_lines: Option<usize>,
    on_empty: Option<String>,
    filter_stderr: bool,
}
```

### 5. 60+ 内置 Filter 分类

从 `src/filters/` 分类统计：

| 类别 | 数量 | 代表 filter |
|------|------|-------------|
| **系统/文件** | 7 | `df`, `du`, `ps`, `stat`, `lsof`, `ping`, `rsync` |
| **构建工具** | 8 | `make`, `gcc`, `mvn-build`, `gradle`, `dotnet-build`, `swift-build` |
| **容器/云** | 4 | `helm`, `skopeo`, `ssh`, `systemctl-status` |
| **静态分析** | 5 | `shellcheck`, `yamllint`, `hadolint`, `markdownlint`, `oxlint` |
| **包管理器** | 6 | `brew-install`, `bundle-install`, `composer-install`, `poetry-install`, `pip` |
| **Infra/Terraform** | 4 | `terraform-plan`, `tofu-init`, `tofu-plan`, `tofu-validate`, `tofu-fmt` |
| **数据库** | 2 | `liquibase`, `sqlite` |
| **其他** | 24+ | `jira`, `jq`, `mise`, `nx`, `ollama`, `sops`, `turbo`, ... |

### 6. TOML vs Rust-native 决策（来自 CONTRIBUTING.md）

| 场景 | 选择 | 原因 |
|------|------|------|
| 输出需要结构重组（JSON/diff 解析） | Rust-native | TOML 只能做行级操作 |
| 需要状态机（跨行上下文的聚合） | Rust-native | TOML 管道的行级处理 |
| 纯行过滤 + 截断 | TOML DSL | 声明式配置，无需编写 Rust 代码 |
| 长尾小众工具 | TOML DSL | 添加 filter 只需编写 `.toml` 文件 |

**TOML filter 的限制**：
- 不能跨行聚合（无法将一组相关的行合并为总结）
- 不能执行外部命令或 API 调用
- 只能线性处理，无分支逻辑
- 正则匹配是行级别或全文级别，无结构化解析

## 结论与洞见

1. **DSL 设计简洁有效**：8 阶段管道覆盖了 80% 的过滤场景
2. **构建时验证是亮点**：编译失败比运行时错误好得多
3. **RegexSet 批处理优化**：多个 strip/keep 模式单次扫描，优于逐个 regex
4. **内联测试降低贡献门槛**：`[[tests.*]]` 让非 Rust 开发者也能测试
5. **60+ 覆盖面广但单文件平均 ~20 行**：每个 filter 都是声明式的，易于维护
