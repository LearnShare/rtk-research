# project-filter-trust-mechanism — 项目级 TOML 过滤器与 Trust 机制

> **研究日期**：2026-06-06
> **研究阶段**：P2.5
> **研究方法**：源码分析 + 实验配置

## 概述
分析 RTK 的项目级过滤器系统（`.rtk/filters.toml`）和 Trust 安全机制（SHA-256 校验 + 信任数据库）。

## 核心发现

1. **信任前加载模型**：未信任的项目 filter 直接跳过（不是"加载但警告"）
2. **SHA-256 完整性校验**：信任存储基于文件哈希，内容变更 → 信任失效
3. **CI 覆盖机制**：`RTK_TRUST_PROJECT_FILTERS=1` 允许 CI 场景跳过信任检查
4. **double 验证**：`rtk verify` 运行时 + `build.rs` 构建时都验证 TOML 内联测试
5. **项目 filter 可提交 Git**：`.rtk/filters.toml` 应被版本控制，通过信任机制防止恶意注入

## 详细分析

### 1. 安全模型 — 信任前加载

**设计动机**：`.rtk/filters.toml` 是项目级文件，可被提交到公开仓库。攻击者可通过它控制 LLM 看到的内容：

- `replace` 规则可改写命令输出
- `match_output` 规则可隐藏错误
- `strip_lines_matching` 可隐藏关键信息

**防护策略**：
```
加载流程:
1. 找到 .rtk/filters.toml → 检查信任状态
2. 未信任 → 跳过（NOT 加载 + 警告）
3. 已信任 → 检查 SHA-256 是否匹配
4. 哈希不匹配 → 跳过（通知用户重新评估）
5. 哈希匹配 → 加载应用
```

用户操作流程：
```bash
# 首次使用项目 filter
$ rtk make install
[rtk] WARNING: untrusted project filters (.rtk/filters.toml)
[rtk] Filters NOT applied. Run `rtk trust` to review and enable.

# 信任
$ rtk trust
Trusted project: C:\works\cc-test\... (sha256: abc123...)
```

### 2. Trust 存储

```json
// ~/.local/share/rtk/trusted_filters.json
{
  "version": 1,
  "trusted": {
    "C:\\works\\cc-test": {
      "sha256": "a1b2c3d4e5f6...",
      "trusted_at": "2026-06-06T12:00:00Z"
    }
  }
}
```

**检验逻辑**：
```rust
pub enum TrustStatus {
    Trusted,                              // 哈希匹配 → 允许
    Untrusted,                            // 不在信任列表中
    ContentChanged { expected, actual },  // 哈希不匹配 → 需要重新信任
    EnvOverride,                          // RTK_TRUST_PROJECT_FILTERS=1 覆盖
}
```

### 3. TOML 内联测试框架

每个 filter 可包含 `[[tests.*]]` 定义：

```toml
[filters.make]
match_command = "^make\\b"
strip_lines_matching = ["Nothing to be done"]
on_empty = "make: ok"

[[tests.make]]
name = "empty output"
input = "make: Nothing to be done for 'all'."
expected = "make: ok"

[[tests.make]]
name = "build error"
input = "make: *** [target] Error 1\nmake: *** [all] Error 2"
expected = "make: *** [target] Error 1\nmake: *** [all] Error 2"
```

**双重验证**：

| 验证点 | 时机 | 机制 |
|--------|------|------|
| 构建时 | `cargo build` | `build.rs` 解析 `[[tests.*]]` 并运行验证 |
| 运行时 | `rtk verify` | `verify_cmd.rs` 加载所有 filter 并运行内联测试 |
| CI 模式 | `rtk verify --require-all` | 每个 filter 必须有内联测试 |

### 4. verify_cmd.rs — 运行时验证

```rust
pub fn run(filter: Option<String>, require_all: bool) -> Result<()> {
    // 加载所有 filter（项目级 + 用户级 + 内置）
    // 运行每个 filter 的 [[tests.*]] 定义
    // filter: Option — 只运行指定 filter
    // require_all: bool — 每个 filter 必须有测试
}
```

### 5. 项目级 filter 文件结构

```toml
# .rtk/filters.toml (在项目根目录)
# rtk init 生成模板

schema_version = 1

[filters.my-tool]
match_command = "^my-tool\\b"
# ... filter 定义 ...

[filters.other-tool]
match_command = "^other-tool\\b"
# ... filter 定义 ...

[[tests.my-tool]]
name = "test name"
input = "..."
expected = "..."
```

`rtk init` 会生成带有注释的模板文件。

### 6. 安全考量总结

| 攻击面 | 防护措施 | 严重性 |
|--------|----------|--------|
| 恶意 filter 隐藏安全扫描输出 | Trust 机制 + SHA-256 校验 | 高 |
| Filter 注入修改命令行为 | filter 不改变命令，只过滤输出 | 中 |
| 项目 filter 覆盖内置 filter 静默替换 | 覆盖时发出警告信息 | 低 |
| 信任文件篡改（`trusted_filters.json`） | 不涉及代码执行，影响有限 | 低 |

## 结论与洞见

1. **默认拒绝原则**：未信任的 filter 自动跳过，而非加载并警告
2. **哈希链持续验证**：内容变更自动失效信任，确保每次修改都需重新审核
3. **环境变量覆盖合理**：CI 场景不交互，`RTK_TRUST_PROJECT_FILTERS=1` 是必要逃生口
4. **内联测试提升质量**：`[[tests.*]]` + 双重验证确保 filter 行为正确
5. **`rtk verify --require-all` 适合 CI**：强制每个 filter 都有测试，适合团队协作
