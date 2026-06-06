# rtk-research-plan-v3 — 实战与社区贡献

> **研究目标**：在 v1（广度覆盖）+ v2（纵深实操）基础上，走向社区贡献、真实场景验证和长期洞察。
>
> **前置基础**：✅ v1 7 阶段 + ✅ v2 6 阶段 = 45 份交付物
>
> **制定日期**：2026-06-06 | **🔵 优先级：低（暂不执行）**

---

## 状态图例

| 标记 | 含义 |
|------|------|
| `⬜` | pending |
| `🏗️` | in-progress |
| `✅` | completed |
| `⏸️` | blocked |
| `🔵` | deferred (低优先级，暂不执行) |

---

## v1+v2 回顾 — 已覆盖 vs 未覆盖

### 已覆盖

```
✅ 静态源码分析（架构/引擎/Hook/追踪/TOML DSL/安全）
✅ 实操产出（TOML filter + Rust filter）
✅ 性能基准（hyperfine 测量）
✅ 生态系统与社区分析
✅ 对比与趋势研判
✅ 安全审计（命令注入/完整性/供应链）
✅ 知识整合（白皮书/技术债务/二次开发方案）
```

### 未覆盖 — v3 重点

| 盲区 | 重要性 | 原因 |
|------|--------|------|
| 实际提交 PR 到上游 | 🔴 高 | 验证整个研究是否真正可落地 |
| 跨平台实测（Linux/macOS） | 🔴 高 | 全部测试在 Windows 上 |
| 真实 LLM 集成端到端测试 | 🔴 高 | 未用实际 Claude Code 验证 |
| GitHub Issues/PRs 社区审计 | 🟡 中 | 了解社区活跃度和贡献机会 |
| CI/CD 流水线分析 | 🟡 中 | 了解发布工程 |
| 代码变动热点分析（churn） | 🟡 中 | 高变动文件 = 风险区域 |
| 已知 BUG/FIXME 追踪 | 🟡 中 | 代码中的待解决项 |
| 长期使用现场研究 | 🟢 低 | 需要数天到数周 |
| 过滤质量对 LLM 输出的影响 | 🔴 高 | 过滤是否丢失关键信息？ |

---

## 阶段总览

| # | 阶段 | 状态 | 周期 | 交付目录 |
|---|------|------|------|----------|
| R0 | 贡献实战：提交 PR | 🔵 | 2 天 | `.docs/contribution/` |
| R1 | 跨平台验证 | 🔵 | 2 天 | `.docs/cross-platform/` |
| R2 | 社区脉动分析 | 🔵 | 1 天 | `.docs/community/` |
| R3 | 代码健康度审计 | 🔵 | 1 天 | `.docs/quality/` |
| R4 | 过滤效果实证研究 | 🔵 | 2 天 | `.docs/empirical/` |

**总预估**：约 8 天 | **状态**：🔵 全部延期 — 低优先级，留存为后续参考

---

## R0：贡献实战 — 提交 PR

> **目标**：将研究成果转化为实际的上游贡献，完整经历 fork → commit → PR → review 流程。

### R0.1 提交 ipconfig TOML filter

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | 实际操作 |
| **具体步骤** | 1. 在 GitHub 上 fork rtk-ai/rtk<br>2. clone fork<br>3. cherry-pick ipconfig.toml<br>4. cargo test + rtk verify 确认通过<br>5. git push + 创建 PR<br>6. 跟踪 PR review 过程<br>7. 记录全部经验 |
| **交付物** | ⬜ `.docs/contribution/pr-ipconfig.md` |

### R0.2 提交 netstat Rust filter（可选）

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **交付物** | ⬜ `.docs/contribution/pr-netstat.md` |

### R0.3 PR Review 体验

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | 实际操作 |
| **任务描述** | 审查 3-5 个上游的开放 PR，应用研究积累的知识做技术评审 |
| **交付物** | ⬜ `.docs/contribution/pr-review-experience.md` |

---

## R1：跨平台验证

> **目标**：在 Linux 和 macOS 上验证 RTK 行为一致性，测量真实的启动时间和性能基线。

### R1.1 Linux (WSL) 性能基线

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | 实际测试 |
| **具体步骤** | 1. WSL 中安装 Rust + RTK<br>2. hyperfine 测量启动时间<br>3. 对比 Windows vs WSL 性能差异<br>4. 验证 SIGPIPE 处理<br>5. 测试 cfg(unix) 路径 |
| **交付物** | ⬜ `.docs/cross-platform/linux-benchmark.md` |

### R1.2 Shell 兼容性测试

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **交付物** | ⬜ `.docs/cross-platform/shell-compatibility.md` |

---

## R2：社区脉动分析

> **目标**：深入 GitHub 社区，了解真实问题和贡献机会。

### R2.1 Issues 分类与趋势

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | 数据分析 |
| **分析重点** | 1. 开放 Issue 分类（bug/feature/question）<br>2. 常见问题模式<br>3. 修复周期统计<br>4. 可贡献的 "good first issue" |
| **交付物** | ⬜ `.docs/community/issue-analysis.md` |

### R2.2 开放 PR 审计

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | 代码审查 |
| **分析重点** | 1. 开放 PR 分类<br>2. Review 模式<br>3. CI 通过率<br>4. 合并时间统计 |
| **交付物** | ⬜ `.docs/community/pr-audit.md` |

---

## R3：代码健康度审计

> **目标**：利用静态分析工具评估代码质量。

### R3.1 代码变动热点分析

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | git churn 分析 |
| **已知热点** | main.rs(25)、init.rs(24)、hook_cmd.rs(15)、permissions.rs(13)、git.rs(13)、registry.rs(12) |
| **交付物** | ⬜ `.docs/quality/hotspot-analysis.md` |

### R3.2 BUG/FIXME 追踪

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | 源码审计 |
| **已知项** | - `pipe filter` 节省率低于 60%（`pipe_cmd.rs`）<br>- `FIXME: pnpm workspace filters`（`main.rs`）<br>- `BUG: "wc " in IGNORED_PREFIXES`（`registry.rs`） |
| **交付物** | ⬜ `.docs/quality/bug-todo-audit.md` |

### R3.3 代码复杂度分析

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | `cargo metrics` or `tokei` |
| **交付物** | ⬜ `.docs/quality/complexity-analysis.md` |

---

## R4：过滤效果实证研究

> **目标**：定量回答 "RTK 过滤是否影响 LLM 输出质量？"—— 这是 RTK 最核心的假设检验。

### R4.1 过滤信息损失率

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | 实验设计 |
| **具体步骤** | 1. 选取 5 个核心命令（git status、cargo test、ruff check、ls、env）<br>2. 同时捕获 raw output 和 RTK filtered output<br>3. 人工标注 "关键信息是否保留"<br>4. 计算每条命令的信息保留率<br>5. 识别常见的丢失信息模式 |
| **交付物** | ⬜ `.docs/empirical/information-loss-rate.md` |

### R4.2 LLM 输出质量对比实验

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | A/B 实验 |
| **具体步骤** | 1. 设计 10 个标准开发场景（debug、review、status check、build）<br>2. 分别用 raw 输出和 RTK 输出提供给同一个 LLM<br>3. 对比 LLM 响应的准确性和完整性<br>4. 统计 RTK 是否导致 LLM 遗漏关键信息 |
| **交付物** | ⬜ `.docs/empirical/llm-output-quality.md` |

### R4.3 长期追踪分析

| 项目 | 内容 |
|------|------|
| **状态** | 🔵 |
| **研究方法** | 数据收集 |
| **具体步骤** | 1. 在 5 个工作日内持续使用 RTK<br>2. 收集 `rtk gain --all --format json` 数据<br>3. 分析命令分布、节省趋势、效率变化<br>4. 识别未被覆盖的命令（`rtk discover`）<br>5. 计算实际 ROI |
| **交付物** | ⬜ `.docs/empirical/field-study.md` |

---

## 附录

### 为什么是这些方向？

| 方向 | 价值 | v1+v2 未覆盖原因 |
|------|------|-----------------|
| 提交 PR | 验证研究是否可转化为实际贡献 | 需要账号、流程、review 交互 |
| 跨平台 | Windows-only 是最大的方法论缺陷 | 无 Linux/macOS 环境 |
| 社区审计 | 了解 RTK 的真实用户痛点 | 需要 GitHub API 访问 |
| 代码健康度 | 从静态分析到动态趋势 | 需要 git 历史数据挖掘 |
| 过滤质量 | 回答 "Validity of Token savings" 的根本问题 | 需要 LLM 实验设计 |

### 建议执行顺序

```
R3.2 BUG 追踪（最快，1 小时）
  → R2 社区脉动（1 天）
    → R0.1 PR 提交（1 天）
      → R1 跨平台（2 天，如环境可用）
        → R4 实证研究（2 天，最重要的独立研究）
```

### v3 核心问题

```
"RTK 真的能帮助 LLM 写得更好代码吗？"
  → R4.1 信息损失率：过滤掉了关键信息吗？
  → R4.2 LLM 输出质量：LLM 的响应质量下降了吗？
  → R4.3 ROI 分析：节省的 Token 值不值得？
```

> 如以上方向已足够覆盖，可按序执行 R0→R4。
