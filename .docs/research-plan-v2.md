# rtk-research-plan-v2 — 进阶深度研究

> **研究目标**：在 v1 全面认知（21 交付物）基础上，深入未覆盖的模块、实践操作和前沿方向，达到可独立贡献和二次开发的水平。
>
> **前置基础**：✅ v1 全部 7 阶段完成（`.docs/research-plan-v1.md`）
>
> **制定日期**：2026-06-06

---

## 状态图例

| 标记 | 含义 |
|------|------|
| `⬜` | pending |
| `🏗️` | in-progress |
| `✅` | completed |
| `⏸️` | blocked |

---

## 阶段总览

| # | 阶段 | 状态 | 周期 | 前置 | 交付目录 |
|---|------|------|------|------|----------|
| Q0 | 盲区探测与实操突破 | ✅ | 2 天 | v1 P2 | `.docs/practice/`, `.docs/unexplored/` |
| Q1 | 安全审计与攻击面分析 | 🏗️ | 2 天 | v1 P2 | `.docs/security/` |
| Q2 | 性能剖析与优化研究 | ✅ | 2 天 | v1 P3 | `.docs/performance/` |
| Q3 | 未探模块深度分析 | ✅ | 2 天 | v1 P1 | `.docs/unexplored/` |
| Q4 | 纵深专项研究 | ✅ | 3 天 | Q0–Q3 | `.docs/depth/` |
| Q5 | 知识缝合与二次开发准备 | ✅ | 2 天 | Q0–Q4 | `.docs/synthesis/` |

**总预估**：约 12 天 | **进度**：✅ 6/6 阶段 全部完成

---

## v1 回顾 — 已覆盖与未覆盖

### 已覆盖（21 交付物，不再重复）

`architecture/`（5）`engine/`（5）`quality/`（3）`ecosystem/`（3）`comparison/`（2）`report/`（3）

### 未覆盖 — v2 重点

| 盲区 | 重要性 | 原因 |
|------|--------|------|
| `learn/` 学习模块 | 🔴 高 | v1 完全未触及 |
| `parser/` 解析器模块 | 🔴 高 | 架构拼图缺失 |
| `stream.rs` 流式处理 | 🔴 高 | 过滤引擎的关键组件 |
| `discover/` 发现引擎深度 | 🟡 中 | v1 仅分析 registry |
| 实战：编写 TOML filter | 🔴 高 | 验证扩展性 |
| 实战：编写 Rust filter | 🔴 高 | 验证贡献流程 |
| 安全审计 | 🔴 高 | 生产安全评估 |
| 性能基准系统测试 | 🟡 中 | v1 只做 basic |
| 跨平台行为 | 🟡 中 | v1 全部 Windows |
| 依赖分析与二进制体积 | 🟢 低 | 工程优化 |

---

## Q0：盲区探测与实操突破

> **目标**：覆盖 v1 完全未触及的模块，并通过亲手编写过滤器验证对扩展机制的理解。

### Q0.1 learn 学习模块 — CLI 纠正检测

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 源码分析 + 数据模拟 |
| **核心文件** | `src/learn/detector.rs`, `src/learn/report.rs`, `src/learn/mod.rs` |
| **分析重点** | 1. 检测算法（如何判断 CLI 命令使用错误）<br>2. 错误模式匹配策略<br>3. 与 Claude Code 历史数据的交互<br>4. `rtk learn` 命令行为<br>5. CLI 纠正规则的生成与格式 |
| **交付物** | ⬜ `.docs/unexplored/learn-module.md` |

### Q0.2 parser 解析器模块

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 源码分析 |
| **核心文件** | `src/parser/formatter.rs`, `src/parser/types.rs`, `src/parser/mod.rs` |
| **分析重点** | 1. 格式化器设计<br>2. 类型系统<br>3. 与 TOML filter 和 Analytics 的协作关系<br>4. 在过滤管线中的位置 |
| **交付物** | ⬜ `.docs/unexplored/parser-module.md` |

### Q0.3 stream 流式处理模块

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 源码分析 |
| **核心文件** | `src/core/stream.rs` |
| **分析重点** | 1. `LineHandler` Trait + `LineStreamFilter` 设计模式<br>2. `FilterMode` / `StdinMode` 枚举含义<br>3. `exec_capture()` 与 `CaptureResult` 结构<br>4. 哪些 filter 使用 stream 架构（git、cargo...）<br>5. 与 tee 的协作 |
| **交付物** | ⬜ `.docs/unexplored/stream-module.md` |

### Q0.4 discover 发现引擎深度分析

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 源码分析 + 运行观察 |
| **核心文件** | `src/discover/lexer.rs`, `src/discover/provider.rs`, `src/discover/report.rs`, `src/discover/rules.rs` |
| **分析重点** | 1. `lexer.rs` shell 令牌化算法<br>2. `rules.rs` 完整规则体系（~80 条）<br>3. `provider.rs` 数据源（ccusage 等）<br>4. `report.rs` 发现报告格式<br>5. 全部 IGNORED / RULES 常量 |
| **交付物** | ⬜ `.docs/unexplored/discover-deep-dive.md` |

### Q0.5 实操：编写一个 TOML filter

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 动手实践 |
| **任务描述** | 为本地系统的一个工具创建 `src/filters/*.toml` |
| **具体步骤** | 1. 选择目标（`ping` 已有，选一个未覆盖的工具）<br>2. 捕获真实命令输出<br>3. 编写 filter 规则<br>4. 添加 `[[tests.*]]` 内联测试<br>5. `cargo test` + `rtk verify` 验证<br>6. 记录编写过程和设计决策 |
| **交付物** | ⬜ `.docs/practice/toml-filter-workshop.md`（含 filter 文件实例） |

### Q0.6 实操：编写一个 Rust filter

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 动手实践 |
| **任务描述** | 实现一个简单的 Rust-native filter |
| **具体步骤** | 1. 设计 filter：选择一个命令（如 `helm list` 的紧凑版）<br>2. 创建 `_cmd.rs` 模块文件<br>3. 实现 run() + filter_output()<br>4. 创建测试夹具 + 快照测试<br>5. 注册到 main.rs Commands 枚举<br>6. 运行 `cargo fmt + clippy + test`<br>7. 记录完整开发过程 |
| **交付物** | ⬜ `.docs/practice/rust-filter-workshop.md`（含完整代码实现） |

---

## Q1：安全审计与攻击面分析

> **目标**：系统评估 RTK 的安全模型，识别潜在风险并提出加固方案。

### Q1.1 命令注入面分析

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 源码审计 + 攻击建模 |
| **分析重点** | 1. `resolved_command()` 路径解析安全<br>2. shell 转义在 proxy/pipe 中的覆盖<br>3. `permissions.rs` 完整规则集<br>4. TOCTOU 风险（配置文件在加载后篡改）<br>5. `RTK_META_COMMANDS` 的边界覆盖 |
| **交付物** | ⬜ `.docs/security/command-injection-audit.md` |

### Q1.2 TOML filter 沙箱评估

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 源码审计 |
| **分析重点** | 1. `toml_filter.rs` 正则拒绝服务风险（ReDoS）<br>2. `.rtk/filters.toml` 覆盖优先级链的安全含义<br>3. `match_output` 短路规则被滥用可能<br>4. `regex` crate 的 `ReDoS` 防护能力<br>5. `RegexSet` 批处理的性能安全边界 |
| **交付物** | ⬜ `.docs/security/toml-filter-sandbox.md` |

### Q1.3 完整性校验体系验证

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 实验验证 |
| **分析重点** | 1. SHA-256 校验在 `settings.json` 修改时的行为<br>2. hook 文件被篡改后的实际表现<br>3. trust 数据库被删除/篡改的恢复路径<br>4. `runtime_check()` 调用时机与 error handling<br>5. `is_operational_command()` fail-open 的实际含义 |
| **交付物** | ⬜ `.docs/security/integrity-verification.md` |

### Q1.4 供应链与依赖安全

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 依赖审计 |
| **分析重点** | 1. 依赖树深度分析（`cargo tree` + `cargo audit`）<br>2. 不安全 crate 标识<br>3. `unsafe_code = "deny"` 的实际执行情况<br>4. rusqlite bundled 模式的安全含义<br>5. 凭据处理（`ANTHROPIC_AUTH_TOKEN` 等） |
| **交付物** | ⬜ `.docs/security/supply-chain-audit.md` |

### Q1.5 安全加固建议报告

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 综合分析 |
| **任务描述** | 汇总全部安全发现，按严重性分级，提出加固建议 |
| **交付物** | ⬜ `.docs/security/security-recommendations.md` |

---

## Q2：性能剖析与优化研究

> **目标**：通过专业的性能工具和分析方法，评估 RTK 性能特征。

### Q2.1 系统性能基准

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 基准测试 |
| **工具** | `hyperfine`, `/usr/bin/time` |
| **具体步骤** | 1. 安装 hyperfine<br>2. 对 15+ 核心命令建立性能基线（startup <10ms）<br>3. TOML filter vs Rust-native 过滤延迟对比<br>4. 原始输出 vs 过滤输出字节数/Token 数对比<br>5. 二进制体积精确测量（`ls -lh` + `file`） |
| **交付物** | ⬜ `.docs/performance/system-benchmark.md` |

### Q2.2 启动时间纵深分析

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | Profiling + 实验 |
| **分析重点** | 1. 统计 `lazy_static!` 正则的首次编译开销<br>2. Config::load() 文件 I/O 延迟<br>3. SQLite 连接打开时间<br>4. Clap 解析时间<br>5. Windows 进程创建开销的影响 |
| **交付物** | ⬜ `.docs/performance/startup-analysis.md` |

### Q2.3 构建系统与优化分析

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 构建实验 |
| **分析重点** | 1. Release profile：`lto=true` + `codegen-units=1` + `panic=abort` + `strip=true` 效果<br>2. 增量编译 vs 全量编译<br>3. 不同优化级别（0/1/2/3/s）的二进制体积对比<br>4. `unsafe_code = "deny"` 的编译期开销<br>5. Windows 8MB 栈保留（`/STACK:8388608`）的影响 |
| **交付物** | ⬜ `.docs/performance/build-system-analysis.md` |

### Q2.4 依赖注入分析

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 依赖分析 |
| **具体步骤** | 1. `cargo tree` → 完整依赖树<br>2. 识别 "heavy" 依赖（rusqlite bundled 是最大贡献者）<br>3. 分析各 crate 在二进制中贡献的体积<br>4. 评估"无 async"声明是否严格维持 |
| **交付物** | ⬜ `.docs/performance/dependency-analysis.md` |

---

## Q3：未探模块深度分析

> **目标**：覆盖 v1 浅层触及的模块，提供完整的技术分析。

### Q3.1 args_utils 参数解析工具

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **交付物** | ⬜ `.docs/unexplored/args-utils.md` |

### Q3.2 runner 通用执行器

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **核心文件** | `src/core/runner.rs` |
| **分析重点** | `rtk err` 和 `rtk test` 的通用执行模式 |
| **交付物** | ⬜ `.docs/unexplored/runner-module.md` |

### Q3.3 telemetry_cmd 遥测命令

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **核心文件** | `src/core/telemetry_cmd.rs`, `src/core/telemetry.rs` |
| **交付物** | ⬜ `.docs/unexplored/telemetry-deep-dive.md` |

### Q3.4 hooks 剩余组件

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **核心文件** | `src/hooks/hook_cmd.rs`, `src/hooks/hook_check.rs`, `src/hooks/verify_cmd.rs`, `src/hooks/hook_audit_cmd.rs`, `src/hooks/permissions.rs`, `src/hooks/constants.rs` |
| **交付物** | ⬜ `.docs/unexplored/hooks-remaining.md` |

### Q3.5 analytics 剩余组件

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **核心文件** | `src/analytics/ccusage.rs`, `src/analytics/session_cmd.rs`, `src/analytics/README.md` |
| **交付物** | ⬜ `.docs/unexplored/analytics-remaining.md` |

### Q3.6 cmds 剩余模块

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **核心文件** | `src/cmds/jvm/gradlew_cmd.rs`, `src/cmds/dotnet/binlog.rs`, `src/cmds/dotnet/dotnet_trx.rs`, `src/cmds/cloud/psql_cmd.rs` |
| **分析重点** | binlog（MSBuild 二进制日志）、trx（测试结果 XML）、psql（PostgreSQL）等特殊格式处理模式 |
| **交付物** | ⬜ `.docs/unexplored/cmds-edge-filters.md` |

---

## Q4：纵深专项研究

> **目标**：在分析基础上进行深挖、对比和实验。

### Q4.1 filters 覆盖度定量分析

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 数据统计 |
| **分析重点** | 1. TOML filter 行数/复杂度/规则数统计<br>2. 各生态系统 filter 密度分布<br>3. 行保留率 vs 节省率关系<br>4. 匹配命令的正则复杂度分析<br>5. 测试覆盖率与 filter 成熟度关联 |
| **交付物** | ⬜ `.docs/depth/filter-coverage-statistics.md` |

### Q4.2 v0.37.2 vs v0.42.2 差异追踪

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | git diff 分析 |
| **具体步骤** | 1. `git diff v0.37.2...develop` 文件范围<br>2. 关键差异归类（新功能 / 修复 / 重构 / 安全）<br>3. 显著架构变化分析<br>4. CLI 行为兼容性变化<br>5. 升级注意事项 |
| **交付物** | ⬜ `.docs/depth/version-diff-analysis.md` |

### Q4.3 端到端 Hook 集成测试

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 实际操作 |
| **具体步骤** | 1. 在测试项目中完整安装 Hook<br>2. 验证 `rtk hook claude` JSON 处理<br>3. 测试 `&&` / `|` / `;` 复合命令<br>4. 测试 `RTK_DISABLED=1` 逃生机制<br>5. 测试无匹配命令的透传行为 |
| **交付物** | ⬜ `.docs/depth/end-to-end-hook-test.md` |

### Q4.4 跨平台行为对比

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 对比分析 |
| **分析重点** | 1. `#[cfg(windows)]` / `#[cfg(unix)]` 分布<br>2. SIGPIPE 处理差异<br>3. `libc` 依赖范围（仅 unix）<br>4. 路径格式（`MAIN_SEPARATOR`）<br>5. Shell quoting 差异 |
| **交付物** | ⬜ `.docs/depth/cross-platform-behavior.md` |

### Q4.5 错误处理与恢复模式统计

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 源码检索 + 分类 |
| **分析重点** | 1. `.context()` / `context()` 使用模式<br>2. `unwrap_or_else` fallback 分布<br>3. `exit(code)` 调用场景<br>4. `eprintln!` 警告输出模式<br>5. 传播路径：main.rs 顶层 error handler |
| **交付物** | ⬜ `.docs/depth/error-handling-patterns.md` |

### Q4.6 SQLite 数据库深度探索

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 实际操作 |
| **具体步骤** | 1. 创建测试数据（运行各种命令）<br>2. 直接 SQLite 查询验证<br>3. 分析查询性能（索引、执行计划）<br>4. 测试 90 天轮换行为<br>5. 项目级 GLOB 查询准确性验证 |
| **交付物** | ⬜ `.docs/depth/sqlite-database-deep-dive.md` |

---

## Q5：知识缝合与二次开发准备

> **目标**：整合所有研究成果，完成知识体系闭环，产出可供实际使用的分析报告和洞见。

### Q5.1 技术债务与改进建议

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 综合分析 |
| **分析重点** | 1. main.rs 3275 行的拆分可能<br>2. `lazy_static!` → `OnceLock` / `LazyLock` 迁移<br>3. 测试夹具扩展建议<br>4. TOML filter 覆盖率盲区<br>5. 文档中不一致之处的勘误 |
| **交付物** | ⬜ `.docs/synthesis/technical-debt.md` |

### Q5.2 RTK 内部架构白皮书

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 综合 v1+v2 |
| **交付物** | ⬜ `.docs/synthesis/rtk-internal-white-paper.md` |
| **章节规划** | 1. 整体架构分层<br>2. 核心数据流设计<br>3. 过滤引擎内部机制<br>4. Hook 系统设计哲学<br>5. 安全模型设计原理<br>6. 扩展性与贡献点分析 |

### Q5.3 二次开发实施方案

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 策划 + 评估 |
| **分析重点** | 1. 可开发的方向评估（新 filter / 新 Agent / 新平台）<br>2. 开发环境搭建要点<br>3. 低门槛切入点（TOML filter > Rust filter > 重构）<br>4. PR 策略：upstream 提议 vs fork<br>5. 优先级建议 |
| **交付物** | ⬜ `.docs/synthesis/secondary-dev-plan.md` |

### Q5.4 知识图谱更新

| 项目 | 内容 |
|------|------|
| **状态** | ⬜ |
| **研究方法** | 整理 |
| **任务描述** | 基于 v2 研究更新 `.docs/report/rtk-knowledge-map.md`，补充新模块和新增的深入分析内容 |
| **交付物** | 🔄 `.docs/report/rtk-knowledge-map.md`（更新版） |

---

## 附录

### A. 目录结构

```
.docs/
├── research-plan-v2.md              # 本计划
├── security/                        # Q1 安全审计
├── performance/                     # Q2 性能剖析
├── unexplored/                      # Q0+Q3 未探模块
├── practice/                        # Q0 实操记录
├── depth/                           # Q4 纵深专项
└── synthesis/                       # Q5 知识缝合
```

### B. v1 vs v2 定位对比

| 维度 | v1（已完） | v2（本计划） |
|------|-----------|-------------|
| **目标** | 全面认知 | 纵深突破 |
| **方法** | 阅读 + 分析 | 实操 + 审计 + 剖析 |
| **产品** | 研究文档 | 可交付的 filter + 安全报告 + 开发方案 |
| **代码交互** | 只读 | 编写 filter + 修改代码 |
| **深度** | 中等（覆盖广度） | 深（逐个模块穿透） |
| **输出** | 21 文档 | ~25 文档 + filter 源码 |

### C. 工具链补充

| 工具 | Q 阶段 | 用途 |
|------|--------|------|
| `hyperfine` | Q2 | 性能基准 |
| `cargo audit` | Q1 | 依赖安全审计 |
| `cargo tree` | Q2 | 依赖树分析 |
| `cargo-bloat` | Q2 | 二进制体积分析 |
| `cargo-llvm-lines` | Q2 | 泛型函数膨胀 |
| SQLite CLI | Q4 | DB 深度探索 |
| WSL / Docker | Q4 | 跨平台测试 |
| `strace` / Process Monitor | Q2 | syscall 追踪 |

### D. 从 v2 到实际贡献的路径

```
v2 研究
  │
  ├─ Q0.5 TOML filter → 提交到上游 RTK（PR）
  ├─ Q0.6 Rust filter  → 提交到上游 RTK（PR）
  ├─ Q1.x 安全发现     → 负责任披露
  ├─ Q5.1 技术债务     → 重构 PR
  └─ Q5.3 二次开发     → fork / 自定义分支
```

---

> **执行说明**：Q0 是实操热身，建议优先完成；Q1–Q2 适合安全/性能兴趣者；Q3 填补剩下知识盲区；Q4–Q5 综合产出。可选择子集执行。
