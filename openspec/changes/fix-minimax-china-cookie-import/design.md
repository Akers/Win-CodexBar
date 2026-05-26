## Context

MiniMax provider 已支持 region 路由（`MiniMaxRegion::Global` ↔ `ChinaMainland`），用于 API endpoint 选择。但 browser cookie 导入流程原先从 `ProviderId::cookie_domain()` 获取静态域名，不感知当前 region。

实际调试发现 MiniMax 正确平台域名映射为：Global → `platform.minimax.io`，China Mainland → `platform.minimaxi.com`。此外，Chromium/Edge 的 App-Bound Encryption 可能阻止自动 cookie 解密；手工 Cookie 虽可进入请求链路，但 `/account/amount` billing endpoint 依赖 `X-Group-Id`，无法可靠从 localStorage 获取，可能返回 500。

最终设计保留 region-aware cookie import 作为辅助路径，同时引入 API Key 优先路径：通过 `/v1/token_plan/remains` 获取配额，该接口只需要 Bearer API Key，不依赖 Cookie 或 group_id。

## Goals / Non-Goals

**Goals:**
- 修复 MiniMax China Mainland region 下的 browser cookie 导入失败问题
- China Mainland region 时搜索 `platform.minimaxi.com` 域名，Global region 时搜索 `platform.minimax.io`
- 在 Chromium ABE 阻止自动 cookie 导入时给出准确提示
- 允许用户在 API Keys UI 保存 MiniMax API Key
- 优先使用 `/v1/token_plan/remains` 获取 MiniMax 用量，避免 `/account/amount` 对 group_id 的依赖
- 保持其他 provider 的 cookie 导入行为不变

**Non-Goals:**
- 不改变其他 provider 的 cookie 域名逻辑
- 不实现 Chromium App-Bound Encryption 绕过
- 不依赖 MiniMax localStorage 作为认证来源

## Decisions

### D1: Region-to-cookie-domain mapping in `MiniMaxRegion`

`MiniMaxRegion` 枚举新增 `cookie_domain()` 方法：

| Region | API Base URL | Cookie Domain |
|--------|-------------|---------------|
| `Global` | `api.minimax.io` | `platform.minimax.io` |
| `ChinaMainland` | `api.minimaxi.com` | `platform.minimaxi.com` |

枚举从 `enum` 改为 `pub enum`，使 Tauri commands 层可直接引用。

**Alternative considered**: 在 `import_browser_cookies` 中硬编码 miniMax 区域域名映射 → 放弃。MiniMaxRegion 本身已有区域概念，将域名作为 region 属性更内聚。

### D2: `import_browser_cookies` 增加 region 参数

命令签名新增 `region: Option<String>`：

```rust
pub fn import_browser_cookies(
    provider_id: String,
    browser_type: String,
    region: Option<String>,
) -> Result<Vec<CookieInfoBridge>, String>
```

对于 `provider_id == "minimax"`，使用 `MiniMaxRegion::from(region).cookie_domain()` 选择域名。

对于其他 provider 或无 region 参数，走原有 `pid.cookie_domain()` 路径。

### D3: 不修改 `ProviderId::cookie_domain()`

`ProviderId::MiniMax => Some("platform.minimaxi.com")` 保持 China/fallback 兼容。Region 感知逻辑在 `import_browser_cookies` 命令层处理。

### D4: 前端传参

前端 MiniMax 配置区域已有的 region state（`"global"` / `"china"`），在调用 `invoke("import_browser_cookies", { providerId, browserType, region })` 时传入。无额外 UI 变更。

## Risks / Trade-offs

- **[Low] Tauri 命令签名变更** → 需确保前端调用方同步传入 region 参数（否则 `Option<String>` 默认 `undefined` → `None`，行为不变）
- **[Medium] Chromium ABE** → 无法稳定自动解密 Edge/Chrome/Brave cookies；需要明确提示用户改用 Firefox 或手工 Cookie/API Key
- **[Low] API Key UI 暴露** → MiniMax 现在可通过 Settings → API Keys 保存 key，走既有 Secret 存储/读取路径

### D5: API Key 优先的 token_plan/remains 路径

MiniMax refresh 在 `FetchContext.api_key` 存在时优先调用：

```text
{api_base}/v1/token_plan/remains
Authorization: Bearer <api_key>
Content-Type: application/json
```

该接口不需要 `group_id`，避免 `/account/amount` 在 `X-Group-Id` 为空时返回 500。解析逻辑支持 `used`/`total`/`remaining` 及多个别名，并将 MiniMax 特有的 `current_interval_usage_count` 视作“剩余额度”而不是“已用额度”。
