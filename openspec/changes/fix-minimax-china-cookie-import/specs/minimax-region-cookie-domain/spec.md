## ADDED Requirements

### Requirement: MiniMax Region Cookie Domain Resolution

MiniMax provider SHALL 根据当前 region 选择对应的浏览器 cookie 搜索域名。

#### Scenario: Global region cookie domain
- **WHEN** region 为 `Global`
- **THEN** cookie 域名返回 `"platform.minimax.io"`

#### Scenario: China Mainland region cookie domain
- **WHEN** region 为 `ChinaMainland`
- **THEN** cookie 域名返回 `"platform.minimaxi.com"`

#### Scenario: Unknown region defaults to Global
- **WHEN** region 为未知值（非 `"global"` 也非 `"china"`）
- **THEN** cookie 域名返回 `"platform.minimax.io"`（Global 默认值）

### Requirement: Cookie Import Region Awareness

`import_browser_cookies` Tauri 命令 SHALL 接受可选的 region 参数，并在 MiniMax provider 下使用正确的 cookie 域名。

#### Scenario: MiniMax China region import
- **WHEN** 调用 `import_browser_cookies("minimax", browser, "china")`
- **THEN** cookie 搜索域名为 `"platform.minimaxi.com"`

#### Scenario: MiniMax Global region import  
- **WHEN** 调用 `import_browser_cookies("minimax", browser, "global")`
- **THEN** cookie 搜索域名为 `"platform.minimax.io"`

#### Scenario: Other providers ignore region
- **WHEN** 调用 `import_browser_cookies("claude", browser, Some("any"))` 等非 MiniMax provider
- **THEN** region 参数被忽略，使用 `ProviderId::cookie_domain()` 返回的域名

#### Scenario: Missing region defaults to static domain
- **WHEN** 调用 `import_browser_cookies("minimax", browser, None)`
- **THEN** cookie 搜索域名为 `"platform.minimaxi.com"`（`ProviderId::cookie_domain()` 默认值）

### Requirement: MiniMax API Key Token Plan Usage

MiniMax provider SHALL support API-key based usage refresh via `/v1/token_plan/remains` without requiring browser cookies or group_id.

#### Scenario: API key takes precedence
- **WHEN** `FetchContext.api_key` is present for MiniMax
- **THEN** provider calls `{api_base}/v1/token_plan/remains` with `Authorization: Bearer <api_key>` before cookie billing

#### Scenario: Token plan remaining fields
- **WHEN** response contains `total` and `remaining` aliases
- **THEN** used percent is computed as `(total - remaining) / total * 100`

#### Scenario: current_interval_usage_count is remaining
- **WHEN** response contains `current_interval_total_count` and `current_interval_usage_count`
- **THEN** `current_interval_usage_count` is treated as remaining quota, not used quota
