# supply-chain-integrity — TOML 沙箱、完整性校验与供应链安全

> **研究日期**：2026-06-06
> **研究阶段**：Q1.2–Q1.4
> **研究方法**：源码审计 + 依赖分析

## 概述
评估 RTK 的 TOML filter 沙箱安全性、SHA-256 完整性校验体系和供应链依赖风险。

## 核心发现

1. **TOML filter 沙箱天然受限**：声明式配置 + 纯文本处理 + 无 I/O = 攻击面极小
2. **SHA-256 完整性链完整**：安装时存储 → 运行时验证 → 篡改检测 → 告警
3. **rusqlite bundled 是最大依赖**：~842KB，引入 SQLite 全部攻击面
4. **`unsafe_code = "deny"` 严格执行**：仅 2 处 `#[allow(unsafe_code)]`（SIGPIPE 信号 + proxy 信号处理）
5. **依赖树深度可控**：27 个直接依赖，无 async 运行时依赖

## 详细分析

### 1. TOML Filter 沙箱评估

**TOML DSL 的天然限制**（不可执行任意代码）：
- 纯声明式：仅正则匹配 + 行过滤 + 截断
- 无 I/O：不能读写文件或执行命令
- 无变量：没有状态保持能力
- 无循环/分支：线性管道执行

**潜在的攻击向量**：

| 向量 | 可能性 | 说明 | 缓解措施 |
|------|--------|------|----------|
| ReDoS（正则拒绝服务） | 🟡 中 | 恶意正则导致 CPU 100% | `regex` crate 有内置超时保护 |
| `match_output` 隐藏错误 | 🟡 中 | 短路规则可吞错误消息 | `unless` 条件可设置例外 |
| `replace` 改写输出 | 🟡 中 | 正则替换可篡改输出内容 | 输出缓存在 tee 目录中 |
| 项目 filter 注入 | 🟡 低 | `.rtk/filters.toml` 可被提交 | Trust 机制 + SHA-256 校验 |
| 正则信息泄露 | 🟢 低 | 错误消息中暴露路径 | `strip_ansi` 预处理 |

**ReDoS 风险评估**：
```rust
// 高危模式（应避免在 TOML filter 中使用）：
// (a+)+b  → 指数级回溯
// (\s*)+$  → 嵌套量词

// RTK 中常见的安全模式：
// "^\\s*$"       → 简单前缀锚定 — 安全
// "^\\[.*\\]"    → 固定边界 — 安全
// "^error\\b"    → 带词边界 — 安全
```

### 2. SHA-256 完整性校验体系

**完整链路**：

```
rtk init (安装)
  → 写入 rtk-rewrite.sh
  → compute_hash() → SHA-256(文件) → .rtk-hook.sha256
  → 文件设为 0444（只读）
  → settings.json 注册 "rtk hook claude"

rtk hook claude (每次执行前)
  → integrity::runtime_check()
  → 比较当前 SHA-256 vs 存储的 Hash
  → Verified → 继续
  → Tampered → 告警 "Hook modified outside rtk init"

rtk verify (手动触发)
  → integrity::run_verify()
  → 检查所有 Agent 的 Hook 完整性
  → 报告 Verified / Tampered / NoBaseline / NotInstalled / OrphanedHash
```

**状态机**：

```rust
pub enum IntegrityStatus {
    Verified,          // 哈希匹配
    Tampered { ... },  // 哈希不匹配（被篡改）
    NoBaseline,        // 有 Hook 但无存储 Hash（旧版本升级）
    NotInstalled,      // Hook 未安装
    OrphanedHash,      // Hash 存在但 Hook 被删除
}
```

**安全假设**：
- Hash 文件本身不防篡改（攻击者可以同时修改 Hook 和 Hash）
- 但提供了"检测"而非"预防"
- 只读属性 (0444) 是防止意外覆盖的减速带

### 3. 依赖供应链分析

**直接依赖（27 个）**：

| 依赖 | 用途 | 安全关注 |
|------|------|----------|
| `clap 4` | CLI 参数解析 | 🟢 广泛使用，维护活跃 |
| `anyhow` | 错误处理 | 🟢 零安全相关 |
| `rusqlite (bundled)` | SQLite 嵌入式数据库 | 🟡 最大依赖，有 CVE 历史 |
| `sha2` | SHA-256 哈希 | 🔴 密码学关键依赖 |
| `regex` | 正则引擎 | 🟡 有 ReDoS 风险（内置保护） |
| `serde / serde_json` | 序列化 | 🟢 广泛使用 |
| `ureq` | HTTP 客户端 | 🟡 用于遥测 |
| `toml` | TOML 解析 | 🟢 格式简单 |
| `which` | 可执行文件查找 | 🟡 路径遍历风险 |
| `flate2` | 压缩 | 🟢 仅用于特定场景 |
| `colored` | 终端颜色 | 🟢 纯格式化 |
| `chrono` | 时间处理 | 🟡 有旧版 CVE |

**构建依赖**：仅 `toml`（用于 build.rs 验证 TOML 语法）。

**供应链风险总结**：

| 风险 | 级别 | 措施 |
|------|------|------|
| 依赖版本过旧 | 🟢 低 | Cargo.lock 锁定，定期更新 |
| 恶意 crate | 🟢 低 | 27 个依赖均为知名 crate |
| 密码学算法 | 🔴 需注意 | SHA-256 由 `sha2` crate 提供，`unsafe_code = "deny"` |
| rusqlite bundled | 🟡 中 | compiled-in SQLite，需要时关注安全更新 |
| 构建时供应链 | 🟡 中 | 使用 `cargo install --git` 而非 crates.io |

### 4. 危险代码审计

**unsafe 代码**（仅 2 处）：

```rust
// 位置 1: src/main.rs (unix SIGPIPE)
#[cfg(unix)]
#[allow(unsafe_code)]
unsafe {
    libc::signal(libc::SIGPIPE, libc::SIG_DFL);
}

// 位置 2: src/main.rs (proxy 信号处理)
#[cfg(unix)]
#[allow(unsafe_code)]
unsafe {
    libc::signal(libc::SIGINT, ...);
    libc::signal(libc::SIGTERM, ...);
}
```

两者仅在 Unix 平台编译，且信号处理是标准的 Rust 模式。

**`#[allow(dead_code)]` 使用**：~10 处，用于：
- 序列化结构体中未使用的字段（serde 反序列化需要）
- 测试辅助函数
- 特性门控的代码路径

### 5. 安全加固建议

| 建议 | 优先级 | 说明 |
|------|--------|------|
| 增加 `cargo audit` 到 CI | 🔴 高 | 定期扫描依赖 CVE |
| TOML filter 中限制 `match_output` 使用 | 🟡 中 | 避免过度激进的短路隐藏错误 |
| 增加 ReDoS 检测测试 | 🟡 中 | 对内置 TOML filter 的正则做超时测试 |
| 考虑 `OnceLock` 替代部分 `lazy_static!` | 🟢 低 | 非安全原因，是代码现代化 |
| Hash 文件防护加强 | 🟢 低 | 探索系统 ACL 保护 hash 文件 |

## 结论与洞见

1. **TOML filter 沙箱天生安全**：声明式 DSL + 无 I/O 限制了攻击面
2. **完整性链有检测能力**：不能防篡改，但能检测和告警
3. **依赖风险可控**：27 个 crate，无异步运行时，知名维护者
4. **unsafe 代码极少**：仅 2 处 Unix 信号处理，其余全 safe Rust
5. **`deny` 系列 lint 有效**：`unsafe_code = "deny"` + `warnings = "deny"` 强约束
