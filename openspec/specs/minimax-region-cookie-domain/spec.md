# MiniMax Region Cookie Domain

## Purpose

Define MiniMax region-aware cookie import behavior and API-key based token plan usage retrieval.

## Requirements

### Requirement: MiniMax Region Cookie Domain Resolution

MiniMax provider SHALL select the browser cookie search domain according to the current region.

#### Scenario: Global region cookie domain
- **WHEN** region is `Global`
- **THEN** the cookie domain is `"platform.minimax.io"`

#### Scenario: China Mainland region cookie domain
- **WHEN** region is `ChinaMainland`
- **THEN** the cookie domain is `"platform.minimaxi.com"`

#### Scenario: Unknown region defaults to Global
- **WHEN** region is an unknown value other than `"global"` or `"china"`
- **THEN** the cookie domain is `"platform.minimax.io"` as the Global default

### Requirement: Cookie Import Region Awareness

The `import_browser_cookies` Tauri command SHALL accept an optional region parameter and use the correct cookie domain for MiniMax.

#### Scenario: MiniMax China region import
- **WHEN** `import_browser_cookies("minimax", browser, "china")` is called
- **THEN** the cookie search domain is `"platform.minimaxi.com"`

#### Scenario: MiniMax Global region import
- **WHEN** `import_browser_cookies("minimax", browser, "global")` is called
- **THEN** the cookie search domain is `"platform.minimax.io"`

#### Scenario: Other providers ignore region
- **WHEN** `import_browser_cookies("claude", browser, Some("any"))` or another non-MiniMax provider is called
- **THEN** the region parameter is ignored
- **AND** the provider uses the domain returned by `ProviderId::cookie_domain()`

#### Scenario: Missing region defaults to static domain
- **WHEN** `import_browser_cookies("minimax", browser, None)` is called
- **THEN** the cookie search domain is `"platform.minimaxi.com"` from the existing `ProviderId::cookie_domain()` fallback

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
