# System Architecture

> Win-CodexBar 是一个 Windows 桌面应用，用于监控 AI 编码工具（Claude、Codex、Cursor、Gemini 等 49+ 提供商）的使用限制、成本和配额。

## Overview

Win-CodexBar 采用 **三层架构**：

1. **共享后端层** (`rust/`)：Rust crate `codexbar`，包含所有业务逻辑、Provider 系统、数据模型、CLI 接口
2. **Tauri 桌面壳层** (`apps/desktop-tauri/src-tauri/`)：Rust 编写的 Tauri 后端，桥接系统托盘、窗口管理、快捷键和前端通信
3. **前端 UI 层** (`apps/desktop-tauri/src/`)：React + TypeScript 的单页应用，通过 Tauri IPC 桥接与后端交互

```
┌─────────────────────────────────────────────────┐
│                   Frontend (React)               │
│  TrayPanel · PopOut · Settings · FloatBar        │
├─────────────────────────────────────────────────┤
│              Tauri Command Bridge                │
│  invoke() ↔ commands/*.rs ↔ AppState            │
├─────────────────────────────────────────────────┤
│              Shared Backend (codexbar crate)     │
│  Providers · Core Models · Settings · Security   │
│  CLI · Browser Cookies · Tray Rendering · Sound  │
├─────────────────────────────────────────────────┤
│              OS Integration                      │
│  System Tray · DPAPI · Windows Registry · Toast  │
└─────────────────────────────────────────────────┘
```

## ADDED Requirements

### Requirement: Workspace Structure

The project uses a Cargo workspace with two members and a pnpm-managed frontend.

#### Scenario: Cargo workspace layout

- **WHEN** a developer clones the repository
- **THEN** the Cargo workspace at root contains two members: `rust` (shared crate) and `apps/desktop-tauri/src-tauri` (Tauri shell)
- **AND** the default build target is the Tauri shell (`default-members = ["apps/desktop-tauri/src-tauri"]`)

#### Scenario: Frontend package layout

- **WHEN** a developer navigates to `apps/desktop-tauri/`
- **THEN** pnpm manages React/TypeScript dependencies
- **AND** Vite serves the dev server on port 1420
- **AND** the frontend dist is built to `apps/desktop-tauri/dist/`

### Requirement: Module Dependency Direction

Dependencies flow strictly downward: Frontend → Tauri Commands → Shared Backend → OS APIs.

#### Scenario: No upward dependency

- **WHEN** the shared backend crate (`rust/`) is compiled
- **THEN** it has zero dependency on Tauri APIs or frontend code
- **AND** it can be built and tested independently via `cargo build -p codexbar`

#### Scenario: Tauri shell depends on shared crate

- **WHEN** the Tauri shell (`apps/desktop-tauri/src-tauri/`) is compiled
- **THEN** it depends on the `codexbar` crate via `path = "../../../rust"`
- **AND** it re-exports no codexbar types directly to the frontend—only via Tauri command serialization

### Requirement: Cross-Platform Awareness

The project targets Windows as primary platform with Linux/WSL support for development.

#### Scenario: Windows-specific code

- **WHEN** compiling for Windows (`cfg(windows)`)
- **THEN** DPAPI encryption, Windows Registry (autostart), Windows toast notifications, and Win32 window management APIs are available
- **AND** browser cookie extraction supports Chrome, Edge, Firefox with DPAPI decryption

#### Scenario: Non-Windows compilation

- **WHEN** compiling for non-Windows targets
- **THEN** Windows-specific features are stubbed or gated behind `cfg(windows)`
- **AND** the CLI and core logic still compile and function (with reduced OS integration)

## Key Data Flows

### Provider Usage Fetch Flow

```
User → Frontend (refresh button)
  → Tauri Command (refresh_providers)
    → AppState lock
      → For each enabled provider:
        → instantiate_provider(id) via factory
        → provider.fetch_usage(ctx) [async]
          → Source selection (auto → oauth → web → cli)
          → HTTP request or CLI invocation
          → Response parsing
        → ProviderFetchResult { usage, cost, source_label }
      → Update provider_cache
      → NotificationManager.check_and_notify()
    → Emit event to frontend
  → Frontend re-renders
```

### Settings Change Flow

```
User → Settings UI (toggle/change)
  → Tauri Command (update_settings)
    → Settings.save() to TOML file
    → Emit "codexbar:settings-updated" event
  → Frontend useSettings hook updates
  → If relevant: tray icon re-render, notification threshold update
```

## Module Summary

| Module | Location | Responsibility |
|--------|----------|---------------|
| `core` | `rust/src/core/` | Provider trait, data models, factory, redactor |
| `providers` | `rust/src/providers/` | 49+ provider implementations |
| `cli` | `rust/src/cli/` | Command-line interface (usage, cost, serve, config) |
| `settings` | `rust/src/settings.rs` | User preferences persistence |
| `browser` | `rust/src/browser/` | Browser detection and cookie extraction |
| `tray` | `rust/src/tray/` | Tray icon pixel rendering |
| `notifications` | `rust/src/notifications.rs` | System notification management |
| `secure_file` | `rust/src/secure_file.rs` | DPAPI-encrypted file storage |
| `commands` | `apps/desktop-tauri/src-tauri/src/commands/` | Tauri IPC command handlers |
| `shell` | `apps/desktop-tauri/src-tauri/src/shell/` | Window management and positioning |
| `floatbar` | `apps/desktop-tauri/src-tauri/src/floatbar/` | Floating bar window |
| `surfaces` | `apps/desktop-tauri/src/surfaces/` | React UI surface components |
| `hooks` | `apps/desktop-tauri/src/hooks/` | React state management hooks |
