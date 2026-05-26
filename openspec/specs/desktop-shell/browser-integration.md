# Browser Integration

> 浏览器检测和 Cookie 提取模块，用于 Provider 的 Web 模式认证。

## ADDED Requirements

### Requirement: Browser Detection

`rust/src/browser/` detects installed browsers on the system.

#### Scenario: Detect available browsers

- **WHEN** `list_detected_browsers()` is called
- **THEN** it scans the system for installed browsers
- **AND** returns a list of detected browsers with their names and installation paths

#### Scenario: Browser enumeration for cookie import

- **WHEN** the frontend shows the Cookie Import UI
- **THEN** it calls `list_detected_browsers()` to populate the browser selector
- **AND** only shows browsers that are actually installed

### Requirement: Cookie Extraction Pipeline

`CookieExtractor` in `rust/src/browser/cookies.rs` extracts cookies from browser databases.

#### Scenario: Chrome cookie extraction

- **WHEN** cookies are extracted from Chrome
- **THEN** the extractor:
  1. Locates Chrome's cookie database at `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Cookies` (or profile-specific path)
  2. Opens the SQLite database with shared lock (Chrome may have it open)
  3. Reads the encryption key from Chrome's `Local State` file
  4. Decrypts the AES-GCM encrypted cookie values using DPAPI-decrypted key
  5. Returns matching cookies for the requested domain

#### Scenario: Edge cookie extraction

- **WHEN** cookies are extracted from Microsoft Edge
- **THEN** the extractor follows the same Chrome-based process (Chromium shared codebase)
- **AND** looks for Edge's data directory at `%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Cookies`

#### Scenario: Firefox cookie extraction

- **WHEN** cookies are extracted from Firefox
- **THEN** the extractor:
  1. Locates Firefox's cookie database at `%APPDATA%\Mozilla\Firefox\Profiles\{profile}\cookies.sqlite`
  2. Opens the SQLite database
  3. Handles Firefox's distinct cookie encryption scheme
  4. Returns matching cookies for the requested domain

### Requirement: Cookie Import UI Flow

The frontend provides a guided cookie import experience.

#### Scenario: Browser cookie import command

- **WHEN** `import_browser_cookies(provider_id, browser_name)` is called
- **THEN** it extracts cookies for the provider's domain from the specified browser
- **AND** stores them as the active cookie source for that provider
- **AND** returns success/failure status

#### Scenario: Import error handling

- **WHEN** cookie import fails (browser not installed, ABE detected, no cookies found)
- **THEN** a descriptive error message is returned
- **AND** the UI shows actionable guidance (e.g., "Chrome App-Bound Encryption is enabled. Try using Edge instead.")

### Requirement: Cookie Source Configuration

Each provider can use cookies from different sources.

#### Scenario: Cookie source options

- **WHEN** `get_provider_cookie_source_options(provider_id)` is called
- **THEN** it returns available cookie sources for the provider
- **AND** sources include: detected browsers, manual entry, none

#### Scenario: Set cookie source

- **WHEN** `set_provider_cookie_source(provider_id, source)` is called
- **THEN** it configures which cookie source the provider uses for web fetches
- **AND** persists the choice in settings

### Requirement: Concurrent Database Access

Cookie extraction must handle concurrent browser database access.

#### Scenario: Browser has database locked

- **WHEN** Chrome is running and has the cookie database locked
- **THEN** the extractor uses shared/read-only mode to open the database
- **AND** falls back to copying the database file if shared mode fails

#### Scenario: WSL access to Windows browser data

- **WHEN** running under WSL
- **THEN** browser cookie paths are resolved to Windows filesystem paths via `/mnt/c/`
- **AND** extraction may be limited by WSL filesystem permissions
