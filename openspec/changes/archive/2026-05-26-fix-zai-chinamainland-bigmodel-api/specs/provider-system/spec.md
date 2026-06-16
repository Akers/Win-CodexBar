## MODIFIED Requirements

### Requirement: Fetch Context Configuration

`FetchContext` controls how providers fetch data.

#### Scenario: Context fields
- **WHEN** a `FetchContext` is created
- **THEN** it contains: `source_mode: SourceMode`, `include_credits: bool`, `web_timeout: u64`, `verbose: bool`, `manual_cookie_header: Option<String>`, `api_key: Option<String>`, `api_region: Option<String>`
- **AND** `web_timeout` is in milliseconds and controls HTTP request timeout
- **AND** `manual_cookie_header` overrides browser cookie extraction when present
- **AND** `api_region` provides the region string for region-aware providers (e.g., `"global"`, `"china"`)

### Requirement: Provider-Specific Modules

Each provider has its own module under `rust/src/providers/` with isolated logic.

#### Scenario: Module boundaries
- **WHEN** Claude-specific parsing logic changes
- **THEN** only files in `rust/src/providers/claude/` are affected
- **AND** no other provider module or cross-provider branching is introduced

#### Scenario: Provider sub-fetcher pattern
- **WHEN** a provider supports multiple fetch methods (e.g., Claude with web, OAuth, and admin API)
- **THEN** it composes multiple fetcher structs internally (e.g., `ClaudeWebApiFetcher`, `ClaudeOAuthFetcher`, `ClaudeAdminApiFetcher`)
- **AND** the main provider struct delegates to the appropriate fetcher based on `SourceMode`

#### Scenario: Region-aware provider URL routing
- **WHEN** a provider supports multiple regions (e.g., Z.ai with Global and China Mainland)
- **THEN** the provider module contains a region enum and a URL dispatch method
- **AND** the fetch method uses `FetchContext.api_region` to select the correct endpoint
- **AND** credential storage is isolated per region to prevent token leakage across regions
