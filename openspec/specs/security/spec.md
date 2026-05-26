# Security

> 安全模块涵盖凭证存储、Cookie 提取、敏感信息脱敏和安全文件操作。

## ADDED Requirements

### Requirement: Secure File Storage

`rust/src/secure_file.rs` provides DPAPI-encrypted file storage on Windows.

#### Scenario: Windows DPAPI encryption

- **WHEN** `write_string(path, content)` is called on Windows
- **THEN** it encrypts the content using Windows DPAPI `CryptProtectData`
- **AND** writes the encrypted blob to the specified file path

#### Scenario: Windows DPAPI decryption

- **WHEN** `read_string(path)` is called on Windows
- **THEN** it reads the encrypted file
- **AND** decrypts using `CryptUnprotectData`
- **AND** returns the original plaintext string

#### Scenario: Non-Windows fallback

- **WHEN** secure file operations are called on non-Windows platforms
- **THEN** they fall back to base64 encoding or plaintext (platform-dependent)
- **AND** log a warning that DPAPI is not available

### Requirement: Browser Cookie Extraction

`rust/src/browser/cookies.rs` extracts cookies from installed browsers.

#### Scenario: Supported browsers

- **WHEN** cookie extraction is attempted
- **THEN** the system detects and supports: Google Chrome, Microsoft Edge, Mozilla Firefox
- **AND** each browser's cookie database location is resolved per-platform

#### Scenario: Cookie extraction process

- **WHEN** `CookieExtractor::extract_for_domain(browser, domain)` is called
- **THEN** it opens the browser's SQLite cookie database
- **AND** queries cookies matching the specified domain
- **AND** decrypts encrypted cookie values using DPAPI (Windows) or the browser's key
- **AND** returns `Vec<Cookie>` with name, value, domain, path, expires, secure, httpOnly

#### Scenario: Chrome App-Bound Encryption (ABE)

- **WHEN** Chrome 127+ uses App-Bound Encryption
- **THEN** `detect_app_bound_encryption()` checks the `Local State` file for `app_bound_encrypted_key`
- **AND** returns `CookieError::AppBoundEncryption` if ABE is detected
- **AND** the UI shows a user-friendly message explaining the limitation

#### Scenario: Cookie header construction

- **WHEN** `build_cookie_header(cookies)` is called
- **THEN** it constructs a `Cookie` HTTP header string from the cookie list
- **AND** formats as `name1=value1; name2=value2`

### Requirement: Sensitive Information Redaction

`rust/src/core/redactor.rs` prevents sensitive data from leaking in logs and diagnostics.

#### Scenario: Redaction patterns

- **WHEN** `Redactor::redact(text)` is called
- **THEN** it replaces: email addresses, API tokens (32+ alphanumeric chars), long cookie values (20+ chars)
- **AND** replaces matches with `[REDACTED]`

#### Scenario: Diagnostic safety

- **WHEN** `get_safe_diagnostics()` is called
- **THEN** the returned `SafeDiagnostics` struct excludes: actual cookie values, API keys, account emails
- **AND** only indicates whether cookies/keys exist (boolean flags)

### Requirement: Credential Management

Credentials are managed through multiple storage mechanisms.

#### Scenario: Manual cookie storage

- **WHEN** user provides a manual cookie header in settings
- **THEN** it is stored via `set_manual_cookie(provider_id, cookie_header)`
- **AND** retrieved via `get_manual_cookies()` which returns only provider IDs, not values

#### Scenario: API key storage

- **WHEN** user provides an API key
- **THEN** it is stored via `set_api_key(provider_id, key)`
- **AND** `get_api_keys()` returns available provider IDs without key values
- **AND** actual key values are only accessed during fetch operations

#### Scenario: OAuth credential storage

- **WHEN** a provider completes OAuth flow
- **THEN** tokens are stored in the system keyring via the `keyring` crate
- **AND** if keychain access is disabled (`disable_keychain_access`), falls back to secure file storage

#### Scenario: Credential revocation

- **WHEN** `revoke_provider_credentials(provider_id)` is called
- **THEN** it removes all stored credentials for that provider: keyring entries, secure files, manual cookies, API keys
- **AND** the provider returns to unauthenticated state

### Requirement: Content Security Policy

The Tauri WebView enforces a strict CSP.

#### Scenario: CSP rules

- **WHEN** the Tauri app loads
- **THEN** the CSP policy restricts: `default-src 'self'`, `script-src 'self'`, `style-src 'self' 'unsafe-inline'`, `img-src 'self' data:`, `connect-src 'self' ipc: http://ipc.localhost http://localhost:*`
- **AND** `object-src 'none'`, `base-uri 'none'`, `frame-ancestors 'none'`

### Requirement: Logging Security

The logging system prevents sensitive data from appearing in logs.

#### Scenario: Sensitive data in logs

- **WHEN** any module logs a message
- **THEN** raw secrets, cookies, and tokens are never logged
- **AND** the `tracing` framework is used with appropriate log levels
- **AND** debug-level logs may contain redacted versions of sensitive data
