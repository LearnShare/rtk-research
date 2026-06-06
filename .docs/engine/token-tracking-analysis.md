# token-tracking-analysis — RTK Token 追踪与计费分析

> **研究日期**：2026-06-06
> **研究阶段**：P2.3
> **研究方法**：源码分析 + SQLite Schema 审计

## 概述
分析 RTK 的 Token 追踪系统：SQLite 数据库 Schema、`TimedExecution` RAII 计时器、聚合查询引擎、项目级 GLOB 匹配和 90 天自动清理策略。

## 核心发现

1. **SQLite bundled 存储**：嵌入式 SQLite（rusqlite crate），无外部数据库依赖
2. **RAII 计时器模式**：`TimedExecution::start()` → 自动记录 elapsed time → `.track()` 写入
3. **双路径记录**：过滤命令走 `.track()`（含节省率），透传命令走 `.track_passthrough()`（0% 节省）
4. **项目级 GLOB 匹配**：用 `GLOB` 而非 `LIKE` 避免路径中 `_` 和 `%` 的通配符问题
5. **90 天自动轮换**：每次插入时清理 `WHERE timestamp < datetime('now', '-90 days')`
6. **增益分析支持 JSON/CSV 导出**：`rtk gain --all --format json` / csv

## 详细分析

### 1. 数据库 Schema

SQLite 数据库位于 `~/.local/share/rtk/tracking.db`，使用 rusqlite (bundled)：

```sql
CREATE TABLE commands (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,

    -- 命令标识
    command         TEXT NOT NULL,    -- 原始命令 (如 "git status")
    rewritten_cmd   TEXT,             -- RTK 重写后的命令 (如 "rtk git status")

    -- Token 指标
    input_tokens    INTEGER NOT NULL,  -- 原始输出 token 数
    output_tokens   INTEGER NOT NULL,  -- 过滤后 token 数
    saved_tokens    INTEGER NOT NULL,  -- 节省 token = input - output
    savings_pct     REAL NOT NULL,     -- 节省百分比

    -- 执行性能
    execution_time_ms INTEGER NOT NULL, -- 执行耗时（毫秒）

    -- 项目范围（v0.42+）
    project_path    TEXT,              -- 项目路径，用于项目级统计

    -- 时间
    timestamp       TEXT NOT NULL      -- ISO 8601 UTC 时间戳
);
```

### 2. TimedExecution RAII 计时器

```rust
pub struct TimedExecution {
    start: Instant,
    project_path: String,
}

impl TimedExecution {
    pub fn start() -> Self {
        Self {
            start: Instant::now(),
            project_path: current_project_path_string(), // 捕获 CWD
        }
    }

    // 过滤命令：记录 input/output/savings
    pub fn track(self, raw: &str, rtk: &str, input: &str, output: &str) {
        let elapsed = self.start.elapsed();
        let input_tokens = count_tokens(input);
        let output_tokens = count_tokens(output);
        let saved = input_tokens.saturating_sub(output_tokens);
        let pct = if input_tokens > 0 {
            saved as f64 / input_tokens as f64 * 100.0
        } else {
            0.0
        };
        Tracker::new()?.record(raw, rtk, input_tokens, output_tokens, elapsed, self.project_path)?;
    }

    // 透传命令：不记录节省（saved = 0, pct = 0）
    pub fn track_passthrough(self, raw: &str, rtk: &str) {
        // 只在数据库中写入一行空记录
    }
}
```

`current_project_path_string()` 获取 CWD 的 `canonicalize()` 路径，用于项目级过滤。

### 3. 双路径追踪体系

```
执行路径 → TimedExecution::start()
    │
    ├── Filter 路径（有过滤）
    │   ├── execute_command() 执行并捕获输出
    │   ├── filter_*() / apply_filter() 过滤
    │   ├── print!() 输出过滤结果
    │   └── timer.track(raw_cmd, rtk_cmd, &raw_output, &filtered_output)
    │       ├── count_tokens(raw) → input_tokens
    │       ├── count_tokens(filtered) → output_tokens
    │       └── INSERT INTO commands(...)
    │
    └── Passthrough 路径（无过滤）
        ├── Stdio::inherit() 流式输出
        └── timer.track_passthrough(raw_cmd, rtk_cmd)
            └── INSERT INTO commands(...)  # saved_tokens=0, savings_pct=0
```

### 4. Token 统计算法

```rust
fn count_tokens(text: &str) -> usize {
    text.split_whitespace().count()  // 按空白分割计数
}
```

这是一个近似算法。RTK 将 token 近似为空白分隔的单词数，理由是：
- CLI 输出的结构化文本中，单词数 ≈ token 数偏离不大
- 性能更好（无需调用 tokenizer API）
- LLM 定价通常按近似 token 计费

### 5. Gain 聚合查询引擎（gain.rs）

`rtk gain` 命令提供多维度分析：

| 模式 | 查询 | 输出格式 |
|------|------|----------|
| `rtk gain`（默认） | `SELECT SUM/savings/avg GROUP BY command ORDER BY saved DESC LIMIT 10` | 终端表格 |
| `--daily` | `SELECT strftime('%Y-%m-%d', timestamp), SUM(...) GROUP BY date` | 表格 |
| `--weekly` | SQLite `strftime('%Y-%W', ...)` 按周聚合 | 表格 |
| `--monthly` | SQLite `strftime('%Y-%m', ...)` 按月聚合 | 表格 |
| `--all` | 同时输出 daily + weekly + monthly | 组合 |
| `--format json` | 全部转为 JSON 输出 | JSON |
| `--format csv` | 全部转为 CSV 输出 | CSV |
| `--graph` | 30 天 ASCII 折线图 | ASCII 图 |
| `--history` | 最近 10 条命令 | 表格 |
| `--quota` | 月度配额分析（Pro/5x/20x） | 表格 + 百分比 |
| `--project` | 加入 `WHERE project_path GLOB '...'` 过滤 | 范围受限 |
| `--failures` | 显示解析失败日志 | 表格 |

### 6. 项目级追踪（GLOB 匹配）

```rust
fn project_filter_params(project_path: Option<&str>) -> (Option<String>, Option<String>) {
    match project_path {
        Some(p) => (
            Some(p.to_string()),                    // 精确匹配
            Some(format!("{}*", p, MAIN_SEPARATOR)), // GLOB 前缀匹配
        ),
        None => (None, None),
    }
}

// SQL 中使用 GLOB 而非 LIKE:
//   WHERE (command = ?1 OR project_path GLOB ?2)
```

选择 GLOB 而不是 LIKE 的原因：
- 路径中的 `_` 在 LIKE 中匹配任意字符 → 误匹配
- 路径中的 `%` 在 LIKE 中匹配任意序列 → 需要转义
- GLOB 使用 Unix 风格的通配符（`*`, `?`），路径中很少包含

### 7. 90 天自动清理

```rust
fn cleanup_old_records(conn: &Connection) -> Result<()> {
    conn.execute(
        "DELETE FROM commands WHERE timestamp < datetime('now', '-90 days')",
        [],
    )?;
    Ok(())
}
```

在 `Tracker::new()` 和每次 `record()` 调用时触发。

### 8. Claude Code 经济学（cc_economics.rs）

`rtk cc-economics` 结合 RTK 追踪数据和 Claude Code 使用数据（ccusage.json）：

```
Spending (CC Usage)               Savings (RTK)
────────────────────              ────────────────
Total tokens consumed              Total tokens saved
Cost per month                     Cost saved per month
Input vs output ratio              Savings efficiency %
```

需要 `ccusage.json` 文件，位置由 Claude Code 自动管理。

## 结论与洞见

1. **简单但实用**：`split_whitespace` 近似 token 计数不够精确但足够决策
2. **RAII 计时器设计好**：start → track 模式避免遗漏记录
3. **项目级 GLOB 匹配有细节**：优先 GLOB 而非 LIKE 是经验教训
4. **双路径追踪完整**：过滤 + 透传都记录，无盲区
5. **可导出是加分项**：JSON/CSV 导出便于集成外部分析工具
