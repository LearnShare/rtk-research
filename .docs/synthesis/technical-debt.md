# technical-debt — RTK 技术债务与改进建议

> **研究日期**：2026-06-06
> **研究阶段**：Q5.1
> **研究方法**：综合分析

## 概述
汇总研究过程中识别的技术债务和可改进方向。

## 核心发现

### 🔴 高优先级

1. **main.rs 3275 行过大**：应拆分 Commands 枚举和 run_cli() match 到独立模块
2. **`lazy_static!` → `LazyLock` 迁移**：Rust 1.80+ 已稳定 `std::sync::LazyLock`，可减少依赖
3. **`#[allow(dead_code)]` 清理**：~10 处未使用代码，可能隐藏未完成的特性

### 🟡 中优先级

4. **TOML filter `match_output` 激进度**：部分短路规则可能吞掉有效错误
5. **测试夹具覆盖不足**：只有 gradlew/glab/golangci/dotnet 有夹具，git/cargo/js 等核心 filter 使用内联数据
6. **窗口相关测试缺失**：没有对 powershell、path、进程创建的专项测试
7. **错误消息不一致**：部分使用 `[rtk: ...]` 格式，部分使用 `rtk: ...`
8. **文档与代码同步**：部分 README 中的 CLI 示例与源码不同步

### 🟢 低优先级

9. **`stream.rs` 采用度低**：BlockHandler/LineHandler 抽象只在 git 和 cargo 中使用
10. **`parser` 模块采用度低**：TokenFormatter + OutputParser 只在 vitest 中使用
11. **`ccusage` 外部依赖**：依赖 `npx ccusage`，失败自动降级，但不够优雅
12. **Config 默认值硬编码**：多个配置项的默认值散落在各处

## 建议

| 建议 | 工作量 | 影响 |
|------|--------|------|
| 拆分 main.rs | 2-3 天 | 显著改善可维护性 |
| LazyLock 迁移 | 1 天 | 减少依赖，现代化 |
| 夹具扩展 | 1 天 | 提高测试质量 |
| stream/parser 推广 | 2-3 天 | 代码统一 |
| error message 统一 | 0.5 天 | 一致性 |
