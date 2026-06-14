# Win-CodexBar

[English README](./README.md)

Win-CodexBar 是一个 Windows 系统托盘应用，让你无需打开一堆仪表盘就能随时掌握 AI 编程工具的用量。它将 [CodexBar](https://github.com/steipete/CodexBar) 的理念移植到 **Tauri + React** 桌面壳层，底层复用共享的 **Rust** 服务商逻辑。

<table>
  <tr>
    <td width="36%" align="center">
      <img src="extra-docs/images/tray-panel.png" alt="Win-CodexBar 托盘面板 — 服务商用量卡片"/>
    </td>
    <td width="64%" align="center">
      <img src="extra-docs/images/settings-providers.png" alt="Win-CodexBar 设置 — 服务商页面"/>
    </td>
  </tr>
</table>

## 功能特性

- **49 个服务商** — 包括 Codex、Claude、Copilot、OpenRouter、Cursor、Gemini、DeepSeek、MiniMax、Kiro、Antigravity、Groq 等。
- **托盘优先工作流** — 紧凑的服务商网格、用量卡片、刷新操作、设置快捷入口和退出控制。
- **服务商设置** — 来源选择、凭据管理、Cookie 导入、令牌账户、API Key、区域选择和托盘显示偏好。
- **Windows 凭据保护** — 应用管理的 API Key、手动 Cookie 和令牌账户使用用户级 DPAPI 加密存储。
- **浏览器 Cookie 导入** — 支持 Chrome、Edge、Brave 和 Firefox，按服务商可选启用。
- **本地 CLI** — 支持脚本化查询用量、成本、配置、诊断和本机回环集成。
- **安装包 + 便携版** — 包含 WebView2 运行时引导、VC++ 运行库引导和 SHA-256 校验文件。

## 本 Fork 的不同之处

本 Fork 在上游 Windows 移植的基础上增加了服务商专属改进，其中最大的变化集中在 **MiniMax 中国大陆区域** 支持：

### MiniMax — CN 区域 API 用量

- **API 端点切换至 `/v1/token_plan/remains`** — 不再需要 `group_id`，仅使用 Bearer API Key 进行认证。该端点直接返回 Token 套餐余额，比旧的账单历史方式更可靠。
- **区域感知路由** — 在每一层将 Global（`platform.minimax.io`）和中国大陆（`platform.minimaxi.com`）分离：
  - API base URL、浏览器 Cookie 域名、账单历史端点和控制台 URL 均按区域独立配置
  - CN 账单数据从 `www.minimaxi.com/account/amount` 获取（与 Web 控制台主机不同）
- **CN 浏览器 Cookie 导入** — 现在可以为 `platform.minimaxi.com` 上的中国大陆账号导入 Cookie，并正确提取父域名 host-key
- **修复区域域名映射** — 旧版本中 Global 和 CN 域名是反的；本 Fork 修正了映射，确保每个区域命中正确的端点

### Z.ai / BigModel

- **中国大陆路由** — Z.ai 服务商根据区域设置自动将 CN 用户路由到 BigModel API
- **响应解析加固** — 添加了 `usage` 字段别名和 `percentage` 字段支持，以适配 BigModel API 响应格式（与上游 Z.ai schema 不同）

### Claude

- **OAuth 凭据读取修复** — 解决了 Claude OAuth 凭据在 Windows 上可能加载失败的问题
- **Windows 托盘启动修复** — 修复了首次启动时影响托盘图标的竞态条件

### 浏览器 Cookie 改进

- **父域名 host-key** — Cookie 提取现在包含父域名条目，修复了浏览器将 Cookie 存储在 `minimaxi.com` 而非 `platform.minimaxi.com` 下的情况
- **导入诊断** — 增加了浏览器 Cookie 导入的详细诊断信息，便于排查提取问题

### UI/UX 打磨

- 修复 Windows 上浮动条透明度问题
- 设置页凭据按钮在窄面板上不再换行
- 为较小尺寸的 Windows 屏幕优化紧凑布局

## 安装

使用 Windows Package Manager 安装：

```powershell
winget install Finesssee.Win-CodexBar
```

或从 [GitHub Releases](https://github.com/Finesssee/Win-CodexBar/releases) 下载最新的安装包/便携版。

- 安装包：`CodexBar-<version>-Setup.exe`
- 便携版：`CodexBar-<version>-portable.exe`
- 校验和：每个发布版本都包含 `.sha256` 文件

Winget 分发已通过 [microsoft/winget-pkgs](https://github.com/microsoft/winget-pkgs/tree/master/manifests/f/Finesssee/Win-CodexBar) 审核。新版本可能需要一点时间才会出现在 Winget 中，因为每次 Winget 更新都固定到特定的发布 URL 和安装包哈希。

## 首次运行

1. 从开始菜单或便携版可执行文件启动 **CodexBar**。
2. 点击托盘图标打开用量面板。
3. 打开 **Settings -> Providers**。
4. 启用你使用的服务商。
5. 添加匹配的凭据类型：OAuth/设备登录、API Key、浏览器 Cookie、本地 CLI 登录或令牌账户。

对于 Claude，建议优先使用浏览器 Cookie/sessionKey，因为它们与 Claude 设置页的用量一致。OAuth 和 CLI 作为后备选项保留。对于基于 CLI 的服务商（如 Codex 和 Gemini），请先登录服务商 CLI。

## 最新版本

**v0.32.4** 修复了 OpenRouter credits 获取问题：使用标准 `/api/v1/credits` 端点替代失效的 `/api/v1/auth/credits` 路径。同时将 OpenRouter key 查询对齐到 `/api/v1/key`，并添加了两条 URL 路径的回归测试。

完整更新历史请查看 [CHANGELOG.md](CHANGELOG.md)。

## 支持的服务商

<details>
<summary>服务商矩阵</summary>

| 服务商 | 认证方式 | 跟踪内容 |
|--------|----------|----------|
| Codex | OAuth / CLI | 会话、周用量、Credits |
| Claude | Cookies / OAuth 后备 / CLI 后备 | 会话（5h）、周用量 |
| Cursor | Cookies | 套餐、用量、账单 |
| Factory | Cookies | 用量 |
| Gemini | gcloud OAuth | 配额 |
| Copilot | GitHub Device Flow / gh CLI / 旧版 token | 套餐用量、Chat |
| Antigravity | 本地 LSP | 用量、按模型配额 |
| z.ai | API Token | 配额 |
| MiniMax | API / Cookies | 用量、账单汇总 |
| Kiro | Cookies / CLI | 月度 Credits、超额用量 |
| Vertex AI | gcloud OAuth | 成本 |
| Augment | Cookies | Credits |
| OpenCode | 本地配置 | 用量 |
| Kimi | Cookies | 5h 速率、周用量 |
| Kimi K2 | API Key | Credits |
| Amp | Cookies | 用量 |
| Warp | 本地配置 | 用量 |
| Ollama | Cookies / API Key | 用量、云端模型、Pace 窗口 |
| Azure OpenAI | API Key | 部署 |
| T3 Chat | Cookies / cURL | 基础、超额 |
| OpenRouter | API Key | Credits |
| JetBrains AI | 本地配置 | 用量 |
| Alibaba | Cookies | 用量 |
| Alibaba Token Plan | Cookies | Token 套餐 Credits、重置日期 |
| NanoGPT | API Key | Credits |
| Infini | API Key | 会话、周用量、配额 |
| Perplexity | Cookies | Credits、套餐 |
| Abacus AI | Cookies | Credits |
| Mistral | Cookies | 账单、用量 |
| OpenCode Go | Cookies | 用量、Zen 余额 |
| Kilo | API Key / CLI | 用量 |
| Codebuff | API Key / 本地配置 | Credits、周用量 |
| DeepSeek | API Key | 余额、用量摘要、成本 |
| Windsurf | 本地缓存 | 日用量、周用量 |
| Manus | Cookies | Credits、刷新 Credits |
| 小米 MiMo | Cookies | 余额、Token 套餐 |
| Doubao | API Key | 请求限制 |
| Command Code | Cookies | 月度 Credits、已购 Credits |
| Crof | API Key | Credits、请求配额 |
| StepFun | Oasis Token | 5h、周用量、Token 刷新 |
| Venice | API Key | USD / DIEM 余额 |
| OpenAI | Admin API / API Key | 用量、请求数、项目级成本、Credit 余额 |
| Grok | Cookies / auth.json | 账单 |
| ElevenLabs | API Key | 订阅 Credits、Voice Slots |
| Deepgram | API Key | 项目用量 |
| Groq | API Key | Enterprise Metrics |
| LLM Proxy | API Key | 配额统计 |

</details>

## 从源码构建

```powershell
# 前置要求：Node.js + pnpm — Rust 和 MinGW 将由脚本在需要时自动安装
git clone https://github.com/Finesssee/Win-CodexBar.git
cd Win-CodexBar
.\dev.ps1
```

常用开发参数：

```powershell
.\dev.ps1 -Release      # 优化构建
.\dev.ps1 -SkipBuild    # 跳过构建，直接启动上次构建
```

CLI 示例：

```bash
codexbar usage -p claude
codexbar usage -p all
codexbar cost -p codex
```

## 发布构建

在 Windows 上做本地发布构建时，使用缓存版发布构建脚本：

```powershell
.\scripts\windows-release-build.ps1 -Ref v0.32.4 -SmokeInstall
```

脚本会构建真实 Tauri release 二进制，校验已签名的安装包依赖，用 Inno Setup 打包，输出安装包/便携版资产，生成 SHA-256 校验文件，并可执行静默安装/卸载冒烟测试。

更多发布自动化说明请查看 [docs/release/ci-cd.md](docs/release/ci-cd.md)。

## 隐私

- **默认本地处理**：服务商数据从已知的本地路径或你配置的服务商 API 读取。
- **按需 Cookie**：浏览器 Cookie 提取仅在你启用的服务商上运行。
- **受保护的凭据**：API Key、手动 Cookie 和令牌账户使用安全文件层；Windows 上会优先使用用户级 DPAPI。
- **安全诊断**：诊断只展示服务商/来源/状态等元数据，绝不展示原始 Cookie、API Key、Bearer Token 或 OAuth 值。
- **已验证更新**：安装包下载需要 GitHub SHA-256 摘要，并在应用前再次校验。

## 更多文档

| 主题 | 链接 |
|------|------|
| 从源码构建 | [extra-docs/BUILDING.md](extra-docs/BUILDING.md) |
| WSL 设置与认证 | [extra-docs/WSL.md](extra-docs/WSL.md) |
| 浏览器 Cookie 详解 | [extra-docs/COOKIES.md](extra-docs/COOKIES.md) |

## 致谢

- 原版 macOS 应用：[steipete/CodexBar](https://github.com/steipete/CodexBar)，作者 Peter Steinberger
- 灵感来源：[ccusage](https://github.com/ryoppippi/ccusage)，用于成本跟踪

## 许可证

MIT — 与原版 CodexBar 保持一致
