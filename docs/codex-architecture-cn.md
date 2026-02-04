# Codex Token 使用情况获取 - 架构图

## 系统架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CodexBar 应用程序                           │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                      StatusItemController                       │ │
│  │                  (菜单栏图标和UI控制器)                          │ │
│  └───────────────────────────┬──────────────────────────────────┘ │
│                              │                                      │
│  ┌───────────────────────────▼──────────────────────────────────┐ │
│  │                        UsageStore                              │ │
│  │               (使用情况数据存储和刷新逻辑)                      │ │
│  └───────────────────────────┬──────────────────────────────────┘ │
│                              │                                      │
│  ┌───────────────────────────▼──────────────────────────────────┐ │
│  │                  CodexProviderDescriptor                       │ │
│  │                   (Provider 配置和策略)                         │ │
│  └───────────────────────────┬──────────────────────────────────┘ │
│                              │                                      │
│  ┌───────────────────────────▼──────────────────────────────────┐ │
│  │                  ProviderFetchPipeline                         │ │
│  │                     (数据获取管道)                              │ │
│  └─┬────────────────────┬───────────────────┬───────────────────┘ │
│    │                    │                   │                      │
└────┼────────────────────┼───────────────────┼──────────────────────┘
     │                    │                   │
     │                    │                   │
┌────▼──────┐      ┌──────▼────────┐   ┌─────▼──────────┐
│  Strategy │      │   Strategy    │   │   Strategy     │
│     1     │      │       2       │   │       3        │
└───────────┘      └───────────────┘   └────────────────┘
```

## 数据源策略详细架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ProviderFetchStrategy 协议                      │
│                                                                      │
│  - id: String                                                        │
│  - kind: ProviderFetchKind                                           │
│  + isAvailable() -> Bool                                             │
│  + fetch() -> ProviderFetchResult                                    │
│  + shouldFallback(error) -> Bool                                     │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
    ┌───────────▼─────────┐      ┌───────────▼─────────┐
    │ 实现类 (具体策略)    │      │ 实现类 (具体策略)    │
    └─────────────────────┘      └─────────────────────┘
```

### 策略 1: OAuth API

```
┌─────────────────────────────────────────────────────────────┐
│              CodexOAuthFetchStrategy                         │
├─────────────────────────────────────────────────────────────┤
│ id: "codex.oauth"                                            │
│ kind: .oauth                                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  isAvailable():                                              │
│    └─> 检查 ~/.codex/auth.json 是否存在                      │
│                                                              │
│  fetch():                                                    │
│    1. 读取凭证                                               │
│       CodexOAuthCredentialsStore.load()                      │
│       ├─> 解析 auth.json                                     │
│       └─> 返回 CodexOAuthCredentials                         │
│                                                              │
│    2. 检查并刷新 Token (如果需要)                            │
│       if credentials.needsRefresh:                           │
│         CodexTokenRefresher.refresh(credentials)             │
│         └─> POST https://auth0.openai.com/oauth/token        │
│                                                              │
│    3. 获取使用情况                                           │
│       CodexOAuthUsageFetcher.fetchUsage(token, accountId)    │
│       └─> GET https://chatgpt.com/backend-api/wham/usage     │
│                                                              │
│    4. 解析响应                                               │
│       JSONDecoder.decode(CodexUsageResponse.self)            │
│       ├─> planType: String                                   │
│       ├─> rateLimit:                                         │
│       │   ├─> primaryWindow (5h)                             │
│       │   └─> secondaryWindow (weekly)                       │
│       └─> credits:                                           │
│           ├─> balance: Double                                │
│           ├─> hasCredits: Bool                               │
│           └─> unlimited: Bool                                │
│                                                              │
│  shouldFallback():                                           │
│    └─> return true (如果 sourceMode == .auto)               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 策略 2: CLI RPC

```
┌─────────────────────────────────────────────────────────────┐
│              CodexCLIUsageStrategy                           │
├─────────────────────────────────────────────────────────────┤
│ id: "codex.cli"                                              │
│ kind: .cli                                                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  isAvailable():                                              │
│    └─> return true (CLI 始终可用)                            │
│                                                              │
│  fetch():                                                    │
│    1. 启动 RPC 服务器                                        │
│       SubprocessRunner.run(                                  │
│         binary: "codex",                                     │
│         args: ["-s", "read-only", "-a", "untrusted",         │
│               "app-server"]                                  │
│       )                                                      │
│                                                              │
│    2. 发送 JSON-RPC 请求                                     │
│       ┌──────────────────────────────────────┐              │
│       │ initialize                            │              │
│       │  { clientName, clientVersion }        │              │
│       └──────────────────────────────────────┘              │
│       ┌──────────────────────────────────────┐              │
│       │ account/read                          │              │
│       │  获取账户信息 (email, plan)            │              │
│       └──────────────────────────────────────┘              │
│       ┌──────────────────────────────────────┐              │
│       │ account/rateLimits/read               │              │
│       │  获取速率限制和积分                    │              │
│       └──────────────────────────────────────┘              │
│                                                              │
│    3. 解析 JSON 响应                                         │
│       {                                                      │
│         "primaryWindow": {                                   │
│           "usedPercent": 45,                                 │
│           "resetAt": 1234567890,                             │
│           "limitWindowSeconds": 18000                        │
│         },                                                   │
│         "secondaryWindow": { ... },                          │
│         "credits": {                                         │
│           "balance": 1234.56,                                │
│           "hasCredits": true                                 │
│         }                                                    │
│       }                                                      │
│                                                              │
│  shouldFallback():                                           │
│    └─> return false (不再回退)                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 策略 3: CLI PTY

```
┌─────────────────────────────────────────────────────────────┐
│              CodexCLIUsageStrategy (PTY fallback)            │
├─────────────────────────────────────────────────────────────┤
│ (当 RPC 失败时自动回退到 PTY 模式)                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  fetch():                                                    │
│    1. 创建伪终端 (PTY)                                       │
│       TTYCommandRunner()                                     │
│       └─> openpty() 创建 master/slave fd                     │
│                                                              │
│    2. 启动 Codex 进程                                        │
│       Process {                                              │
│         executable: "codex",                                 │
│         args: ["-s", "read-only", "-a", "untrusted"],        │
│         stdin: slave_fd,                                     │
│         stdout: slave_fd,                                    │
│         stderr: slave_fd                                     │
│       }                                                      │
│                                                              │
│    3. 发送命令                                               │
│       write(master_fd, "/status\n")                          │
│                                                              │
│    4. 读取输出                                               │
│       output = read(master_fd) // 持续读取直到超时           │
│       ┌──────────────────────────────────────┐              │
│       │ Credits: 1,234.56                     │              │
│       │ 5h limit: 45% used (resets in 2h)    │              │
│       │ Weekly limit: 23% used (resets in 4d)│              │
│       └──────────────────────────────────────┘              │
│                                                              │
│    5. 解析文本 (CodexStatusProbe.parse)                      │
│       - 提取 Credits: 后面的数字                             │
│       - 提取 5h limit 中的百分比和重置时间                   │
│       - 提取 Weekly limit 中的百分比和重置时间               │
│                                                              │
│       使用正则表达式:                                        │
│       - Credits: #"Credits:\s*([0-9][0-9.,]*)"#              │
│       - Percent: #"(\d+)%"#                                  │
│       - Reset: #"resets in (.+?)(?:\)|$)"#                   │
│                                                              │
│    6. 返回 CodexStatusSnapshot                               │
│       {                                                      │
│         credits: 1234.56,                                    │
│         fiveHourPercentLeft: 55,                             │
│         weeklyPercentLeft: 77,                               │
│         fiveHourResetDescription: "2h 30m",                  │
│         weeklyResetDescription: "4 days",                    │
│         rawText: "..."                                       │
│       }                                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 附加策略: Web Dashboard

```
┌─────────────────────────────────────────────────────────────┐
│            CodexWebDashboardStrategy                         │
├─────────────────────────────────────────────────────────────┤
│ id: "codex.web.dashboard"                                    │
│ kind: .webDashboard                                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  isAvailable():                                              │
│    └─> return settings.openAIWebAccessEnabled                │
│                                                              │
│  fetch():                                                    │
│    1. 导入浏览器 Cookie                                      │
│       OpenAIDashboardBrowserCookieImporter                   │
│       ├─> Safari: Cookies.binarycookies                      │
│       ├─> Chrome: SQLite cookies DB                          │
│       └─> Firefox: cookies.sqlite                            │
│                                                              │
│    2. 创建 WebKit 数据存储                                   │
│       WKWebsiteDataStore (per-account)                       │
│       └─> UUID from accountEmail                             │
│                                                              │
│    3. 加载页面                                               │
│       WKWebView.load(                                        │
│         "https://chatgpt.com/codex/settings/usage"           │
│       )                                                      │
│                                                              │
│    4. 注入 JavaScript 提取数据                               │
│       OpenAIDashboardScrapeScript.js                         │
│       ├─> 提取速率限制                                       │
│       ├─> 提取积分余额                                       │
│       ├─> 提取使用详情图表                                   │
│       └─> 提取积分历史表格                                   │
│                                                              │
│    5. 解析提取的数据                                         │
│       OpenAIDashboardParser.parse(html)                      │
│       └─> OpenAIDashboardSnapshot                            │
│           ├─> usageBreakdown (按模型)                        │
│           ├─> creditsHistory (历史记录)                      │
│           ├─> codeReviewRemaining (%)                        │
│           └─> creditsPurchaseURL                             │
│                                                              │
│    6. 缓存 Cookie                                            │
│       KeychainCacheStore.save(                               │
│         account: "cookie.codex",                             │
│         service: "com.steipete.codexbar.cache"               │
│       )                                                      │
│                                                              │
│  shouldFallback():                                           │
│    └─> return true (如果 sourceMode == .auto)               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 会话管理架构

```
┌─────────────────────────────────────────────────────────────┐
│                    CodexCLISession (Actor)                   │
├─────────────────────────────────────────────────────────────┤
│                   持久化 CLI 会话管理器                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  属性:                                                       │
│    - process: Process?           // 当前运行的进程           │
│    - primaryFD: Int32            // PTY master fd            │
│    - primaryHandle: FileHandle?  // master 文件句柄          │
│    - secondaryHandle: FileHandle? // slave 文件句柄          │
│    - processGroup: pid_t?        // 进程组 ID                │
│    - binaryPath: String?         // codex 二进制路径         │
│    - startedAt: Date?            // 启动时间                 │
│                                                              │
│  方法:                                                       │
│    + captureStatus(binary, timeout) -> String                │
│      ├─> ensureStarted() // 确保会话已启动                  │
│      ├─> send("/status\n") // 发送命令                      │
│      ├─> readChunk() // 读取输出                            │
│      ├─> 检测状态标记 ("Credits:", "5h limit", etc.)        │
│      ├─> 处理更新提示 (跳过)                                │
│      └─> 返回捕获的文本                                     │
│                                                              │
│    + reset() // 清理并关闭会话                              │
│      ├─> send("/exit\n")                                     │
│      ├─> terminate()                                         │
│      ├─> kill(-pgid, SIGTERM)                                │
│      └─> cleanup file handles                                │
│                                                              │
│    - ensureStarted() // 启动或重用会话                      │
│      ├─> 检查现有会话是否可用                               │
│      ├─> openpty() // 创建新 PTY                            │
│      ├─> Process.run() // 启动进程                          │
│      └─> setpgid() // 设置进程组                            │
│                                                              │
│    - readChunk() -> Data // 非阻塞读取                      │
│      └─> read(primaryFD) 循环直到 EAGAIN                     │
│                                                              │
│    - send(text) // 发送文本到 PTY                           │
│      └─> write(primaryFD, text.utf8)                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 数据模型架构

```
┌─────────────────────────────────────────────────────────────┐
│                       数据模型层次结构                        │
└─────────────────────────────────────────────────────────────┘

CodexUsageResponse (OAuth API 响应)
├─ planType: PlanType?
│  └─ enum: guest, free, plus, pro, team, enterprise, etc.
├─ rateLimit: RateLimitDetails?
│  ├─ primaryWindow: WindowSnapshot?
│  │  ├─ usedPercent: Int
│  │  ├─ resetAt: Int (timestamp)
│  │  └─ limitWindowSeconds: Int
│  └─ secondaryWindow: WindowSnapshot?
└─ credits: CreditDetails?
   ├─ balance: Double?
   ├─ hasCredits: Bool
   └─ unlimited: Bool

                ↓ 转换为 ↓

UsageSnapshot (统一数据结构)
├─ primary: RateWindow
│  ├─ usedPercent: Double
│  ├─ windowMinutes: Int?
│  ├─ resetsAt: Date?
│  └─ resetDescription: String?
├─ secondary: RateWindow?
├─ tertiary: RateWindow?
├─ updatedAt: Date
└─ identity: ProviderIdentitySnapshot
   ├─ providerID: UsageProvider (.codex)
   ├─ accountEmail: String?
   ├─ accountOrganization: String?
   └─ loginMethod: String? (plan type)

                ↓ 显示在 ↓

MenuBarDisplayText (UI 显示)
├─ primary text: "45% · 2h"
├─ secondary text: "23% · 4d"
└─ icon: colored/dimmed based on usage
```

## 错误处理流程

```
┌─────────────────────────────────────────────────────────────┐
│                      错误处理机制                             │
└─────────────────────────────────────────────────────────────┘

Strategy.fetch()
    ↓
┌───▼──────────────────────────────┐
│ 执行数据获取                       │
└────┬────────────────────┬────────┘
     │ 成功               │ 失败
     ↓                    ↓
┌────▼────────┐    ┌──────▼─────────────────────┐
│ 返回结果     │    │ 抛出错误                    │
│             │    │ - NetworkError              │
└─────────────┘    │ - Unauthorized              │
                  │ - ParseFailed               │
                  │ - TimedOut                  │
                  │ - NotInstalled              │
                  └──────┬──────────────────────┘
                         ↓
                  Strategy.shouldFallback(error)
                         ↓
                    ┌────▼────┐
                    │ true?   │
                    └─┬───┬───┘
              true    │   │    false
                    ┌─▼─┐ └─▼────────┐
                    │尝试 │ 报告错误给 │
                    │下一个│   用户    │
                    │策略  │           │
                    └─────┘ └──────────┘

特殊处理:
├─ CodexStatusProbe 解析失败
│  └─> 自动重试一次 (更大终端尺寸)
│
├─ OAuth token 过期
│  └─> 自动刷新 token
│
├─ CLI 会话崩溃
│  └─> CodexCLISession.reset() 重新启动
│
└─ 更新提示阻塞
   └─> 自动检测并跳过更新对话框
```

## 缓存机制架构

```
┌─────────────────────────────────────────────────────────────┐
│                         缓存层                                │
└─────────────────────────────────────────────────────────────┘

1. OAuth Token 缓存
   位置: ~/.codex/auth.json
   内容: { tokens, last_refresh }
   刷新: 8天后自动刷新

2. Browser Cookie 缓存
   位置: macOS Keychain
   账户: cookie.codex
   服务: com.steipete.codexbar.cache
   内容: { cookieHeader, sourceLabel, storedAt }
   刷新: 根据需要重新导入

3. Web Dashboard 缓存
   位置: WKWebsiteDataStore (per-account)
   UUID: 从 accountEmail 生成
   内容: WebKit cookies + localStorage
   刷新: 根据 cookie 过期时间

4. Cost Usage 缓存
   位置: ~/Library/Caches/CodexBar/cost-usage/codex-v1.json
   内容: { sessions, costs, lastScan }
   刷新: 最小 60 秒间隔
   窗口: 最近 30 天

5. 使用情况内存缓存
   位置: UsageStore (in-memory)
   内容: 最新的 UsageSnapshot
   刷新: 10 秒定时器（默认）
```

## 并发和线程模型

```
┌─────────────────────────────────────────────────────────────┐
│                      线程和并发模型                           │
└─────────────────────────────────────────────────────────────┘

@MainActor
├─ StatusItemController
├─ UsageStore
├─ SettingsStore
└─ UI Components

async/await
├─ UsageFetcher
├─ CodexOAuthUsageFetcher
├─ CodexStatusProbe
└─ OpenAIDashboardFetcher

Actor (isolated)
├─ CodexCLISession
│  └─> 保护共享的 PTY 状态
└─ (其他会话管理器)

Background Queue
├─ File I/O (session logs)
├─ Cost calculation
└─ Cookie import

并发策略:
├─ 主数据源: 顺序执行 (带回退)
└─ Web Dashboard: 并行执行 (不阻塞主路径)
```

这些架构图展示了 CodexBar 如何通过分层、模块化的设计来实现可靠的 Codex token 使用情况监控。
