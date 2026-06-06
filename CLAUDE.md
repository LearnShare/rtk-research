# .docs/CLAUDE.md — RTK 研究项目指南

> **目标**：系统性地深入理解 RTK（Rust Token Killer）并形成可指导二次开发的技术知识体系。

---

## 研究范围

- **研究对象**：[rtk-ai/rtk](https://github.com/rtk-ai/rtk) v0.42.2
- **研究根目录**：`C:\works\cc-test\`
- **RTK 代码位置**：`C:\works\cc-test\rtk\`（develop 分支）
- **研究计划**：`.docs/research-plan-v1.md`

---

## 文件规范

### 命名约定

所有交付物文件统一使用 **kebab-case** 命名：
```
✅ .docs/architecture/command-routing-flow.md
✅ .docs/engine/filter-engine-deep-dive.md
✅ .docs/report/rtk-knowledge-map.md
❌ .docs/architecture/命令路由流程.md
❌ .docs/engine/FilterEngineDeepDive.md
```

### 目录结构

交付物按主题分类保存：

| 目录 | 内容 |
|------|------|
| `.docs/architecture/` | P1 架构分析交付物 |
| `.docs/engine/` | P2 核心机制交付物 |
| `.docs/quality/` | P3 测试质量交付物 |
| `.docs/ecosystem/` | P4 生态分析交付物 |
| `.docs/comparison/` | P5 对比分析交付物 |
| `.docs/report/` | P6 综合报告交付物 |

### 交付物模板

每个交付物文件遵循以下结构：

```markdown
# title — 简短副标题

> **研究日期**：YYYY-MM-DD
> **研究阶段**：P0.x / P1.x / ...
> **研究方法**：源码分析 / 实际操作 / 对比研究 / ...

## 概述
<!-- 一句话说明本文档覆盖的内容 -->

## 核心发现
<!-- 关键结论列表 -->

## 详细分析
<!-- 按逻辑分段展开 -->

## 结论与洞见
<!-- 总结 + 待进一步探索的问题 -->
```

---

## 研究执行规则

1. **按计划顺序执行**：P0 → P1 → ... → P6，除非有明确的独立前提
2. **每完成一个任务即保存交付物**：不积压，不留到批量处理
3. **交付物使用相对路径引用**：`[架构文档](../architecture/command-routing-flow.md)`
4. **每个交付物标注研究日期和方法**：便于追溯
5. **遇到阻塞时记录在阻点日志中**：不跳过，`⚠️` 标记
6. **发现值得深入的方向用 `🔍` 标记**：作为 P6 综合报告的素材

---

## RTK 研究关键上下文

### 项目概况
- **作者**：Patrick Szymkowiak
- **许可证**：Apache 2.0
- **语言**：Rust（edition 2021，MSRV 1.91）
- **无 async**：单线程设计，启动 <10ms
- **核心目标**：通过智能过滤压缩 CLI 输出，为 LLM 节省 60-90% Token

### 核心模块一览

| 模块 | 路径 | 职责 |
|------|------|------|
| 入口路由 | `rtk/src/main.rs` | Clap 解析 → Commands 枚举分发 + fallback 逻辑 |
| 命令过滤 | `rtk/src/cmds/*/` | 各生态系统的 Rust-native 过滤器 |
| 核心基础设施 | `rtk/src/core/` | 配置、追踪、工具函数、过滤引擎、TOML DSL |
| Hook 系统 | `rtk/src/hooks/` | 7+ AI 代理的 Hook 适配 |
| 分析统计 | `rtk/src/analytics/` | Token 节省分析、Claude Code 经济学 |
| 发现引擎 | `rtk/src/discover/` | Claude Code 历史分析，发现遗漏的节省机会 |
| 学习模块 | `rtk/src/learn/` | CLI 错误纠正检测 |
| TOML 过滤器 | `rtk/src/filters/` | 60+ 内置 TOML 声名式过滤配置 |

### 关键技术决策
- **过滤引擎**：双轨制 — Rust-native（复杂逻辑）+ TOML DSL（简单声明式），`CONTRIBUTING.md` 中有决策标准
- **Hook 策略**：Auto-Rewrite（默认，100% 覆盖）vs Suggest（非侵入，70-85% 覆盖）
- **安全模型**：`is_operational_command()` 白名单 + `integrity.rs` SHA-256 校验 + `trust.rs` 项目信任机制
- **追踪系统**：SQLite 本地存储，90 天自动清理，项目级 GLOB 模式匹配
- **构建优化**：LTO + codegen-units=1 + panic=abort + strip，目标二进制 <5MB

---

## 参考链接

- [研究计划](research-plan-v1.md)
- [RTK 仓库本地副本](../rtk/)
- [RTK 架构文档](../rtk/docs/contributing/ARCHITECTURE.md)
- [RTK 技术文档](../rtk/docs/contributing/TECHNICAL.md)
- [RTK GitHub](https://github.com/rtk-ai/rtk)
- [RTK 官网](https://www.rtk-ai.app)
