# command-walkthrough — RTK 命令功能全景体验

> **研究日期**：2026-06-06
> **研究阶段**：P0.2
> **研究方法**：实际操作 + 输出对比

## 概述
逐一运行 RTK 全部命令族，记录输出样式、过滤效果和发现的问题。

## 核心发现

1. **RTK v0.37.2 功能可用**：大部分命令正常运作，输出清晰紧凑
2. **验证体系完备**：`rtk verify` 145/145 测试通过
3. **命令重写部分受限**：`rtk rewrite` 对 `npm install` 等未覆盖命令正确返回 exit=1
4. **有未信任的项目过滤器警告**：`.rtk/filters.toml` 存在但未信任
5. **Session 追踪显示 37% RTK 采用率**：仍有提升空间

## 详细分析

### 文件操作命令

| 命令 | 状态 | 效果 | 备注 |
|------|------|------|------|
| `rtk ls` | ⚠️ 空输出 | 可能需要参数 | 无参数时无输出，`rtk ls -la .` 正常 |
| `rtk tree -L 2 src` | ⚠️ 空输出 | 可能需要参数 | 输出为空 |
| `rtk read Cargo.toml -n` | ✅ | 带行号紧凑输出 | 支持 line_numbers 选项 |
| `rtk find -name "*.toml"` | ✅ | 紧凑列表输出 | 显示 60+ 个 TOML 文件 |
| `rtk wc Cargo.toml` | ✅ | `72L 246W 1796B` | 极简输出，无表格装饰 |
| `rtk diff file1 file2` | ✅ | 紧凑 unified diff | 仅显示变更行 |

### Git 命令

| 命令 | 状态 | 效果 | 备注 |
|------|------|------|------|
| `rtk git status` | ✅ | 仓库状态简化为单行 | `clean — nothing to commit` |
| `rtk git log -5` | ✅ | 紧凑单行日志 | 含相对时间 + 作者 |
| `rtk git branch` | ✅ | 分组显示 | current/remote-only 分组 |

### 分析与系统命令

| 命令 | 状态 | 效果 | 备注 |
|------|------|------|------|
| `rtk env --filter PATH` | ✅ | 路径变量压缩显示 | 长路径截断并标注字符数 |
| `rtk deps ./rtk` | ✅ | 依赖分组显示 | Rust 21 依赖 |
| `rtk json Cargo.toml` | ✅ | 错误提示清晰 | 正确识别 TOML 非 JSON |
| `rtk grep "fn run"` | ❌ | 错误：`--path` 不识别 | v0.37.2 参数格式变化 |
| `rtk summary "echo build ok"` | ✅ | 启发式摘要 | "Build successful" |
| `rtk log` (stdin) | ✅ | 日志分类摘要 | 按 error/warn/info 分组 |

### 代理与分析命令

| 命令 | 状态 | 效果 | 备注 |
|------|------|------|------|
| `rtk proxy echo "hello"` | ✅ | 透传输出 | 不过滤，仅追踪 |
| `rtk pipe --filter grep` | ✅ | stdin 管道过滤 | 正常透传 |
| `rtk gain` | ✅ | 完整统计面板 | 146 命令，12.5% 节省 |
| `rtk gain --history` | ✅ | 历史摘要 | 最近命令列表 |
| `rtk discover` | ✅ | 节省机会分析 | 0 session 可用（本地数据少） |
| `rtk session` | ✅ | Session 概况 | 6 session，平均 32% 采用率 |

### Hook 与配置命令

| 命令 | 状态 | 效果 | 备注 |
|------|------|------|------|
| `rtk rewrite "git status"` | ✅ | → `rtk git status` | exit=3（非标准行为） |
| `rtk rewrite "npm install"` | ✅ | 无输出 | exit=1 = 无匹配（预期） |
| `rtk verify` | ✅ | 145/145 tests passed | 完整验证通过 |
| `rtk config` | ✅ | 显示配置 | 含 tracking/display/filters/tee/hooks/limits |
| `rtk init --show` | ⬜ 未测 | | P0.3 中测 |

### TOML 过滤器引擎

| 命令 | 状态 | 效果 | 备注 |
|------|------|------|------|
| `rtk trust` | ⬜ 未测 | | P2.5 中测 |
| TOML fallback | ⚠️ 警告 | `.rtk/filters.toml` 未信任 | 需运行 `rtk trust` |

### 未测试的命令（需要特定环境）

- `rtk docker`, `rtk kubectl` — 需要 Docker 环境
- `rtk cargo test/build` — 需要 cargo 项目编译
- `rtk npm`, `rtk pnpm`, `rtk go` — 需要对应环境
- `rtk gh`, `rtk glab` — 需要 GitHub/GitLab CLI
- `rtk aws`, `rtk psql`, `rtk curl`, `rtk wget` — 需要网络环境

## 结论与洞见

1. **命令行覆盖率 ~70%**：大部分核心命令在无特殊环境要求下可运行
2. **输出风格统一**：紧凑、无表格框线、关键信息优先
3. **版本差异影响**：`rtk grep` 的 `--path` 参数在 v0.37.2 中不工作（v0.42.2 源码中已修复）
4. **安全管理严格**：未信任的项目 TOML filter 会被拒绝执行
5. **`rtk rewrite` exit code 行为**：git status 返回 exit=3 而非 0，可能是 v0.37.2 的 bug
