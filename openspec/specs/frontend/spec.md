# Frontend UI

> 基于 React 18 + TypeScript 的前端应用，运行在 Tauri WebView 中，通过 IPC 桥接与 Rust 后端通信。

## ADDED Requirements

### Requirement: Application Entry and Routing

The frontend entry is `apps/desktop-tauri/src/App.tsx`.

#### Scenario: Initial bootstrap

- **WHEN** the React app mounts
- **THEN** it calls `getBootstrapState()` to load initial state
- **AND** configures theme based on `bootstrap.settings.theme`
- **AND** initiates the first update check via `checkForUpdates()`
- **AND** listens for `"global-shortcut-triggered"` events to toggle tray panel

#### Scenario: Surface routing

- **WHEN** the app renders
- **THEN** it determines which surface to show based on window identity:
  - Settings window → `<DetachedSettingsApp>`
  - Float bar window → `<FloatBar>`
  - Main window → `<SurfaceRouter>` which switches between `TrayPanel`, `PopOutPanel`, `Settings`

### Requirement: Tray Panel Surface

`TrayPanel` is the primary interaction surface, shown as a dropdown from the system tray.

#### Scenario: Provider grid navigation

- **WHEN** the tray panel opens in overview mode
- **THEN** it shows a provider grid with an "All" button and one button per provider
- **AND** clicking a provider button switches to detail mode showing only that provider
- **AND** clicking "All" returns to overview mode

#### Scenario: Provider card display

- **WHEN** a provider card is rendered in the menu stack
- **THEN** it shows: provider icon, display name, usage bar with percentage, session/weekly labels
- **AND** optionally shows: account email, plan name, cost, reset countdown, pace badge
- **AND** email is hidden when `settings.hidePersonalInfo` is true

#### Scenario: Dynamic window resize

- **WHEN** the number of visible providers changes
- **THEN** the panel measures its content height
- **AND** resizes the Tauri window to fit (clamped 200-920px height, fixed 310px width)
- **AND** re-anchors the window position relative to the tray icon

#### Scenario: Refresh behavior

- **WHEN** user clicks the refresh button
- **THEN** `refresh_providers()` is called via Tauri command
- **AND** a loading spinner is shown during refresh
- **AND** the panel shows cached data while refreshing

### Requirement: Settings Surface

The settings surface provides tabbed configuration UI.

#### Scenario: Settings tabs

- **WHEN** the settings surface is opened
- **THEN** it shows tabs: General, Providers, Display, Advanced, About, API Keys, Cookies, Token Accounts
- **AND** each tab manages a distinct settings domain
- **AND** changes are persisted via `update_settings()` on each modification

#### Scenario: Provider detail pane

- **WHEN** a provider is selected in the Providers tab
- **THEN** the detail pane shows: Identity section, Usage section, Cost section, Pace section
- **AND** Quick Actions: open dashboard, open status page, trigger login, revoke credentials
- **AND** provider-specific settings: Cookie Source, Region, API Key

### Requirement: Float Bar Surface

The float bar is a compact, always-on-top overlay.

#### Scenario: Float bar rendering

- **WHEN** the float bar is visible
- **THEN** it renders a narrow horizontal bar with compact usage indicators for selected providers
- **AND** supports configurable opacity (0-255), orientation (horizontal/vertical), and click-through mode

#### Scenario: Float bar settings section

- **WHEN** the float bar settings section is shown
- **THEN** it provides controls for: enable/disable, opacity slider, orientation toggle, click-through toggle, provider selection, dark text toggle

### Requirement: State Management Hooks

State is managed through custom React hooks in `apps/desktop-tauri/src/hooks/`.

#### Scenario: useProviders hook

- **WHEN** `useProviders()` is called
- **THEN** it returns `{ providers: ProviderUsageSnapshot[], isRefreshing: boolean, refresh: () => void, hasCachedData: boolean }`
- **AND** it manages provider data fetching and caching lifecycle

#### Scenario: useSettings hook

- **WHEN** `useSettings(snapshot)` is called with initial snapshot
- **THEN** it returns `{ settings: SettingsSnapshot, update: (partial) => void }`
- **AND** it listens for `"codexbar:settings-updated"` DOM events to sync state

#### Scenario: useUpdateState hook

- **WHEN** `useUpdateState()` is called
- **THEN** it returns `{ updateState, checkNow, download, apply, dismiss, openRelease }`
- **AND** manages the update lifecycle (check → download → apply → dismiss)

### Requirement: Tauri Bridge Types

TypeScript types in `apps/desktop-tauri/src/types/bridge.ts` mirror Rust structs.

#### Scenario: Type contract

- **WHEN** the frontend sends/receives data via Tauri IPC
- **THEN** all types are defined in `bridge.ts` and match the Rust serialization output
- **AND** key types include: `SurfaceMode`, `ProviderUsageSnapshot`, `BootstrapState`, `SettingsSnapshot`, `RateWindowSnapshot`, `CostSnapshotBridge`

#### Scenario: Contract versioning

- **WHEN** `BootstrapState` is received
- **THEN** it includes a `contractVersion` field
- **AND** the frontend validates compatibility with its expected version

### Requirement: Internationalization (i18n)

The frontend supports English and Chinese locales.

#### Scenario: Locale provider

- **WHEN** the app initializes
- **THEN** `LocaleProvider` wraps the entire app
- **AND** `useLocale()` hook provides `t(key)` function for string lookup
- **AND** language preference is stored in `settings.uiLanguage`

#### Scenario: Language switching

- **WHEN** user changes the UI language in settings
- **THEN** all UI strings update immediately
- **AND** the preference is persisted via `update_settings()`

### Requirement: Theme Support

The frontend supports Auto, Light, and Dark themes.

#### Scenario: Theme application

- **WHEN** `useTheme(preference)` is called
- **THEN** CSS custom properties or class names are applied based on the theme
- **AND** `Auto` mode follows the system dark/light preference

### Requirement: Chart Components

The frontend renders usage and cost charts.

#### Scenario: Chart types

- **WHEN** provider detail view shows charts
- **THEN** it renders: BarChart (usage breakdown), LineChart (usage/cost history), usage breakdown chart, credits history chart
- **AND** charts use a consistent color palette from `chartPalette.ts`
- **AND** charts support animation via `useChartAnimation` hook
