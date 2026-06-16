# Settings Management

> 用户偏好设置的持久化管理，支持 TOML 文件存储、Provider 级别配置和 Windows 注册表集成。

## ADDED Requirements

### Requirement: Settings Structure

`Settings` in `rust/src/settings.rs` defines all user-configurable options.

#### Scenario: Global settings fields

- **WHEN** Settings is loaded
- **THEN** it contains these fields:
  - `enabled_providers: HashSet<String>` — active provider IDs
  - `refresh_interval_secs: u64` — auto-refresh interval (default 300)
  - `start_minimized: bool` — start without showing UI
  - `start_at_login: bool` — launch on Windows login
  - `show_notifications: bool` — enable toast notifications
  - `sound_enabled: bool` — play sound on threshold events
  - `sound_volume: u8` — notification volume (0-100)
  - `high_usage_threshold: f64` — percentage for "high" warning (default 75)
  - `critical_usage_threshold: f64` — percentage for "critical" warning (default 90)
  - `tray_icon_mode: TrayIconMode` — single icon, merged icons, or per-provider
  - `merge_tray_icons: bool` — combine provider icons
  - `show_as_used: bool` — invert percentage display (show used vs remaining)
  - `surprise_animations: bool` — enable occasional animations
  - `enable_animations: bool` — general animation toggle
  - `reset_time_relative: bool` — show relative vs absolute reset time
  - `hide_personal_info: bool` — redact emails in UI
  - `show_debug_settings: bool` — show advanced debug options
  - `disable_keychain_access: bool` — skip OS keychain for credentials
  - `update_channel: UpdateChannel` — stable or pre-release
  - `global_shortcut: String` — keyboard shortcut for tray toggle
  - `auto_download_updates: bool` — automatically download updates
  - `install_updates_on_quit: bool` — apply updates on exit
  - `ui_language: Language` — English or Chinese
  - `theme: ThemePreference` — Auto, Light, or Dark

#### Scenario: Float bar settings fields

- **WHEN** Settings includes float bar configuration
- **THEN** it contains: `float_bar_enabled: bool`, `float_bar_opacity: u8`, `float_bar_orientation: String`, `float_bar_click_through: bool`, `float_bar_provider_ids: Vec<String>`, `float_bar_dark_text: bool`

### Requirement: Provider-Level Configuration

`ProviderConfig` in `rust/src/settings.rs` stores per-provider overrides.

#### Scenario: Provider config fields

- **WHEN** a provider has custom configuration
- **THEN** `ProviderConfig` contains: `cookie_source: Option<String>`, `usage_source: Option<String>`, `api_region: Option<String>`, `manual_cookie_header: Option<String>`, `api_token: Option<String>`, `workspace_id: Option<String>`, `ide_base_path: Option<String>`, `openai_web_extras: Option<bool>`, `historical_tracking: bool`, `avoid_keychain_prompts: bool`

#### Scenario: Provider config accessors

- **WHEN** `settings.cookie_source(ProviderId::Claude)` is called
- **THEN** it returns the cookie source for Claude, or the default if not set
- **AND** similar accessors exist for: `usage_source`, `api_region`, `manual_cookie_header`, `api_token`

### Requirement: Settings Persistence

Settings are saved to a TOML file in the user's app data directory.

#### Scenario: Load settings

- **WHEN** `Settings::load()` is called
- **THEN** it reads from the OS-specific app data directory (e.g., `%APPDATA%/codexbar/settings.toml` on Windows)
- **AND** returns default values for any missing fields

#### Scenario: Save settings

- **WHEN** `settings.save()` is called
- **THEN** it serializes all settings to TOML format
- **AND** writes to the app data directory
- **AND** creates the directory if it doesn't exist

### Requirement: Windows Autostart Integration

The `start_at_login` setting integrates with Windows Registry.

#### Scenario: Enable autostart

- **WHEN** `set_start_at_login(true)` is called
- **THEN** it writes a `String` value to `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` with key `"CodexBar"` and value being the app executable path
- **AND** returns `Ok(())` on success

#### Scenario: Disable autostart

- **WHEN** `set_start_at_login(false)` is called
- **THEN** it deletes the `"CodexBar"` value from the Run key
- **AND** returns `Ok(())` on success (or if key didn't exist)

#### Scenario: Check autostart status

- **WHEN** `is_start_at_login_enabled()` is called
- **THEN** it checks for the `"CodexBar"` entry in the Run key
- **AND** returns `true` only if the entry exists and points to the current executable

### Requirement: Settings Snapshot for Frontend

The Tauri command `get_settings_snapshot` provides a serialization-friendly snapshot.

#### Scenario: Snapshot conversion

- **WHEN** `get_settings_snapshot()` is called
- **THEN** it converts the internal `Settings` struct to a `SettingsSnapshot` matching the `bridge.ts` type
- **AND** sensitive fields (manual cookies, API tokens) are not included in the snapshot
- **AND** the snapshot is suitable for rendering the settings UI

### Requirement: Provider Metrics Preferences

Users can customize which metrics are displayed for each provider.

#### Scenario: Metric preference structure

- **WHEN** `provider_metrics` is configured
- **THEN** it maps provider IDs to `MetricPreference` structs
- **AND** `MetricPreference` controls which usage windows and labels are shown

### Requirement: Provider Ordering

Users can reorder providers in the UI.

#### Scenario: Custom provider order

- **WHEN** `provider_order: Vec<String>` is set
- **THEN** providers are displayed in the specified order in the tray panel and settings
- **AND** providers not in the order list appear after ordered providers, sorted alphabetically

#### Scenario: Reorder command

- **WHEN** frontend calls `reorder_providers(ordered_ids)`
- **THEN** it updates `settings.provider_order` and saves
- **AND** emits settings update event
