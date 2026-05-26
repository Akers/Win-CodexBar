## 1. FetchContext 扩展

- [x] 1.1 在 `rust/src/core/provider.rs` 的 `FetchContext` 结构体中添加 `api_region: Option<String>` 字段 — 对应 spec: FetchContext Region Propagation
- [x] 1.2 更新所有 `FetchContext` 构造点，从 settings 读取 `api_region` 并注入（`rust/src/` 中搜索 `FetchContext` 的构造位置） — 对应 spec: FetchContext Region Propagation, api_region defaults to global

## 2. ZaiRegion 枚举与 URL 路由

- [x] 2.1 在 `rust/src/providers/zai/mod.rs` 中定义 `ZaiRegion` 枚举（`Global`, `ChinaMainland`）并实现 `From<&str>` — ProviderId: Zai, 对应 spec: Zai Region Enum
- [x] 2.2 实现 `api_url(region: ZaiRegion) -> &'static str` 方法，global 返回 `https://api.z.ai/api/monitor/usage/quota/limit`，china 返回 `https://bigmodel.cn/api/monitor/usage/quota/limit` — ProviderId: Zai, 对应 spec: Region-based API URL Routing
- [x] 2.3 实现 `dashboard_url(region: ZaiRegion) -> &'static str` 方法，global 返回 `https://z.ai/dashboard`，china 返回 `https://bigmodel.cn/` — ProviderId: Zai, 对应 spec: Region-based Dashboard URL
- [x] 2.4 实现 `credential_target(region: ZaiRegion) -> &'static str` 方法，global 返回 `codexbar-zai`，china 返回 `codexbar-zai-china` — ProviderId: Zai, 对应 spec: Region-based Credential Storage

## 3. ZaiProvider fetch 逻辑适配

- [x] 3.1 修改 `fetch_usage_api` 方法：从 `ctx.api_region` 解析 `ZaiRegion`，使用 `api_url(region)` 替代硬编码 `ZAI_API_URL` — ProviderId: Zai, 对应 spec: Region-based API URL Routing
- [x] 3.2 修改 `get_api_token` 方法签名，接受 region 参数，使用 `credential_target(region)` 替代硬编码 `ZAI_CREDENTIAL_TARGET` — ProviderId: Zai, 对应 spec: Region-based Credential Storage
- [x] 3.3 删除不再需要的 `ZAI_API_URL` 和 `ZAI_CREDENTIAL_TARGET` 常量 — ProviderId: Zai

## 4. Tauri Commands 层适配

- [x] 4.1 修改 `apps/desktop-tauri/src-tauri/src/commands/provider_settings.rs` 中获取 dashboard URL 的逻辑，对 Zai provider 根据 region 返回对应的 dashboard URL — ProviderId: Zai, 对应 spec: Region-based Dashboard URL
- [x] 4.2 验证 `FetchContext` 构造点正确传递 `api_region`（检查 `apps/desktop-tauri/src-tauri/` 和 `rust/src/` 中的 fetch pipeline） — 对应 spec: FetchContext Region Propagation

## 5. 测试与验证

- [x] 5.1 为 `ZaiRegion` 枚举和 `From<&str>` 实现编写单元测试 — 对应 spec: Zai Region Enum
- [x] 5.2 为 `api_url`, `dashboard_url`, `credential_target` 方法编写单元测试 — 对应 spec: Region-based API URL Routing, Dashboard URL, Credential Storage
- [x] 5.3 运行 `cargo test --manifest-path rust/Cargo.toml` 和 `cargo test --manifest-path apps/desktop-tauri/src-tauri/Cargo.toml` 确保所有测试通过
- [x] 5.4 运行 `cargo fmt --all` 和 `cargo clippy --all-targets -- -D warnings` 确保代码质量
