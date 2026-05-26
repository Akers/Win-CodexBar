## ADDED Requirements

### Requirement: Zai Region Enum

系统 SHALL 定义 `ZaiRegion` 枚举，包含 `Global` 和 `ChinaMainland` 两个变体，并支持从字符串 `"global"` / `"china"` 解析。

#### Scenario: Parse global region string
- **WHEN** `ZaiRegion::from("global")` 被调用
- **THEN** 返回 `ZaiRegion::Global`

#### Scenario: Parse china region string
- **WHEN** `ZaiRegion::from("china")` 被调用
- **THEN** 返回 `ZaiRegion::ChinaMainland`

#### Scenario: Parse unknown region string
- **WHEN** `ZaiRegion::from("unknown")` 被调用
- **THEN** 返回 `ZaiRegion::Global`（默认值）

### Requirement: Region-based API URL Routing

ZaiProvider SHALL 根据当前 region 配置选择对应的 API 端点 URL。

#### Scenario: Global region API URL
- **WHEN** region 为 `Global`
- **THEN** API 请求发送到 `https://api.z.ai/api/monitor/usage/quota/limit`

#### Scenario: China Mainland region API URL
- **WHEN** region 为 `ChinaMainland`
- **THEN** API 请求发送到 `https://bigmodel.cn/api/monitor/usage/quota/limit`

### Requirement: Region-based Dashboard URL

系统 SHALL 根据 Z.ai 的 region 配置提供对应的 Dashboard URL。

#### Scenario: Global region dashboard URL
- **WHEN** region 为 `Global`
- **THEN** Dashboard URL 为 `https://z.ai/dashboard`

#### Scenario: China Mainland region dashboard URL
- **WHEN** region 为 `ChinaMainland`
- **THEN** Dashboard URL 为 `https://bigmodel.cn/`

### Requirement: Region-based Credential Storage

ZaiProvider SHALL 根据 region 使用不同的凭证存储 target，避免 global 和 china token 混用。

#### Scenario: Global region credential target
- **WHEN** region 为 `Global`
- **THEN** 凭证存储使用 target `codexbar-zai`

#### Scenario: China Mainland region credential target
- **WHEN** region 为 `ChinaMainland`
- **THEN** 凭证存储使用 target `codexbar-zai-china`

### Requirement: FetchContext Region Propagation

`FetchContext` SHALL 包含 `api_region` 字段，使 provider 在 fetch 时能获取当前 region 配置。

#### Scenario: FetchContext contains api_region
- **WHEN** 创建 `FetchContext` 用于 Z.ai fetch
- **THEN** `api_region` 字段包含当前 settings 中 `zai` provider 的 `api_region` 值（`"global"` 或 `"china"`）

#### Scenario: api_region defaults to global
- **WHEN** 创建 `FetchContext` 且 settings 中未配置 region
- **THEN** `api_region` 默认为 `"global"`
