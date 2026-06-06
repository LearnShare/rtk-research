# secondary-dev-plan — RTK 二次开发实施方案

> **研究日期**：2026-06-06
> **研究阶段**：Q5.3
> **研究方法**：综合 v1+v2 研究

## 概述
基于全部研究成果，制定 RTK 二次开发的可行方向和实施路径。

## 可开发方向

### 1. 新 TOML filter（低门槛，推荐入门）

**候选列表**（从本地系统发现的未覆盖命令）：
- `ipconfig` ✅ 已实现
- `systeminfo` — Windows 系统信息
- `tasklist` — Windows 进程列表（可连接成树状）
- `netstat` ✅ 已实现（Rust filter）
- `route print` — 路由表
- `wmic` — WMI 查询
- `certutil` — 证书工具

**门槛**：仅需创建 `.toml` 文件 + 内联测试

### 2. 新 Rust filter（中级）

**候选**：
- `ping`（已有 TOML filter，可升级为 Rust-native 带统计）
- `tracert` / `traceroute` — 路由追踪
- `nslookup` / `dig` — DNS 查询

**门槛**：需 Rust 基础 + 修改 main.rs 4 处

### 3. 新 Agent Hook 支持

**候选**：
- Mistral Vibe（上游已计划 #800）

### 4. 重构/优化（高级）

**候选**：
- main.rs 拆分
- `lazy_static!` → `LazyLock` 迁移
- 测试夹具系统化

## 贡献路径

### 上游贡献（PR 到 rtk-ai/rtk）

适合提交给上游的贡献：
1. ✅ ipconfig TOML filter（已实现）
2. 新 TOML filter（简单）
3. Bug 修复
4. 文档改进

### Fork 定制

适合 fork 开发的：
1. 企业/团队专用的自定义 filter 集
2. 集成内部工具的 Hook
3. 非公开的安全策略

## 开发环境

```
工作目录: C:\works\cc-test\rtk\
Rust 版本: 1.96.0
构建: cargo build (debug ~60s, release ~3min)
测试: cargo test --all (2065 tests, ~30s)
质量门: cargo fmt + cargo clippy + cargo test
```
