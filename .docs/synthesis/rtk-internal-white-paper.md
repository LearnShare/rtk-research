# rtk-internal-white-paper — RTK 内部架构白皮书

> **研究日期**：2026-06-06
> **研究阶段**：Q5.2
> **研究方法**：综合 v1 (21 交付物) + v2 (20+ 交付物)

## 一、架构总览

RTK 是一个 CLI-LLM 代理，位于 shell 命令和 AI Agent 之间，核心职责是 **压缩 CLI 输出以最小化 Token 消耗**。

```
LLM Agent → Hook → rtk rewrite → CLI → output → rtk filter → compressed output → LLM
                      ↓                            ↑
              rtk rewrite (exit:0/1/2/3)     token tracking (SQLite)
```

## 二、7 个关键架构决策

1. **单线程无 async**：启动 <10ms，无 tokio/async-std 依赖
2. **双轨过滤**：Rust-native（复杂结构）+ TOML DSL（声明式行过滤）
3. **三层回退**：Full（完全过滤）→ Degraded（部分+警告）→ Passthrough（原始截断）
4. **四层安全**：白名单 + SHA-256 + 权限决策 + 信任管理
5. **RAII 追踪**：`TimedExecution::start() → .track()` 确保无遗漏
6. **嵌入式 TOML**：`build.rs` 编译时合并 60+ filter → `include_str!`
7. **单一真相源**：`discover/registry.rs` 的 `rewrite_command()` 是全体重写的唯一入口

## 三、模块职责速览

```
main.rs (3275行) → Clap → Commands enum → run_cli() → fallback
  ├─ cmds/ (9生态, 30+ filter) → 各生态系统过滤逻辑
  ├─ core/ (14组件) → 共享基础设施 (config/tracking/filter/toml_filter/tee/...)
  ├─ hooks/ (7组件) → Agent 集成 (init/rewrite/integrity/trust/verify)
  ├─ analytics/ (4组件) → Token 统计 (gain/cc_economics/session)
  ├─ discover/ (5组件) → 重写规则 + 历史发现 (registry/rules/lexer/provider)
  ├─ learn/ (2组件) → CLI 纠正检测 (detector/report)
  ├─ parser/ (2组件) → 结构化解析 (types/formatter)
  └─ filters/ (60+ TOML) → 声明式行过滤规则
```

## 四、数据流

```
命令输入 → main.rs 解析 → match Commands 枚举 → 专用 filter
                                                      ↓
                              执行子命令 → 捕获 stdout/stderr
                                                      ↓
                          Rust filter_fn() / TOML apply_filter()
                                                      ↓
                        TimedExecution.track() → SQLite
                                                      ↓
                                      print!() 压缩输出
```

## 五、安全模型

```
is_operational_command() 白名单
  → integrity::runtime_check() SHA-256
    → permissions::check_command() Deny/Ask/Allow
      → trust::is_trusted() 项目 TOML filter
        → lexer::contains_unattestable_construct() $()/``/>\n
```

## 六、扩展点

| 扩展点 | 门槛 | 说明 |
|--------|------|------|
| TOML filter | 🔵 极低 | 创建 .toml 文件 + 内联测试 |
| Rust filter | 🟡 中 | 创建 _cmd.rs + main.rs 注册 |
| 新 Agent | 🟡 中 | 实现 hook JSON 处理器 |
| 新模块 | 🔴 高 | 理解 core 架构 + 注册路由 |
