# CLI Module

> Win-CodexBar 的命令行接口，通过 `cargo run -p codexbar` 运行，支持 usage 查询、成本扫描、HTTP 服务等子命令。

## ADDED Requirements

### Requirement: CLI Entry Point

The CLI binary is built from the `codexbar` package in `rust/Cargo.toml`.

#### Scenario: Binary name and invocation

- **WHEN** the project is built with `cargo build -p codexbar`
- **THEN** it produces a `codexbar` binary (or `codexbar.exe` on Windows)
- **AND** running with no subcommand defaults to the `usage` command behavior

#### Scenario: Global flags

- **WHEN** `codexbar` is invoked
- **THEN** it accepts global flags: `--verbose`, `--json-output`, `--log-level <LEVEL>`, `--no-color`
- **AND** top-level flags for default usage mode: `--provider <NAME>`, `--format <FMT>`, `--json`, `--pretty`, `--status`, `--all-accounts`, `--no-credits`, `--source <MODE>`, `--web-timeout <MS>`

### Requirement: Usage Command

The `usage` subcommand queries provider rate limit usage.

#### Scenario: Single provider usage

- **WHEN** `codexbar usage -p claude` is executed
- **THEN** it fetches usage for the Claude provider
- **AND** outputs the session percentage, weekly percentage, reset time, and account email (if available)

#### Scenario: All providers usage

- **WHEN** `codexbar usage` is executed without a provider flag
- **THEN** it fetches usage for all enabled providers
- **AND** outputs a summary for each

#### Scenario: JSON output

- **WHEN** `codexbar usage -p claude --json` is executed
- **THEN** it outputs structured JSON with all usage fields
- **AND** the JSON schema matches `ProviderUsageSnapshot` bridge type

### Requirement: Cost Command

The `cost` subcommand scans local usage logs for cost estimation.

#### Scenario: Cost scanning

- **WHEN** `codexbar cost` is executed
- **THEN** it scans JSONL log files for API usage records
- **AND** calculates estimated costs based on provider pricing data

### Requirement: Serve Command

The `serve` subcommand starts an HTTP server for programmatic access.

#### Scenario: HTTP server mode

- **WHEN** `codexbar serve --port <PORT>` is executed
- **THEN** it starts an HTTP server on the specified port
- **AND** exposes REST endpoints for usage queries

### Requirement: Autostart Command

The `autostart` subcommand manages Windows login startup behavior.

#### Scenario: Enable autostart

- **WHEN** `codexbar autostart --enable` is executed on Windows
- **THEN** it writes a registry entry to `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`
- **AND** the application launches on next user login

#### Scenario: Disable autostart

- **WHEN** `codexbar autostart --disable` is executed on Windows
- **THEN** it removes the registry entry
- **AND** the application does not launch on login

### Requirement: Account Command

The `account` subcommand manages provider authentication.

#### Scenario: Login flow

- **WHEN** `codexbar account login -p claude` is executed
- **THEN** it initiates the OAuth flow for the specified provider
- **AND** stores credentials in the system keyring or secure file storage

#### Scenario: Logout flow

- **WHEN** `codexbar account logout -p claude` is executed
- **THEN** it removes stored credentials for the specified provider

### Requirement: Config Command

The `config` subcommand manages application settings.

#### Scenario: Show configuration

- **WHEN** `codexbar config show` is executed
- **THEN** it displays current settings in a readable format

#### Scenario: Set configuration

- **WHEN** `codexbar config set <KEY> <VALUE>` is executed
- **THEN** it updates the specified setting and persists to the TOML settings file

### Requirement: Exit Codes

The CLI uses consistent exit codes for error reporting.

#### Scenario: Exit code semantics

- **WHEN** a CLI command completes
- **THEN** exit codes follow: `0` = success, `1` = unexpected failure, `2` = provider missing, `3` = parse error, `4` = CLI timeout, `64` = usage error
- **AND** non-zero exits print a human-readable error message to stderr
