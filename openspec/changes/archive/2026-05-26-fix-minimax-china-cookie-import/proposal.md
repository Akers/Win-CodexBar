## Why

MiniMax provider 在 China Mainland region 下存在两类认证/用量获取问题：

1. Browser cookie import 未显式接收当前 region，无法按 region 选择正确浏览器域名。
2. Chromium/Edge App-Bound Encryption 可能阻止自动 cookie 解密；即使用户手工录入 Cookie，旧的 `/account/amount` billing endpoint 仍依赖难以可靠获取的 `X-Group-Id`，会返回 500（例如 `strconv.ParseInt: parsing "": invalid syntax`）。

最终可稳定工作的路径是：允许 MiniMax 保存 API Key，并优先使用官方/上游验证的 `/v1/token_plan/remains` Bearer API 获取额度。Cookie import 继续保持 region-aware，并在 Chromium ABE 场景给出明确手工 Cookie 提示。

## What Changes

- MiniMax 的 cookie 域名与 region 联动：Global → `platform.minimax.io`，China Mainland → `platform.minimaxi.com`
- `import_browser_cookies` Tauri 命令支持接收 region 参数
- 前端 Import from Browser 调用时传入当前 provider 的 region 配置
- Cookie extraction 覆盖父域 host_key，并在失败时输出安全诊断；Chromium ABE 报错文案更准确
- MiniMax 加入 API Keys 设置列表，用户可在 UI 保存 API Key
- MiniMax refresh 优先使用 `ctx.api_key` 调用 `{api_base}/v1/token_plan/remains`，无需 Cookie 和 group_id
- 解析 `token_plan/remains` 的 used/total/remaining 多字段别名，正确把 `current_interval_usage_count` 视为剩余额度

## Capabilities

### New Capabilities
- `minimax-region-cookie-domain`: MiniMax provider 的 region-aware cookie 域名解析，使 cookie 导入能根据当前 region 选择正确的浏览器域名
- `minimax-token-plan-remains`: MiniMax provider 可通过 API Key 调用 `/v1/token_plan/remains` 获取配额，避免 cookie billing 对 group_id 的依赖

### Modified Capabilities
- `provider-system`: `import_browser_cookies` 命令签名和 cookie 域名解析逻辑需要支持 region 感知

## Impact

- **`rust/src/providers/minimax/mod.rs`** — MiniMaxRegion 新增 cookie/dashboard/billing URL；新增 token_plan/remains 获取与解析逻辑
- **`apps/desktop-tauri/src-tauri/src/commands/browser_import.rs:41-83`** — `import_browser_cookies` 需新增 `region` 参数
- **`apps/desktop-tauri/src/surfaces/settings/`** — 前端 React 组件需在调用 import 时传入当前 region
- **`rust/src/settings/api_keys.rs`** — API Keys 设置列表新增 MiniMax
