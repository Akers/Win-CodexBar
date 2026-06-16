## Context

Z.ai provider（`rust/src/providers/zai/mod.rs`）当前使用硬编码的 `https://api.z.ai` 作为 API 端点，`https://z.ai/dashboard` 作为 Dashboard URL。Region 选择 UI 已实现（`RegionSection.tsx` + `provider_settings.rs`），配置存储已支持 `api_region` 字段，但后端未根据 region 切换 URL。

参考实现：MiniMax provider（`rust/src/providers/minimax/mod.rs:156-161`）已实现了 region-based URL 路由，使用 `MiniMaxRegion` 枚举 + `api_base_url()` 方法模式。

当前 Z.ai 架构：
- `ZaiProvider` 是无状态结构体（仅持有 `ProviderMetadata`）
- `fetch_usage_api()` 使用 `ZAI_API_URL` 常量发送请求
- `ProviderMetadata.dashboard_url` 是 `Option<&'static str>`，在构造时固定
- 凭证使用 `keyring::Entry::new("codexbar-zai", "api_token")` 存取

## Goals / Non-Goals

**Goals:**
- 当 `api_region` 为 `"china"` 时，API 请求发送到 `https://bigmodel.cn` 域名
- 当 `api_region` 为 `"china"` 时，Dashboard URL 指向 `bigmodel.cn`
- 当 `api_region` 为 `"china"` 时，使用独立的凭证存储（避免 global/china token 混用）
- 保持与 MiniMax region 路由相同的模式一致性
- `FetchContext` 需传递 `api_region` 到 provider（当前缺失）

**Non-Goals:**
- 不改变 quota response 的数据结构和解析逻辑（两个 region 的 API 响应格式一致）
- 不实现 region 自动检测或 fallback（用户手动选择即可）
- 不修改 `Provider` trait 签名
- 不修改前端 UI（Region 选择已实现）

## Decisions

### D1: 通过 `FetchContext` 传递 region

**选择**：在 `FetchContext` 中新增 `api_region: Option<String>` 字段，由 fetch pipeline 从 settings 读取并注入。

**替代方案**：
- A) 在 `ZaiProvider::new()` 时传入 region — 不可行，provider 在启动时实例化一次，region 可以运行时切换
- B) 在 `ZaiProvider` 中直接读取 settings 文件 — 破坏了依赖注入，不利于测试

**理由**：`FetchContext` 已经是传递 fetch 配置的标准渠道（`api_key`, `source_mode` 等都通过它传递）。新增 `api_region` 符合现有模式。

### D2: 新增 `ZaiRegion` 枚举

**选择**：在 `zai/mod.rs` 中定义 `enum ZaiRegion { Global, ChinaMainland }` 并实现 `From<&str>` 转换。

**理由**：与 MiniMax 的 `MiniMaxRegion` 模式一致。使用枚举而非字符串比较，编译时更安全。

### D3: URL 常量定义

**选择**：使用匹配分支返回静态字符串，而非 `const`：
```rust
fn api_url(region: ZaiRegion) -> &'static str {
    match region {
        ZaiRegion::Global => "https://api.z.ai/api/monitor/usage/quota/limit",
        ZaiRegion::ChinaMainland => "https://bigmodel.cn/api/monitor/usage/quota/limit",
    }
}
```

**理由**：与 MiniMax 模式一致。避免分散的常量定义。

### D4: Dashboard URL 动态化

**选择**：`ProviderMetadata` 中的 `dashboard_url` 保持为 global 默认值。在 `metadata()` 方法中改为动态计算或新增 `dashboard_url_for_region(region)` 方法，由调用方（Tauri commands）根据 region 传给前端。

**替代方案**：修改 `ProviderMetadata` 为非 `'static` — 影响面太大。

**理由**：`ProviderMetadata` 是静态的，dashboard URL 由 Tauri commands 层根据 region 单独处理，避免改动 trait 或 metadata 结构。

### D5: 凭证存储分离

**选择**：使用不同的 credential target 区分 region：
- Global: `codexbar-zai`
- ChinaMainland: `codexbar-zai-china`

**理由**：两个 region 的 token 不互通（z.ai token ≠ bigmodel.cn token），必须分开存储。

## Risks / Trade-offs

- **[BigModel API 路径未确认]** → 需要验证 `bigmodel.cn` 的 API 路径是否与 `api.z.ai` 完全一致。如果不同，需要额外的路径常量。
- **[Dashboard URL 传递]** → 当前 `ProviderMetadata.dashboard_url` 是静态的，需要确保 Tauri commands 层正确将 region-aware dashboard URL 传给前端。
- **[已有用户凭证迁移]** → 已存储 `codexbar-zai` token 的中国区用户，切换 region 后需要重新输入 token。这是预期行为，但需要在 UI 上有清晰提示。
