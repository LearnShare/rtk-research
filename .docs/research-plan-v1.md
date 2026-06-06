# rtk-research-plan-v1

> **研究目标**：系统性地深入理解 RTK（Rust Token Killer）的架构设计、核心机制、生态系统及最佳实践，形成可指导二次开发和贡献的技术知识体系。
>
> **目标仓库**：`C:\works\cc-test\rtk`（v0.42.2，`github.com/rtk-ai/rtk`）
>
> **研究根目录**：`.docs/`
>
> **制定日期**：2026-06-06

---

## 状态图例

| 标记 | 含义 | 说明 |
|------|------|------|
| `⬜` | pending | 尚未开始 |
| `🏗️` | in-progress | 当前正在执行 |
| `✅` | completed | 已完成并交付 |
| `⏸️` | blocked | 被阻塞 |

---

## 阶段总览

| # | 阶段 | 状态 | 周期 | 前置 | 交付目录 |
|---|------|------|------|------|----------|
| P0 | 环境搭建与快速上手 | ✅ | 1 天 | 无 | `.docs/` |
| P1 | 架构源码深度分析 | ✅ | 3 天 | P0 | `.docs/architecture/` |
| P2 | 核心机制专项研究 | ✅ | 4 天 | P1 | `.docs/engine/` |
| P3 | 测试与质量保障体系 | ⬜ | 2 天 | P1 | `.docs/quality/` |
| P4 | 生态系统与社区分析 | ✅ | 2 天 | P1 | `.docs/ecosystem/` |
| P5 | 横向对比与趋势研判 | ✅ | 1 天 | P2, P4 | `.docs/comparison/` |
| P6 | 研究报告撰写与知识沉淀 | ✅ | 2 天 | P0–P5 | `.docs/report/` |

**总预估**：约 15 天 | **进度**：✅ 7/7 阶段 | **当前**：全部完成

---

## P0：环境搭建与快速上手

> **目标**：搭建可编译、可调试的 RTK 开发环境，形成对工具的第一手使用体验。

### P0.1 编译与安装

| 项目 | 内容 |
|------|------|
| **状态** | ✅ completed |
| **研究方法** | 实际操作 |
| **任务描述** | 验证已安装 RTK 环境，确认 Rust 工具链版本 |
| **具体步骤** | 1. 确认 Rust 工具链版本（1.96.0 ≥ 1.91）✓<br>2. 验证已安装 RTK v0.37.2<br>3. 验证 `rtk gain` 正常<br>4. 测量二进制体积 8.17 MB<br>5. ⚠️ 源码 v0.42.2 vs 二进制 v0.37.2 有版本差异 |
| **交付内容** | ✅ `.docs/build-verification.md` |

### P0.2 功能全景体验

| 项目 | 内容 |
|------|------|
| **状态** | ✅ completed |
| **研究方法** | 实际操作 + 文档对照 |
| **任务描述** | 逐一运行 RTK 所有命令族，记录输出样式和过滤效果 |
| **覆盖范围** | 20+ 命令，覆盖文件/系统/git/分析/hook/配置 6 大类 |
| **交付内容** | ✅ `.docs/command-walkthrough.md` |

### P0.3 Hook 安装与端到端测试

| 项目 | 内容 |
|------|------|
| **状态** | ✅ completed |
| **研究方法** | 实际操作 + 源码跟踪 |
| **任务描述** | 安装 PreToolUse hook，验证命令重写链路完整 |
| **具体步骤** | 1. `rtk init --show` 验证 hook 安装 ✓<br>2. 验证 settings.json `PreToolUse` hook ✓<br>3. 测试 `rtk rewrite "git status"` ✓<br>4. 测试 `rtk rewrite "cargo test && git push"` 复合命令 ✓<br>5. ⚠️ `--dry-run` 在 v0.37.2 中不支持 |
| **交付内容** | ✅ `.docs/hook-installation-test.md` |

---

## P1：架构源码深度分析

> **目标**：从源码层面完整理解 RTK 的整体架构、模块划分和数据流。

### P1.1 入口与命令路由系统

| 项目 | 内容 |
|------|------|
| **状态** | 🏗️ in-progress |
| **研究方法** | 静态源码分析 |
| **核心文件** | `rtk/src/main.rs`（3276 行） |
| **分析重点** | 1. `Cli` 结构体与 `Commands` 枚举设计<br>2. 全局 flags（`--verbose`, `--ultra-compact`, `--skip-env`）传递机制<br>3. `run_fallback()` 的 TOML 过滤回退链路<br>4. `is_operational_command()` 安全白名单机制<br>5. SIGPIPE 信号处理<br>6. `RTK_META_COMMANDS` 设计意图 |
| **交付内容** | ⬜ `.docs/architecture/command-routing-flow.md` |

### P1.2 模块架构全景

| 项目 | 内容 |
|------|------|
| **状态** | ✅ completed |
| **交付内容** | ✅ `.docs/architecture/module-architecture.md` |

### P1.3 Core 基础设施分析

| 项目 | 内容 |
|------|------|
| **状态** | ✅ completed |
| **交付内容** | ✅ `.docs/architecture/core-infrastructure.md` |

### P1.4 命令过滤模块分析

| 项目 | 内容 |
|------|------|
| **状态** | ✅ completed |
| **交付内容** | ✅ `.docs/architecture/command-filter-comparison.md` |

### P1.5 测试体系分析

| 项目 | 内容 |
|------|------|
| **状态** | ✅ completed |
| **交付内容** | ✅ `.docs/architecture/testing-strategy.md` |

---

## P2：核心机制专项研究

> **目标**：深入研究 RTK 最核心的技术机制。

### P2.1 过滤引擎深度解剖

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 源码追踪 + Debug 输出分析 |
| **分析重点** | 1. Rust-native 过滤：正则构建、行处理、状态机设计<br>2. TOML 过滤管道：从 `toml_filter.rs` 到 `build.rs` 嵌入式资源<br>3. `lazy_static!` 正则编译策略与性能分析<br>4. Filter fallback 机制（`unwrap_or_else`）<br>5. 四种压缩策略实现对比：Smart Filtering / Grouping / Truncation / Deduplication |
| **具体步骤** | 1. 使用 `-vvv` 跟踪过滤过程<br>2. 针对 `git log` 逐行跟踪过滤管道<br>3. 分析 `regex` crate 的使用模式 |
| **交付内容** | ⬜ `.docs/engine/filter-engine-deep-dive.md` |

### P2.2 Hook 系统与命令重写

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 源码分析 + 实验测试 |
| **核心文件** | `src/hooks/*`, `src/discover/registry.rs` |
| **分析重点** | 1. 7+ 种 AI 代理 Hook 适配对比<br>2. `rtk rewrite` 单一真相源设计<br>3. 重写注册表（`registry.rs`）和重写规则（`rules.rs`）<br>4. `init.rs` 安装/卸载流程<br>5. `integrity.rs` SHA-256 完整性校验<br>6. `trust.rs` 项目信任机制<br>7. 复合命令处理（`&&`, `\|`） |
| **交付内容** | ⬜ `.docs/engine/hook-system-analysis.md` |

### P2.3 Token 追踪与计费分析

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 源码分析 + SQLite 数据库审计 |
| **分析重点** | 1. 数据库 Schema（`commands` 表结构）<br>2. `TimedExecution` 集成到过滤路径<br>3. Token 统计算法（input vs output, savings %）<br>4. 项目级追踪（GLOB 模式匹配）<br>5. 90 天轮换清理策略<br>6. `gain.rs` 聚合查询优化<br>7. `cc_economics.rs` Claude Code 经济学 |
| **具体步骤** | 1. `rtk gain --all --format json` 导出数据<br>2. 直接查询 SQLite DB 验证<br>3. 分析 `discover/lexer.rs` 发现算法 |
| **交付内容** | ⬜ `.docs/engine/token-tracking-analysis.md` |

### P2.4 TOML 过滤 DSL 语言设计

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 源码分析 + 语法设计研究 |
| **分析重点** | 1. `toml_filter.rs` 的 DSL 解析与执行（`FilterRule`, `MatchOutputRule`, `ReplaceRule`）<br>2. 8 阶段管道设计<br>3. `build.rs` 构建时 TOML 合并与验证<br>4. 60+ 内置 filter 覆盖率分析<br>5. TOML vs Rust-native 决策标准<br>6. 过滤器查找优先级链 |
| **交付内容** | ⬜ `.docs/engine/toml-dsl-language-design.md` |

### P2.5 项目级 TOML 过滤器与 Trust 机制

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 源码分析 + 实验配置 |
| **分析重点** | 1. `.rtk/filters.toml` 加载与优先级<br>2. `rtk trust` / `untrust` 安全机制<br>3. `trust.rs` 信任数据库管理<br>4. `rtk verify` 内联测试框架<br>5. 安全考量：TOML 执行沙箱、路径遍历防护 |
| **交付内容** | ⬜ `.docs/engine/project-filter-trust-mechanism.md` |

---

## P3：测试与质量保障体系

> **目标**：系统理解 RTK 的质量保障策略。

### P3.1 单元测试与快照测试

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 源码分析 + 运行测试 |
| **任务描述** | 运行全部测试，分析测试覆盖模式 |
| **具体步骤** | 1. `cargo test --all` 运行全部测试并记录结果<br>2. 分析 `insta` 快照测试模式<br>3. 分析 Token 节省断言（`count_tokens()` + ≥60%）<br>4. 评估测试夹具质量 |
| **交付内容** | ⬜ `.docs/quality/unit-test-snapshot-analysis.md` |

### P3.2 集成测试与端到端验证

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 实际操作 |
| **任务描述** | 运行 `#[ignore]` 集成测试，评估真实命令过滤效果 |
| **具体步骤** | 1. `cargo test --ignored -- --test-threads=1`<br>2. 手动对比 raw vs RTK 输出<br>3. `hyperfine 'rtk git status' 'git status' --warmup 3`<br>4. 测量内存占用 |
| **交付内容** | ⬜ `.docs/quality/integration-test-report.md` |

### P3.3 性能基准测试

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 基准测试工具 |
| **具体步骤** | 1. 安装 `hyperfine`<br>2. 对 10+ 核心命令建立性能基线（<10ms）<br>3. 测量各命令输出压缩率<br>4. 分析 TOML vs Rust-native 延迟差异<br>5. 二进制体积测量（<5MB） |
| **交付内容** | ⬜ `.docs/quality/performance-benchmark.md` |

---

## P4：生态系统与社区分析

> **目标**：理解 RTK 在 AI 编程辅助工具生态中的定位。

### P4.1 项目演进历史分析

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | Git 历史分析 |
| **具体步骤** | 1. 分析 `CHANGELOG.md` 版本演进<br>2. 关键里程碑：v0.1 → v0.42 的功能演进<br>3. 从 git log 分析贡献者分布<br>4. 版本发布节奏和策略 |
| **交付内容** | ⬜ `.docs/ecosystem/project-evolution-history.md` |

### P4.2 GitHub 社区生态

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 数据分析 + 文档调研 |
| **分析重点** | 1. Issue / PR 趋势分析<br>2. 核心贡献者网络<br>3. 用户反馈和使用案例<br>4. Discord 社群生态<br>5. 文档质量评估（多语言 README、架构文档、使用指南） |
| **交付内容** | ⬜ `.docs/ecosystem/community-health-analysis.md` |

### P4.3 外部文章与社区内容收集

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 搜索 + 摘要整理 |
| **已发现资源** | - [CSDN 深度拆解](https://blog.csdn.net/weixin_36829761/article/details/161060542)<br>- [Jimmy Song 技术介绍](https://jimmysong.io/zh/ai/rtk/)<br>- [WaveSpeed 博客](https://wavespeed.ai/blog/posts/what-is-rtk-token-efficiency/)<br>- [daily.dev 推荐](https://app.daily.dev/posts/cli-proxy-that-reduces-llm-token-consumption-by-60-90-on-common-dev-commands-rqzedtufl)<br>- [Dev.to 实践指南](https://dev.to/arshtechpro/how-rtk-reduces-llm-token-usage-for-ai-coding-agents-2kfd)<br>- [Hacker News 讨论](https://news.ycombinator.com/from?site=github.com%2frtk-ai) |
| **任务描述** | 深入阅读以上文章，提取技术洞见和社区观点 |
| **交付内容** | ⬜ `.docs/ecosystem/external-articles-review.md` |

---

## P5：横向对比与趋势研判

> **目标**：将 RTK 放在更大的 AI 工程化工具生态中对比分析。

### P5.1 同类工具对比

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 调研 + 对比分析 |
| **对比维度** | 1. RTK vs `reachingforthejack/rtk`（Rust Type Kit）名称混淆<br>2. RTK vs `.claude/settings.json` 原生过滤<br>3. RTK vs shell-level 过滤（`sed/awk`）<br>4. RTK vs 编译时 AST 过滤<br>5. RTK vs AI Agent 内置 output compaction |
| **评估维度** | 功能覆盖、性能开销、易用性、可扩展性、成熟度 |
| **交付内容** | ⬜ `.docs/comparison/tool-comparison-matrix.md` |

### P5.2 Token 优化技术趋势

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **研究方法** | 行业调研 |
| **分析重点** | 1. "Context-layer optimization" 概念的兴起<br>2. Token 压缩技术定位<br>3. 大模型上下文窗口扩展对 RTK 价值的影响<br>4. 从 CLI 代理到通用代理的演进可能<br>5. Proprietary vs Open Source 方案对比 |
| **交付内容** | ⬜ `.docs/comparison/token-optimization-trends.md` |

---

## P6：研究报告撰写与知识沉淀

> **目标**：将所有研究成果整合为完整的技术知识体系。

### P6.1 核心研究报告

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **交付内容** | ⬜ `.docs/report/rtk-deep-research-report.md` |
| **章节规划** | 1. 项目概述与背景<br>2. 架构设计与模块分解<br>3. 过滤引擎原理详解<br>4. Hook 系统与集成架构<br>5. Token 追踪与经济模型<br>6. 测试与质量保障策略<br>7. 性能基准与优化空间<br>8. 社区生态与项目健康度<br>9. 横向对比与定位分析<br>10. 未来展望与建议 |

### P6.2 知识图谱与备忘单

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **交付内容** | ⬜ `.docs/report/rtk-knowledge-map.md` |
| **内容规划** | - 模块速查表（模块名 → 功能 → 入口文件）<br>- 命令覆盖矩阵（支持的命令 + 过滤效果）<br>- 配置文件参考（config.toml, filters.toml）<br>- 常用开发命令备忘录 |

### P6.3 贡献指南整合

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ pending |
| **交付内容** | ⬜ `.docs/report/rtk-contribution-guide.md` |
| **内容规划** | - 新命令过滤模块实现步骤清单<br>- TOML filter 编写指南<br>- 测试夹具创建标准<br>- PR 提交流程速查 |

---

## 附录

### A. 目录结构约定

```
.docs/
├── research-plan-v1.md              # 研究计划
├── CLAUDE.md                        # 研究规则和目标
├── build-verification.md            # P0.1 编译验证
├── command-walkthrough.md           # P0.2 功能全景体验
├── hook-installation-test.md        # P0.3 Hook 安装验证
├── architecture/                    # P1 架构分析交付物
│   ├── command-routing-flow.md
│   ├── module-architecture.md
│   ├── core-infrastructure.md
│   ├── command-filter-comparison.md
│   └── testing-strategy.md
├── engine/                          # P2 核心机制交付物
│   ├── filter-engine-deep-dive.md
│   ├── hook-system-analysis.md
│   ├── token-tracking-analysis.md
│   ├── toml-dsl-language-design.md
│   └── project-filter-trust-mechanism.md
├── quality/                         # P3 测试质量交付物
│   ├── unit-test-snapshot-analysis.md
│   ├── integration-test-report.md
│   └── performance-benchmark.md
├── ecosystem/                       # P4 生态分析交付物
│   ├── project-evolution-history.md
│   ├── community-health-analysis.md
│   └── external-articles-review.md
├── comparison/                      # P5 对比分析交付物
│   ├── tool-comparison-matrix.md
│   └── token-optimization-trends.md
└── report/                          # P6 综合报告交付物
    ├── rtk-deep-research-report.md
    ├── rtk-knowledge-map.md
    └── rtk-contribution-guide.md
```

### B. 参考资源索引

| 类型 | 名称 | 链接 |
|------|------|------|
| 仓库 | RTK GitHub | `https://github.com/rtk-ai/rtk` |
| 官网 | rtk-ai.app | `https://www.rtk-ai.app` |
| 架构 | ARCHITECTURE.md | `rtk/docs/contributing/ARCHITECTURE.md` |
| 技术 | TECHNICAL.md | `rtk/docs/contributing/TECHNICAL.md` |
| 功能 | FEATURES.md | `rtk/docs/usage/FEATURES.md` |
| Hook | Hook README | `rtk/src/hooks/README.md` |
| TOML 过滤器 | Filters README | `rtk/src/filters/README.md` |
| 命令模块 | Cmds README | `rtk/src/cmds/README.md` |
| 编码规范 | Core README | `rtk/src/core/README.md` |
| 社区 | Discord | `https://discord.gg/RySmvNF5kF` |

### C. 研究工具清单

| 工具 | 用途 |
|------|------|
| `hyperfine` | 命令行性能基准测试 |
| `cargo test` + `insta` | 快照测试管理 |
| `/usr/bin/time -l` | 内存使用测量 |
| `clippy` | 代码质量分析 |
| `cargo-outdated` | 依赖版本检查 |
| SQLite 客户端 | DB schema 探索 |

### D. 术语表

| 术语 | 定义 |
|------|------|
| Proxy Mode | RTK 不过滤输出，仅记录使用统计（`rtk proxy <cmd>`） |
| Passthrough | 未匹配任何过滤规则的命令直接透传 |
| TOML DSL | 基于 TOML 的声明式过滤管道，8 阶段处理 |
| Rust-native filter | 用 Rust 代码编写的命令特定过滤器 |
| PreToolUse | Claude Code Hook 机制，在命令执行前拦截 |
| Auto-Rewrite | Hook 自动重写命令（100% 覆盖率） |
| Suggest | Hook 通过系统提示建议使用 RTK（70-85% 覆盖率） |
| Tee | 失败时保存原始输出以便恢复查看 |
| FilterLevel | `None`/`Minimal`/`Aggressive` 三级过滤强度 |

---

> **执行说明**：从 **P0** 开始逐阶段执行。每完成一个任务，将状态更新为 `✅`，交付物保存到指定路径。所有文件采用 `kebab-case` 命名，引用使用相对路径。
