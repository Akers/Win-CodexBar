# Desktop Shell (Tauri)

> Tauri 桌面壳层负责系统托盘集成、窗口管理、全局快捷键和前后端桥接。

## ADDED Requirements

### Requirement: Application Lifecycle

The Tauri app starts via `apps/desktop-tauri/src-tauri/src/main.rs`.

#### Scenario: Startup sequence

- **WHEN** the Tauri app launches
- **THEN** it initializes logging, loads `AppState` into Tauri managed state
- **AND** registers the single-instance plugin (second invocation reopens the tray panel)
- **AND** registers the shortcut bridge plugin for global hotkeys
- **AND** calls `tray_bridge::setup(app)` to create the system tray icon and menu
- **AND** calls `floatbar::install(app.handle())` to initialize the floating bar

#### Scenario: Single instance enforcement

- **WHEN** a second instance of the app is launched
- **THEN** the primary instance receives the activation signal
- **AND** reopens the tray panel via `shell::reopen_to_target(app, SurfaceMode::TrayPanel, ...)`
- **AND** the second instance exits

### Requirement: Tauri Command Bridge

The shell exposes backend functionality to the frontend via Tauri `invoke_handler`.

#### Scenario: Bootstrap state command

- **WHEN** frontend calls `get_bootstrap_state()`
- **THEN** it returns `BootstrapState` containing: `contract_version`, `surface_modes`, `commands`, `events`, `providers`, `settings`
- **AND** this is the first call the frontend makes on startup

#### Scenario: Provider refresh commands

- **WHEN** frontend calls `refresh_providers()`
- **THEN** it locks `AppState`, iterates enabled providers, fetches usage for each, updates cache
- **AND** emits a frontend event when complete

#### Scenario: Settings commands

- **WHEN** frontend calls `update_settings(snapshot)`
- **THEN** it updates the settings struct and calls `settings.save()`
- **AND** emits `"codexbar:settings-updated"` event to all windows

### Requirement: Application State

`AppState` in `apps/desktop-tauri/src-tauri/src/state.rs` holds runtime state.

#### Scenario: State fields

- **WHEN** AppState is accessed
- **THEN** it contains: `is_refreshing: bool`, `provider_refresh_started_at: Option<Instant>`, `provider_cache_updated_at: Option<Instant>`, `provider_cache: Vec<ProviderUsageSnapshot>`, `notification_manager: NotificationManager`, `proof_config: Option<ProofConfig>`, `last_shown_at: Option<Instant>`
- **AND** it is wrapped in `Mutex<AppState>` for thread-safe access

### Requirement: Surface Mode Management

The app supports multiple display surfaces (window modes).

#### Scenario: Surface mode enum

- **WHEN** the frontend sets a surface mode
- **THEN** it selects from: `hidden` (no window visible), `trayPanel` (dropdown from tray), `popOut` (detached panel), `settings` (settings window)
- **AND** each mode has a corresponding window configuration

#### Scenario: Tray panel positioning

- **WHEN** surface mode changes to `trayPanel`
- **THEN** the window is positioned below the system tray icon
- **AND** the window has no decorations, is always-on-top, and skips taskbar

### Requirement: System Tray Integration

The tray bridge (`apps/desktop-tauri/src-tauri/src/tray_bridge.rs`) manages the system tray.

#### Scenario: Tray icon rendering

- **WHEN** provider data is refreshed
- **THEN** the tray icon is rendered as a 32x32 RGBA image showing usage bars
- **AND** the icon uses color coding: green (low usage) → yellow → red (critical/exhausted)
- **AND** the icon shows error state with desaturated colors when a provider fails

#### Scenario: Tray menu

- **WHEN** the user right-clicks the tray icon
- **THEN** a context menu appears with options: refresh, show panel, settings, quit
- **AND** left-click toggles the tray panel visibility

### Requirement: Window Management

The shell module (`apps/desktop-tauri/src-tauri/src/shell/`) handles window lifecycle.

#### Scenario: Frameless window

- **WHEN** the main window is shown
- **THEN** it has no native decorations (title bar, border)
- **AND** the frontend renders custom window controls
- **AND** the window uses DWM (Desktop Window Manager) for blur/transparency effects

#### Scenario: Dynamic window sizing

- **WHEN** the tray panel content changes (provider count, data updates)
- **THEN** the frontend measures content height and calls `window.setSize()`
- **AND** calls `reanchor_tray_panel()` to reposition relative to the tray icon
- **AND** height is clamped between `MIN_HEIGHT=200` and `MAX_HEIGHT=920` pixels

### Requirement: Float Bar

The float bar (`apps/desktop-tauri/src-tauri/src/floatbar/`) provides a compact always-on-top overlay.

#### Scenario: Float bar visibility

- **WHEN** `show_float_bar` command is invoked
- **THEN** a narrow horizontal window appears at the configured screen edge
- **AND** it shows compact usage bars for selected providers
- **AND** it can optionally be click-through (non-interactive overlay)

#### Scenario: Float bar configuration

- **WHEN** user changes float bar settings (opacity, orientation, providers)
- **THEN** the float bar window updates immediately
- **AND** settings persist across restarts

### Requirement: Global Shortcut

The shortcut bridge registers a system-wide keyboard shortcut.

#### Scenario: Shortcut registration

- **WHEN** user configures a global shortcut (e.g., `Alt+Shift+C`)
- **THEN** `register_global_shortcut` registers it with the OS
- **AND** pressing the shortcut anywhere in the OS toggles the tray panel

#### Scenario: Shortcut change

- **WHEN** user changes the global shortcut in settings
- **THEN** the old shortcut is unregistered and the new one is registered
- **AND** the shortcut persists across app restarts

### Requirement: Provider Detail View

The shell provides commands for detailed provider inspection.

#### Scenario: Provider detail fetch

- **WHEN** frontend calls `get_provider_detail(provider_id)`
- **THEN** it fetches fresh usage data for the specific provider
- **AND** returns detailed information including cost breakdown, pace, and chart data

#### Scenario: Dashboard and status page links

- **WHEN** frontend calls `open_provider_dashboard(provider_id)`
- **THEN** it opens the provider's usage dashboard URL in the default browser
- **AND** only providers with `dashboard_url` defined support this command

### Requirement: Event System

The shell emits events to the frontend for reactive updates.

#### Scenario: Settings update event

- **WHEN** settings are changed via `update_settings`
- **THEN** a `"codexbar:settings-updated"` DOM event is dispatched on the window
- **AND** the frontend `useSettings` hook re-reads the settings

#### Scenario: Shortcut triggered event

- **WHEN** the global shortcut is pressed
- **THEN** a `"global-shortcut-triggered"` Tauri event is emitted
- **AND** the frontend toggles the tray panel surface mode
