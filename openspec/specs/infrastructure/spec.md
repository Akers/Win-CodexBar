# Infrastructure

> 构建系统、CI/CD 管道、开发环境和发布流程。

## ADDED Requirements

### Requirement: Cargo Workspace Build

The project uses a Cargo workspace with resolver v3.

#### Scenario: Workspace structure

- **WHEN** `cargo build` is run from the project root
- **THEN** it builds both workspace members: `rust` and `apps/desktop-tauri/src-tauri`
- **AND** the default member is `apps/desktop-tauri/src-tauri` (the Tauri shell)

#### Scenario: Shared crate standalone build

- **WHEN** `cargo build -p codexbar` is run
- **THEN** it builds only the shared Rust crate
- **AND** produces a CLI binary without Tauri dependencies
- **AND** does not require Tauri or frontend tooling

#### Scenario: Release profile

- **WHEN** building in release mode
- **THEN** the Rust profile uses: `opt-level = 3`, `lto = true`, `codegen-units = 1`, `strip = true`, `panic = "abort"`
- **AND** produces an optimized, stripped binary

### Requirement: Frontend Build

The React frontend is built with Vite and bundled by Tauri.

#### Scenario: Development server

- **WHEN** `pnpm run dev` or `tauri:dev` is started
- **THEN** Vite serves the frontend on `http://127.0.0.1:1420`
- **AND** Tauri's dev mode proxies to this URL

#### Scenario: Production build

- **WHEN** `pnpm run build` is executed
- **THEN** Vite builds the React app to `apps/desktop-tauri/dist/`
- **AND** Tauri bundles the dist into the final binary

#### Scenario: Frontend testing

- **WHEN** `pnpm test` is executed
- **THEN** Vitest runs tests in jsdom environment
- **AND** test files match `src/**/*.{test,spec}.{ts,tsx}`

### Requirement: Tauri Build and Packaging

The Tauri app is packaged for Windows distribution.

#### Scenario: Tauri build command

- **WHEN** `npm run tauri:build` is executed
- **THEN** it runs `beforeBuildCommand: pnpm run build` for the frontend
- **AND** compiles the Rust backend in release mode
- **AND** produces an executable at `target/release/codexbar-desktop-tauri.exe`

#### Scenario: Inno Setup installer

- **WHEN** the release workflow runs
- **THEN** Inno Setup builds an installer from `rust/installer/codexbar.iss`
- **AND** the installer bundles: the main executable, VC++ Redistributable, WebView2 Bootstrapper
- **AND** produces `CodexBar-{VERSION}-Setup.exe`

#### Scenario: Portable build

- **WHEN** the release workflow runs
- **THEN** it also produces a portable executable `CodexBar-{VERSION}-portable.exe`
- **AND** both installer and portable have SHA-256 checksums generated

### Requirement: Development Scripts

Development scripts simplify the build and run cycle.

#### Scenario: Linux/WSL development

- **WHEN** `./dev.sh` is executed
- **THEN** it checks for Rust and pnpm installations
- **AND** optionally builds the CLI (`--cli`) or desktop shell
- **AND** supports flags: `--release`, `--skip-build`, `--verbose`

#### Scenario: Windows development

- **WHEN** `.\dev.ps1` is executed
- **THEN** it checks Rust and MinGW installations (installs MinGW if needed)
- **AND** builds and runs the desktop shell
- **AND** supports flags: `-Release`, `-SkipBuild`, `-Verbose`

### Requirement: CI Pipeline

GitHub Actions CI runs on every push and pull request.

#### Scenario: Linux quality checks

- **WHEN** a PR is created
- **THEN** CI runs on Ubuntu 24.04: `cargo fmt --check`, `cargo clippy -D warnings`, `cargo test` for both manifests
- **AND** also builds and tests the frontend with `pnpm install && pnpm build`

#### Scenario: Windows build and test

- **WHEN** a PR is created
- **THEN** CI runs on Windows: `cargo clippy`, `cargo test`, release build, smoke test (`codexbar.exe --version`)
- **AND** targets `x86_64-pc-windows-msvc`

### Requirement: Release Workflow

The release workflow creates distributable assets.

#### Scenario: Manual release trigger

- **WHEN** the release workflow is manually triggered
- **THEN** it reads the version from `rust/Cargo.toml`
- **AND** builds the Tauri app in release mode
- **AND** creates both installer and portable executables with SHA-256 checksums
- **AND** uploads all assets to the GitHub release

### Requirement: Code Quality

The project enforces code quality through automated checks.

#### Scenario: Formatting

- **WHEN** `cargo fmt --all` is run
- **THEN** it formats all Rust code according to project standards
- **AND** CI checks formatting with `--check`

#### Scenario: Linting

- **WHEN** `cargo clippy --all-targets -- -D warnings` is run
- **THEN** it catches common mistakes and enforces best practices
- **AND** treats all warnings as errors (no warnings allowed in CI)

### Requirement: Dependency Management

Dependencies are managed carefully to minimize bloat.

#### Scenario: Rust dependencies

- **WHEN** a new Rust dependency is considered
- **THEN** it must not duplicate existing functionality
- **AND** must be actively maintained
- **AND** requires confirmation before addition (per AGENTS.md guidelines)

#### Scenario: Frontend dependencies

- **WHEN** pnpm installs dependencies
- **THEN** it uses a lockfile (`pnpm-lock.yaml`) for reproducibility
- **AND** CI uses `--frozen-lockfile` to detect lockfile drift
