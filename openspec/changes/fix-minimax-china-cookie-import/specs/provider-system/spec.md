## ADDED Requirements

### Requirement: Region-Aware Cookie Domain Resolution

Provider system SHALL support region-aware browser cookie domain resolution for providers with multi-region support.

#### Scenario: MiniMax region-aware cookie import
- **WHEN** user initiates cookie import for MiniMax with China Mainland region selected
- **THEN** the cookie import command uses `"platform.minimaxi.com"` as the search domain
- **AND** the `import_browser_cookies` Tauri command accepts an optional `region` parameter to enable this behavior

#### Scenario: Provider router supports multi-domain cookie lookup
- **WHEN** a provider has multiple login domains based on region (e.g., Global vs China)
- **THEN** the provider's region enum exposes a `cookie_domain()` method returning the appropriate domain per region

### Requirement: Provider API Key Context Usage

Provider refresh SHALL pass saved API keys through `FetchContext.api_key` and providers MAY prefer this key over cookie/browser flows when it is more reliable.

#### Scenario: MiniMax saved API key usage
- **WHEN** user saves a MiniMax API key through Settings → API Keys
- **THEN** `build_fetch_context` provides that key as `FetchContext.api_key`
- **AND** MiniMax refresh uses the API-key `token_plan/remains` path before cookie billing
