# Provider System

> Provider 系统是 Win-CodexBar 的核心，通过统一的 trait 抽象来支持 49+ 个 AI 编码工具提供商的使用限制查询。

## ADDED Requirements

### Requirement: Provider Trait Contract

All providers implement the `Provider` trait defined in `rust/src/core/provider.rs`.

#### Scenario: Trait method signatures

- **WHEN** a new provider is implemented
- **THEN** it must implement `Provider` trait with: `id()`, `metadata()`, `fetch_usage(ctx)`, `available_sources()`, `supports_oauth()`, `supports_web()`, `supports_cli()`, `detect_version()`
- **AND** `fetch_usage` returns `Result<ProviderFetchResult, ProviderError>` asynchronously

#### Scenario: Provider metadata

- **WHEN** `metadata()` is called on any provider
- **THEN** it returns `ProviderMetadata` containing: `id`, `display_name`, `session_label`, `weekly_label`, `supports_opus`, `supports_credits`, `default_enabled`, `is_primary`, `dashboard_url`, `status_page_url`
- **AND** metadata fields are all `&'static str` or static references (compile-time known)

### Requirement: Provider Registry (ProviderId)

The system identifies each provider by a `ProviderId` enum variant in `rust/src/core/provider.rs`.

#### Scenario: Complete provider enumeration

- **WHEN** the system enumerates all supported providers
- **THEN** `ProviderId` contains exactly these 49 variants: Codex, Claude, Cursor, Factory, Gemini, Antigravity, Copilot, Zai, MiniMax, Kiro, VertexAI, Augment, OpenCode, Kimi, KimiK2, Amp, Warp, Ollama, AzureOpenAI, T3Chat, OpenRouter, Synthetic, JetBrains, Alibaba, AlibabaTokenPlan, NanoGPT, Infini, Perplexity, Abacus, Mistral, OpenCodeGo, Kilo, Bedrock, Codebuff, DeepSeek, Windsurf, Manus, MiMo, Doubao, CommandCode, Crof, StepFun, Venice, OpenAIApi, Grok, ElevenLabs, Deepgram, Groq, LLMProxy
- **AND** each variant has a unique string representation used in settings, CLI flags, and frontend routing

### Requirement: Provider Factory

The `instantiate` function in `rust/src/core/provider_factory.rs` creates provider instances.

#### Scenario: Factory dispatches correct type

- **WHEN** `instantiate(ProviderId::Claude)` is called
- **THEN** it returns `Box::new(ClaudeProvider::new())`
- **AND** the returned `Box<dyn Provider>` can be used polymorphically

#### Scenario: Factory covers all providers

- **WHEN** `instantiate` is called with any `ProviderId` variant
- **THEN** it returns a valid `Box<dyn Provider>` implementation
- **AND** there are no `unimplemented!()` or `panic!()` branches

### Requirement: Source Mode Selection

Providers support multiple source modes for fetching usage data.

#### Scenario: Source mode enum

- **WHEN** a provider declares its available sources via `available_sources()`
- **THEN** it returns a `Vec<SourceMode>` where `SourceMode` is one of: `Auto`, `OAuth`, `Web`, `Cli`
- **AND** `Auto` mode attempts sources in priority order: OAuth → Web → Cli

#### Scenario: Fetch context source mode

- **WHEN** `fetch_usage` is called with `FetchContext { source_mode: Auto, .. }`
- **THEN** the provider tries the most authoritative source first (typically OAuth if authenticated)
- **AND** falls back to next available source on failure

### Requirement: Provider Fetch Result

All provider fetches return a standardized `ProviderFetchResult`.

#### Scenario: Result structure

- **WHEN** a provider successfully fetches usage data
- **THEN** it returns `ProviderFetchResult` containing: `usage: UsageSnapshot`, `cost: Option<CostSnapshot>`, `source_label: String`
- **AND** `source_label` indicates which source was used (e.g., "oauth", "web", "cli")

#### Scenario: Fetch error handling

- **WHEN** a provider fetch fails
- **THEN** it returns `ProviderError` with a descriptive message
- **AND** the error is surfaced to the frontend as `error: string` in the `ProviderUsageSnapshot`
- **AND** the tray icon renders in error state (desaturated colors)

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

### Requirement: Fetch Context Configuration

`FetchContext` controls how providers fetch data.

#### Scenario: Context fields

- **WHEN** a `FetchContext` is created
- **THEN** it contains: `source_mode: SourceMode`, `include_credits: bool`, `web_timeout: u64`, `verbose: bool`, `manual_cookie_header: Option<String>`, `api_key: Option<String>`, `api_region: Option<String>`
- **AND** `web_timeout` is in milliseconds and controls HTTP request timeout
- **AND** `manual_cookie_header` overrides browser cookie extraction when present
- **AND** `api_region` provides the region string for region-aware providers (e.g., `"global"`, `"china"`)

### Requirement: Provider Enable/Disable

Users control which providers are active.

#### Scenario: Default enabled providers

- **WHEN** a new installation starts with no saved settings
- **THEN** providers with `default_enabled: true` in their metadata are active
- **AND** this includes at minimum: Claude, Codex, Cursor, Copilot, Gemini

#### Scenario: Provider toggle

- **WHEN** user disables a provider in settings
- **THEN** the provider is removed from the refresh cycle
- **AND** its usage data is not shown in the UI
- **AND** its tray icon contribution is removed
