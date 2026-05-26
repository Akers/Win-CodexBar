# Data Models

> Win-CodexBar 的核心数据模型，定义在 `rust/src/core/` 下，描述使用率窗口、快照、成本、配额等概念。

## ADDED Requirements

### Requirement: RateWindow

`RateWindow` in `rust/src/core/rate_window.rs` represents a single usage rate window (e.g., "session" or "weekly").

#### Scenario: Construction

- **WHEN** a `RateWindow` is created with `RateWindow::new(used_percent)`
- **THEN** it stores `used_percent: f64` (0.0-100.0)
- **AND** optional fields: `window_minutes`, `resets_at`, `reset_description`

#### Scenario: Threshold checks

- **WHEN** `is_exhausted()` is called on a RateWindow
- **THEN** it returns `true` if `used_percent >= 100.0`
- **AND** `is_nearly_exhausted()` returns `true` if `used_percent >= critical_usage_threshold` (default 90%)

#### Scenario: Remaining calculation

- **WHEN** `remaining_percent()` is called
- **THEN** it returns `100.0 - used_percent`, clamped to minimum 0.0

#### Scenario: Reset countdown

- **WHEN** `format_countdown()` is called and `resets_at` is set
- **THEN** it returns a human-readable string like "2h 15m" until reset
- **AND** returns `None` if `resets_at` is not set

### Requirement: UsageSnapshot

`UsageSnapshot` in `rust/src/core/usage_snapshot.rs` aggregates multiple rate windows for a single provider.

#### Scenario: Multi-window structure

- **WHEN** a `UsageSnapshot` is created
- **THEN** it contains: `primary: RateWindow` (required), `secondary: Option<RateWindow>`, `model_specific: Option<RateWindow>`, `tertiary: Option<RateWindow>`, `extra_rate_windows: Vec<NamedRateWindow>`
- **AND** metadata: `updated_at: DateTime<Utc>`, `account_email: Option<String>`, `account_organization: Option<String>`, `login_method: Option<String>`

#### Scenario: Most restrictive window

- **WHEN** `most_restrictive()` is called
- **THEN** it returns a reference to the RateWindow with the highest `used_percent` among all windows
- **AND** this is used for tray icon rendering and threshold notifications

#### Scenario: Exhaustion check

- **WHEN** `any_exhausted()` is called
- **THEN** it returns `true` if any rate window (primary, secondary, model_specific, tertiary, or extra) is exhausted
- **AND** this triggers the "exhausted" notification type

### Requirement: CostSnapshot

`CostSnapshot` tracks cost-related data for a provider.

#### Scenario: Cost structure

- **WHEN** a `CostSnapshot` is created
- **THEN** it contains cost information for the current billing period
- **AND** includes fields for: total cost, itemized costs by model or usage type, period start/end dates

### Requirement: UsagePace

`UsagePace` in `rust/src/core/usage_pace.rs` calculates usage velocity predictions.

#### Scenario: Pace calculation

- **WHEN** pace is calculated for a provider
- **THEN** it estimates time-until-exhaustion based on current usage rate
- **AND** categorizes pace into levels (e.g., "slow", "moderate", "fast", "critical")

### Requirement: SessionQuota

`SessionQuota` in `rust/src/core/session_quota.rs` models per-session usage limits.

#### Scenario: Session tracking

- **WHEN** a provider uses session-based quotas
- **THEN** `SessionQuota` tracks current session usage against the session limit
- **AND** calculates remaining session capacity

### Requirement: WidgetSnapshot

`WidgetSnapshot` in `rust/src/core/widget_snapshot.rs` is a display-optimized snapshot.

#### Scenario: Widget rendering data

- **WHEN** the frontend needs to render provider widgets
- **THEN** `WidgetSnapshot` provides pre-formatted data optimized for display
- **AND** includes display labels, formatted percentages, and status text

### Requirement: ProviderFetchResult

`ProviderFetchResult` is the unified return type for provider fetch operations.

#### Scenario: Result composition

- **WHEN** a provider fetch completes successfully
- **THEN** the result contains: `usage: UsageSnapshot` (required), `cost: Option<CostSnapshot>`, `source_label: String`
- **AND** this result is serialized and sent to the frontend via Tauri IPC

### Requirement: ProviderUsageSnapshot (Bridge Type)

`ProviderUsageSnapshot` in `bridge.ts` is the frontend counterpart of the backend result.

#### Scenario: Bridge type fields

- **WHEN** provider data arrives at the frontend
- **THEN** `ProviderUsageSnapshot` contains: `providerId`, `displayName`, `primary: RateWindowSnapshot`, `secondary`, `modelSpecific`, `tertiary`, `extraRateWindows`, `cost`, `planName`, `accountEmail`, `sourceLabel`, `updatedAt`, `error`, `pace`, `accountOrganization`, `trayStatusLabel`, `fetchDurationMs`

#### Scenario: Error state

- **WHEN** a provider fetch fails
- **THEN** `error` field contains the error message string
- **AND** all rate window fields may be null/default
- **AND** the UI shows an error state for that provider

### Requirement: Credentials

`Credentials` in `rust/src/core/credentials.rs` defines credential types.

#### Scenario: Credential types

- **WHEN** a provider authenticates
- **THEN** credentials are stored as one of: manual cookies, browser-extracted cookies, OAuth tokens, API keys, token accounts
- **AND** credentials are never logged or displayed in diagnostics (redacted via `Redactor`)

### Requirement: Token Accounts

`TokenAccount` in `rust/src/core/token_accounts.rs` represents API token-based billing.

#### Scenario: Token account structure

- **WHEN** a provider supports token-based billing (e.g., Alibaba, NanoGPT)
- **THEN** each token account has: name, token value, balance, active status
- **AND** the active token account is used for usage queries
