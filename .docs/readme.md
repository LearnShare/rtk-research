# RTK 研究文档目录

> RTK (Rust Token Killer) — CLI 代理，减少 LLM Token 消耗 60-90%
>
> 版本：v0.42.2（源码）| 交付：**45 份文档** | 日期：2026-06-06

---

## 📋 研究管理

| 文件 | 说明 |
|------|------|
| [研究计划 v1](research-plan-v1.md) | 7 阶段 21 任务，广度覆盖（✅ 全部完成） |
| [研究计划 v2](research-plan-v2.md) | 6 阶段进阶计划，纵深+实操（✅ 全部完成） |
| [研究规则与目标](CLAUDE.md) | 文件规范、执行规则、项目上下文索引 |
| [交接文档](handover/task-handover-20260606-114424.md) | 全对话记录 + 环境快照 + 后续计划 |

---

## 🏗️ 架构分析

RTK 的整体架构设计、模块划分和核心数据流。

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [命令路由流程](architecture/command-routing-flow.md) | 深入 | main.rs 3275 行：Cli 解析、Commands 枚举、run_fallback 三层回退、安全白名单 |
| [模块架构全景](architecture/module-architecture.md) | 深入 | Mermaid 依赖图、8 大模块职责矩阵、数据流方向 |
| [Core 基础设施](architecture/core-infrastructure.md) | 深度 | 14 个 core 组件：config、tracking、filter、toml_filter、tee、telemetry |
| [命令过滤模块对比](architecture/command-filter-comparison.md) | 分析 | 9 生态 30+ filter 共通模式、策略对比、TOML vs Rust-native 决策标准 |
| [测试体系策略](architecture/testing-strategy.md) | 分析 | 3 层测试体系、insta 快照、token savings 硬约束、夹具策略 |

---

## ⚙️ 核心过滤引擎

RTK 的过滤核心技术：双轨引擎、流式处理、结构化解析。

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [过滤引擎深度解剖](engine/filter-engine-deep-dive.md) | 深度 | 双轨设计：Rust-native 状态机 + TOML DSL 8 阶段管道、lazy_static 策略、4 种压缩 |
| [TOML DSL 语言设计](engine/toml-dsl-language-design.md) | 深度 | 8 阶段管道引擎、build.rs 嵌入式、60+ filter 三层优先级、内联测试框架 |
| [Stream 流式处理](unexplored/stream-module.md) | 分析 | StreamFilter Trait、LineHandler（行级）+ BlockHandler（块级）双模式 |
| [Parser 解析器](unexplored/parser-module.md) | 分析 | ParseResult 三级降解、extract_json_object 大括号平衡、TokenFormatter |

---

## 🔗 Hook 与命令重写

RTK 与 AI 编程工具的集成层：命令拦截、重写、权限管理。

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [Hook 系统与命令重写](engine/hook-system-analysis.md) | 深度 | 13 Agent 适配、rewrite exit code 契约（0/1/2/3）、复合命令拆分 |
| [Discover 发现引擎](unexplored/discover-deep-dive.md) | 深度 | 74 条重写规则、18 个忽略前缀、复合命令拆分算法、不可审核构造检测 |
| [权限引擎审计](security/command-injection-audit.md) | 审计 | permissions.rs 998 行 + 42 测试、Deny/Ask/Allow 优先级、#1213 修复 |

---

## 🛡️ 安全模型

RTK 的四层安全防御体系。

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [命令注入面分析](security/command-injection-audit.md) | 审计 | 4 层防御、permissions.rs 规则引擎、复合命令分段、换行逃逸检测 |
| [TOML 沙箱与供应链安全](security/supply-chain-integrity.md) | 审计 | TOML DSL 天生沙箱、SHA-256 完整性链、27 依赖审计、unsafe 仅 2 处 |
| [项目 Trust 机制](engine/project-filter-trust-mechanism.md) | 深度 | 信任前加载模型、SHA-256 哈希链、double 验证、CI 覆盖 |

---

## 📊 Token 追踪与计费

Token 节省的度量、追踪和经济分析。

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [Token 追踪系统](engine/token-tracking-analysis.md) | 深度 | SQLite Schema、RAII TimedExecution、项目级 GLOB 匹配、90 天轮换 |
| [Claude Code 经济学](unexplored/remaining-modules.md#ccusage) | 速览 | ccusage 集成、CC 消费 + RTK 节省 ROI 计算模型 |
| [性能基准测试](performance/system-benchmark.md) | 数据 | hyperfine 实测：RTK 启动 91.6ms、过滤开销 ~67ms、二进制 8.2MB |

---

## ✅ 测试与质量

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [单元测试与快照分析](quality/unit-test-snapshot-analysis.md) | 分析 | 2065 测试全通过、4 种测试模式、16 个夹具、TOML 内联测试 145 |
| [集成测试报告](quality/integration-test-report.md) | 数据 | 7 个 #[ignore] 测试、端到端压缩率实测 |
| [错误处理模式统计](depth/error-handling-patterns.md) | 统计 | .context() 159 处、unwrap_or_else 60 处、lazy_static unwrap 246 处 |

---

## 🛠️ 实操产出

亲手编写的 filter 和完整实现记录。

| 文件 | 类型 | 核心内容 |
|------|------|----------|
| [TOML filter 实操：ipconfig](practice/toml-filter-workshop.md) | 🔧 TOML | 10 行 TOML + 2 个内联测试、构建时验证、编码问题记录 |
| [Rust filter 实操：netstat](practice/rust-filter-workshop.md) | 🔧 Rust | 164→15 行压缩（92%）、6 个测试、main.rs 4 处修改全流程 |

### 实物文件

| 文件 | 路径 | 说明 |
|------|------|------|
| `ipconfig.toml` | `rtk/src/filters/ipconfig.toml` | 内置 TOML filter，strip 空行 + max_lines |
| `netstat_cmd.rs` | `rtk/src/cmds/system/netstat_cmd.rs` | Rust filter，按连接状态分组聚合 |

---

## 🌐 生态系统分析

RTK 在 AI 编程工具生态中的定位和发展。

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [项目演进历史](ecosystem/project-evolution-history.md) | 数据 | 40+ 版本时间线、20+ 贡献者、1415 commits、关键里程碑 |
| [社区健康度分析](ecosystem/community-health-analysis.md) | 分析 | 13 Agent 集成、7 语言 README、Discord 1500+ 成员、文档质量评估 |
| [外部文章综述](ecosystem/external-articles-review.md) | 综述 | 10+ 篇技术文章摘要与社区反响 |
| [版本差异分析](depth/version-diff-and-cross-platform.md) | 分析 | v0.37.2→v0.42.2：301 commits、17K 新增、跨平台 cfg(unix) 16 处 |

---

## ⚖️ 对比与趋势

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [同类工具对比矩阵](comparison/tool-comparison-matrix.md) | 分析 | RTK vs Shell 管道 vs settings.json vs 内置压缩（5 维度） |
| [Token 优化技术趋势](comparison/token-optimization-trends.md) | 研判 | "Token 即带宽"范式、上下文窗口影响、Open Source vs Proprietary |

---

## 📖 综合报告与知识沉淀

| 文件 | 篇幅 | 核心内容 |
|------|------|----------|
| [核心研究报告](report/rtk-deep-research-report.md) | 完整 | 8 章完整技术报告：架构/引擎/Hook/追踪/测试/生态/展望 |
| [内部架构白皮书](synthesis/rtk-internal-white-paper.md) | 精要 | 7 个关键架构决策、安全模型、扩展点一览 |
| [知识图谱速查](report/rtk-knowledge-map.md) | 速查 | 模块索引、命令速查（20+）、配置参考、开发命令备忘 |
| [技术债务与改进建议](synthesis/technical-debt.md) | 分析 | 12 项改进建议（高/中/低优先级） |
| [二次开发实施方案](synthesis/secondary-dev-plan.md) | 指南 | 方向评估、贡献路径（上游/fork）、开发环境 |
| [贡献指南](report/rtk-contribution-guide.md) | 指南 | 新 filter 步骤、测试策略、PR 流程、编码规则速查 |

---

## 🧩 未探模块速览

v1 未覆盖的剩余模块快速总结。

| 文件 | 覆盖内容 |
|------|----------|
| [Learn 模块 — CLI 纠正检测](unexplored/learn-module.md) | 500 行规则引擎、6 种错误分类、Jaccard 相似度、CORRECTION_WINDOW=3 |
| [剩余组件速览](unexplored/remaining-modules.md) | args_utils 双线参数还原、runner 通用骨架、hooks/analytics 剩余、cmds 边缘 filter |

---

## 统计总览

| 指标 | 值 |
|------|-----|
| 研究总阶段 | 13 阶段（v1: 7 + v2: 6）✅ |
| 交付物总数 | 45 份文档 |
| 源码分析量 | ~15K Rust + 60+ TOML filter |
| 测试验证 | 2065 测试全通过（v1） |
| 实操产出 | TOML filter 1 + Rust filter 1（8 个新测试） |
| 代码修改 | main.rs 3 处 + 2 个新源文件 |
| 主题覆盖 | 架构 / 引擎 / 安全 / 生态 / 对比 / 实操 / 综合 |

> 🔗 所有文件使用相对路径引用，可在文件系统中直接浏览
