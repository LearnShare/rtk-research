# RTK 研究文档目录

> RTK (Rust Token Killer) v0.42.2 深度研究 — 2026-06-06
>
> 本目录包含全部研究交付物，按主题分类。点击文件名查看详情。

---

## 📋 研究计划

| 文件 | 说明 |
|------|------|
| [研究计划](research-plan-v1.md) | 7 阶段 21 任务，含状态跟踪 |
| [研究指南](CLAUDE.md) | 文件规范、执行规则、项目上下文 |

---

## 🔧 P0 — 环境搭建与快速上手

| 文件 | 说明 |
|------|------|
| [编译验证](build-verification.md) | Rust 工具链验证、RTK 版本确认、二进制体积 |
| [命令全景实测](command-walkthrough.md) | 20+ 命令逐一运行记录、过滤效果评价 |
| [Hook 安装测试](hook-installation-test.md) | PreToolUse Hook 安装/重写/卸载全链路验证 |

---

## 🏗️ P1 — 架构源码深度分析

| 文件 | 说明 |
|------|------|
| [命令路由流程](architecture/command-routing-flow.md) | main.rs 3275 行分析：Clap 解析、fallback 链路、安全白名单 |
| [模块架构全景](architecture/module-architecture.md) | 8 大模块依赖图、数据流方向、职责矩阵 |
| [Core 基础设施](architecture/core-infrastructure.md) | 14 个 core 组件：config/tracking/filter/toml_filter/tee/telemetry |
| [命令过滤模块对比](architecture/command-filter-comparison.md) | 9 个生态 30+ filter 的共通模式与策略对比 |
| [测试体系策略](architecture/testing-strategy.md) | 三层测试体系：单元测试 + 快照测试 + 集成测试 |

---

## ⚙️ P2 — 核心机制专项研究

| 文件 | 说明 |
|------|------|
| [过滤引擎深度解剖](engine/filter-engine-deep-dive.md) | Rust-native + TOML DSL 双轨引擎、lazy_static 策略、四种压缩策略 |
| [Hook 系统与命令重写](engine/hook-system-analysis.md) | 13 种 Agent Hook 适配、rewrite exit code 契约、SHA-256 完整性 |
| [Token 追踪与计费分析](engine/token-tracking-analysis.md) | SQLite Schema、RAII 计时器、项目级 GLOB 匹配、90 天轮换 |
| [TOML DSL 语言设计](engine/toml-dsl-language-design.md) | 8 阶段管道、build.rs 嵌入式、60+ 内置 filter 覆盖、三层优先级 |
| [项目过滤器与 Trust 机制](engine/project-filter-trust-mechanism.md) | 信任前加载模型、SHA-256 哈希链、双重验证、CI 覆盖 |

---

## ✅ P3 — 测试与质量保障体系

| 文件 | 说明 |
|------|------|
| [单元测试与快照](quality/unit-test-snapshot-analysis.md) | 2065 测试全量分析、四种测试模式、TOML 内联测试 |
| [集成测试报告](quality/integration-test-report.md) | 7 个 #[ignore] 测试、端到端压缩率实测 |
| [性能基准测试](quality/performance-benchmark.md) | 二进制体积、编译时间、启动延迟、性能目标对照 |

---

## 🌐 P4 — 生态系统与社区分析

| 文件 | 说明 |
|------|------|
| [项目演进历史](ecosystem/project-evolution-history.md) | 40+ 版本时间线、20+ 贡献者分布、关键里程碑 |
| [社区健康度分析](ecosystem/community-health-analysis.md) | 13 种 Agent 集成、多语言文档、Discord 生态 |
| [外部文章综述](ecosystem/external-articles-review.md) | 10+ 篇技术文章摘要与社区反响 |

---

## ⚖️ P5 — 横向对比与趋势研判

| 文件 | 说明 |
|------|------|
| [同类工具对比矩阵](comparison/tool-comparison-matrix.md) | RTK vs Shell 管道 vs settings.json vs 内置压缩 |
| [Token 优化技术趋势](comparison/token-optimization-trends.md) | "Token 即带宽" 范式、上下文窗口影响、趋势研判 |

---

## 📖 P6 — 综合报告

| 文件 | 说明 |
|------|------|
| [核心研究报告](report/rtk-deep-research-report.md) | 完整技术报告：架构/过滤引擎/Hook/追踪/测试/生态/展望 |
| [知识图谱速查](report/rtk-knowledge-map.md) | 模块索引、命令速查、配置参考、开发备忘 |
| [贡献指南](report/rtk-contribution-guide.md) | 新 filter 实现步骤、测试策略、PR 流程、编码规则 |

---

## 统计总览

| 指标 | 值 |
|------|-----|
| 研究阶段 | 7 / 7 ✅ |
| 交付物总数 | 21 份 |
| 覆盖源码文件 | 50+ Rust 源文件 |
| 代码量分析 | ~15K Rust + 60+ TOML |
| 测试验证 | 2065 passed |
| 过滤模块 | 30+ Rust-native + 60+ TOML |
| AI Agent 覆盖 | 13 种 |

> 💡 阅读建议：按阶段顺序（P0 → P6）阅读以建立完整知识体系，或直接点击感兴趣的主题文件。
>
> 🔗 所有交付物使用相对路径引用，可在 GitHub / 本地文件系统中直接浏览。
