# rtk-knowledge-map — RTK 知识图谱与速查手册

> **研究日期**：2026-06-06
> **研究阶段**：P6.2
> **研究方法**：综合整理

## 模块速查表

| 模块 | 功能 | 入口文件 | 代码量 |
|------|------|----------|--------|
| 入口路由 | CLI 解析 + fallback | `src/main.rs` | 3275 行 |
| 配置 | RtkConfig 加载/保存 | `src/core/config.rs` | 200+ 行 |
| 追踪 | SQLite Token 统计 | `src/core/tracking.rs` | 500+ 行 |
| 工具函数 | ANSI strip / truncate / exec | `src/core/utils.rs` | 150+ 行 |
| 过滤引擎 | 语言感知过滤 | `src/core/filter.rs` | 120+ 行 |
| TOML DSL | 8 阶段管道引擎 | `src/core/toml_filter.rs` | 400+ 行 |
| Tee | 失败恢复 | `src/core/tee.rs` | 200+ 行 |
| 遥测 | 匿名使用统计 | `src/core/telemetry.rs` | 150+ 行 |
| Git 过滤 | git/gh/glab/gt/diff | `src/cmds/git/git.rs` | 2847 行 |
| Cargo 过滤 | cargo/npm/pnpm | `src/cmds/rust/cargo_cmd.rs` | 2216 行 |
| Hook Init | 安装/卸载 | `src/hooks/init.rs` | 600+ 行 |
| Rewrite | 命令重写 | `src/hooks/rewrite_cmd.rs` | 100+ 行 |
| Integrity | SHA-256 校验 | `src/hooks/integrity.rs` | 200+ 行 |
| Trust | 信任管理 | `src/hooks/trust.rs` | 200+ 行 |
| 发现引擎 | 历史分析 | `src/discover/registry.rs` | 300+ 行 |
| Gain | 统计面板 | `src/analytics/gain.rs` | 400+ 行 |
| 过滤器配置 | 60+ 内置 | `src/filters/*.toml` | 各 10-30 行 |

## 命令速查

### 文件操作
```bash
rtk ls [args]           # 目录列表（树形）
rtk tree [args]         # 目录树
rtk read [-n] <file>    # 文件读取（可选行号）
rtk find [args]         # 文件查找
rtk wc <file>           # 词/行/字符统计
```

### Git 操作
```bash
rtk git status          # 压缩状态
rtk git log [-N]        # 单行日志
rtk git diff [args]     # 压缩 diff
rtk git branch          # 分组分支列表
rtk gc add/commit/push  # 简化为 "ok"
```

### 代码质量
```bash
rtk cargo {build,test,clippy,check}
rtk ruff [check,format]
rtk lint [args]         # eslint
rtk prettier [args]
rtk format [args]       # 通用格式化器检测
```

### 系统工具
```bash
rtk grep <pattern> [path]
rtk env [--filter NAME]
rtk deps [path]
rtk json <file> [--depth N]
rtk log [file]          # 日志去重
rtk pipe [--filter NAME] # stdin 管道模式
```

### 分析统计
```bash
rtk gain                # Token 节省总览
rtk gain --history      # 历史命令
rtk gain --daily        # 每日分解
rtk gain --all --format json
rtk discover            # 发现节省机会
rtk session             # Session 采用率
```

### Hook 与配置
```bash
rtk init --show         # 查看 Hook 状态
rtk init --global       # 安装全局 Hook
rtk init --uninstall    # 卸载
rtk rewrite "cmd"       # 测试重写效果
rtk verify              # 完整性 + TOML 测试
rtk trust               # 信任项目 filter
rtk config              # 查看/创建配置
```

## 配置参考

### config.toml
```toml
[tracking]
enabled = true
history_days = 90

[display]
colors = true
emoji = true
max_width = 120

[filters]
ignore_dirs = [".git", "node_modules", "target", "__pycache__"]
ignore_files = ["*.lock", "*.min.js"]

[tee]
enabled = true
mode = "failures"
max_files = 20
max_file_size = 1048576

[hooks]
exclude_commands = []

[limits]
grep_max_results = 200
status_max_files = 15
passthrough_max_chars = 2000
```

### .rtk/filters.toml
```toml
schema_version = 1

[filters.my-tool]
description = "My custom filter"
match_command = "^my-tool\\b"
strip_lines_matching = ["verbose lines"]
head_lines = 30
on_empty = "my-tool: ok"
```

### 环境变量
```bash
RTK_DISABLED=1                  # 完全禁用 RTK
RTK_NO_TOML=1                   # 跳过 TOML 引擎
RTK_TOML_DEBUG=1                # TOML 调试信息
RTK_TRUST_PROJECT_FILTERS=1     # CI 信任覆盖
RTK_TELEMETRY_DISABLED=1        # 禁用遥测
RTK_DB_PATH=<path>              # 数据库路径覆盖
RTK_TEE_DIR=<dir>               # Tee 目录覆盖
```

## 开发命令备忘

```bash
# 构建
cargo build                    # 调试构建
cargo build --release          # 发布构建（LTO + strip）
cargo check                    # 仅检查

# 测试
cargo test --all               # 全部测试（2065）
cargo test --ignored           # 集成测试
rtk verify                     # TOML 测试

# 质量
cargo fmt --all                # 格式化
cargo clippy --all-targets     # Lint
cargo test --all               # 自动化所有
```
