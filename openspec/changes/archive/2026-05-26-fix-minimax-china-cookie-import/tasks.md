## 1. MiniMaxRegion cookie_domain 方法

- [x] 1.1 将 `MiniMaxRegion` 从 `enum` 改为 `pub enum` (`rust/src/providers/minimax/mod.rs:74`)
- [x] 1.2 新增 `MiniMaxRegion::cookie_domain()` 方法：Global → `"platform.minimax.io"`，ChinaMainland → `"platform.minimaxi.com"`
- [x] 1.3 新增 `impl From<&str> for MiniMaxRegion`：`"china"` → ChinaMainland，其他 → Global（如未存在）

## 2. import_browser_cookies 命令改造

- [x] 2.1 在 `import_browser_cookies` 签名中新增 `region: Option<String>` 参数 (`apps/desktop-tauri/src-tauri/src/commands/browser_import.rs:41`)
- [x] 2.2 当 `provider_id == "minimax"` 且 `region == Some("china")` 时，覆盖 cookie domain 为 `MiniMaxRegion::ChinaMainland.cookie_domain()`
- [x] 2.3 添加 `use codexbar::providers::minimax::MiniMaxRegion` 引用到 browser_import 模块

## 3. 前端传参

- [x] 3.1 找到 MiniMax 的 Import from Browser 前端组件，定位 `invoke("import_browser_cookies", ...)` 调用点
- [x] 3.2 新增 `region` 字段到 invoke 参数中，从 MiniMax provider 的 region state 获取（`"global"` / `"china"`）
- [x] 3.3 确认 TypeScript 类型定义同步更新（如存在 `ImportCookiesArgs` 接口）

## 4. 测试与验证

- [x] 4.1 新增 `MiniMaxRegion::cookie_domain()` 的单元测试（Global / ChinaMainland / unknown 三个场景）
- [x] 4.2 运行 `cargo test -p codexbar` 和 `cargo test --manifest-path apps/desktop-tauri/src-tauri/Cargo.toml`
- [x] 4.3 运行 `cargo fmt --all && cargo clippy --all-targets -- -D warnings`

## 5. Chromium ABE 与 Cookie 诊断

- [x] 5.1 扩展 cookie host_key 查询，覆盖父域 cookie（如 `.minimaxi.com`）
- [x] 5.2 在 cookie import 失败时输出安全诊断（profile/DB/host_key 统计，不泄漏 cookie 值）
- [x] 5.3 针对 Chromium App-Bound Encryption 提供准确错误提示，不再建议已失败的 Edge 路径

## 6. MiniMax API Key token_plan/remains 路径

- [x] 6.1 将 MiniMax 加入 API Keys 设置列表
- [x] 6.2 在 MiniMax provider 中新增 `fetch_token_plan_remains(api_key, region)`
- [x] 6.3 在 `fetch_usage` 中优先使用 `ctx.api_key` 调用 `/v1/token_plan/remains`
- [x] 6.4 新增 `parse_token_plan_response()`，支持 used/total/remaining 多字段别名
- [x] 6.5 正确处理 `current_interval_usage_count` 为“剩余”而非“已用”
- [x] 6.6 新增 token_plan/remains 解析单元测试
