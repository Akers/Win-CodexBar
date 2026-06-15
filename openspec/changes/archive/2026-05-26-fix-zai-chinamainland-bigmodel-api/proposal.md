## Why

Z.ai provider 在用户选择 Region 为 "China Mainland (BigModel)" 时，仍使用国际版 `api.z.ai` 域名和 `z.ai` OAuth/Dashboard URL，导致中国区用户无法正确获取 usage 数据。需要在 region 为 `china` 时，将 API 端点和 OAuth/Dashboard URL 切换到 `bigmodel.cn` 域名。

## What Changes

- 在 `ZaiProvider` 中新增 region 枚举（`ZaiRegion::Global` / `ZaiRegion::ChinaMainland`）和基于 region 的 URL 路由逻辑
- `fetch_usage_api` 方法根据当前 region 选择对应的 API 基础 URL（`api.z.ai` vs `bigmodel.cn`）
- `ProviderMetadata` 中的 `dashboard_url` 根据 region 动态返回（`z.ai/dashboard` vs `bigmodel.cn`）
- 凭证目标（credential target）根据 region 区分，避免 global/china token 混用
- 前端 LoginSection 中 OAuth 登录链接根据 region 动态切换

## Capabilities

### New Capabilities

- `zai-region-routing`: Z.ai provider 根据 region 配置（global/china）动态选择 API 端点、OAuth URL 和 Dashboard URL

### Modified Capabilities

- `provider-system`: Provider trait 需支持 region-aware URL 路由，影响 `fetch_usage` 和 `metadata` 的行为

## Impact

- `rust/src/providers/zai/mod.rs` — 核心变更：新增 `ZaiRegion` 枚举和 URL 路由
- `rust/src/providers/zai/mcp_details.rs` — MCP 详情菜单可能需适配不同 region URL
- `rust/src/settings.rs` / `rust/src/settings/types.rs` — 已有 `api_region` 支持，无需改动
- `apps/desktop-tauri/src-tauri/src/commands/provider_settings.rs` — 已有 region options，无需改动
- `apps/desktop-tauri/src/surfaces/settings/providers/sections/RegionSection.tsx` — 已有 UI，无需改动
- 参考：`rust/src/providers/minimax/mod.rs` — MiniMax 已实现类似 region 路由模式

## Non-goals

- 不改变 Z.ai 的数据解析逻辑（quota response 格式两个 region 一致）
- 不改变前端 Region 选择 UI
- 不添加 region 自动检测或自动回退逻辑（MiniMax 的 fallback 模式不适用于 Z.ai）
- 不修改 `FetchContext` 或 `Provider` trait 签名
