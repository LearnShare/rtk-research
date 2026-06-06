# version-diff-and-cross-platform — v0.37.2 vs v0.42.2 差异 & 跨平台分析

> **研究日期**：2026-06-06
> **研究阶段**：Q4.2, Q4.4
> **研究方法**：git diff + cfg 分析

## 版本差异

### 规模变化

| 指标 | v0.37.2 (4月) | v0.42.2 (6月) | 变化 |
|------|-------------|-------------|------|
| 提交数 | 基线 | +301 commits | +301 |
| 变更文件 | — | 136 文件 | +136 |
| 代码新增 | — | 17,214 行 | +17K |
| 代码删除 | — | 3,262 行 | -3.2K |
| 净增长 | — | 13,952 行 | +14K |

### 关键新功能

| 功能 | 说明 | 引入版本 |
|------|------|----------|
| 项目级追踪 `--project` | `rtk gain --project` 支持 CWD 过滤 | v0.38+ |
| `--dry-run` 预览 | `rtk init --dry-run` 不写入仅预览 | v0.40+ |
| `--auto-patch` / `--no-patch` | settings.json 自动打补丁 | v0.40+ |
| 透明前缀 `transparent_prefixes` | Docker exec 等包装命令 | v0.41+ |
| Copilot 集成 | `rtk init --copilot` | v0.41+ |
| 环境变量覆盖 | `RTK_TRUST_PROJECT_FILTERS=1` | v0.42 |
| `find_state_index` 等过滤改进 | 更精确的列定位 | 多版本 |

## 跨平台行为分析

### cfg(unix) 分布（16 处）

| 文件 | 用途 | 数量 |
|------|------|------|
| `src/main.rs` | SIGPIPE 信号处理 + proxy 信号处理 | 2 |
| `src/core/utils.rs` | `resolved_command` 路径解析 | 2 |
| `src/core/telemetry.rs` | 数据目录路径 | 1 |
| `src/core/stream.rs` | 命令执行 | 2 |
| `src/hooks/integrity.rs` | Hook 文件路径权限 0o444 | 4 |
| `src/hooks/init.rs` | Hook 安装目录、脚本格式、权限设置 | 5 |

### 关键差异

| 维度 | Windows | Unix (macOS/Linux) |
|------|---------|-------------------|
| SIGPIPE | 不适用（Windows 无 SIGPIPE） | `libc::signal(SIGPIPE, SIG_DFL)` |
| 进程创建开销 | ~50-100ms 额外延迟 | ~5-10ms |
| 路径格式 | `\` + 驱动器号 | `/` |
| Hook 脚本 | 不适用（Windows 使用原生 command hook） | `.sh` 脚本文件 |
| `which` crate | 支持 Windows PATHEXT | 标准 PATH 搜索 |
| 权限设置 | 不适用 | `chmod 0o444` hash 文件 |
| Shell | `cmd /C` | `sh -c` |
| 数据目录 | `%APPDATA%/rtk/` | `~/.local/share/rtk/` |

### 跨平台建议

1. Windows 比 Unix 慢 3-5x（进程创建延迟），非 RTK 自身问题
2. 无 Windows 特定的 `#[cfg(windows)]` 代码，所有 Windows 适配通过 `cfg(unix)` + 默认路径处理
3. `dirs` crate 处理平台数据目录差异
4. CI 应同时运行 3 个平台的测试
