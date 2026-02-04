# Codex Token 使用情况获取实现详解

本文档详细说明了 CodexBar 项目如何获取 Codex 的 token 使用情况和剩余情况。

## 概述

CodexBar 使用多种数据源来获取 Codex 的使用情况，并实现了智能的回退机制以确保可靠性。

## 数据源优先级

### 应用程序模式（App）
当在 macOS 应用中运行时，默认的数据源优先级为：

1. **OAuth API**（首选）
   - 如果 `~/.codex/auth.json` 文件存在且包含有效凭证
   
2. **CLI RPC**
   - 通过本地 Codex CLI 的 JSON-RPC 接口
   
3. **CLI PTY 回退**
   - 如果 RPC 失败，通过伪终端运行 `codex /status` 命令

4. **OpenAI Web 仪表板**（可选附加功能）
   - 需要在设置中启用 "OpenAI web extras"
   - 提供额外的使用详情和历史数据

### 命令行模式（CLI）
当使用 `codexbar` CLI 工具时，默认优先级为：

1. **OpenAI Web 仪表板**
2. **CLI RPC**
3. **CLI PTY 回退**

## 实现细节

### 1. OAuth API 方式

**文件位置**: `Sources/CodexBarCore/Providers/Codex/CodexOAuth/CodexOAuthUsageFetcher.swift`

**工作原理**:
```swift
// 1. 从 ~/.codex/auth.json 读取 OAuth 凭证
let credentials = try CodexOAuthCredentialsStore.load()

// 2. 检查 token 是否需要刷新（超过8天）
if credentials.needsRefresh {
    credentials = try await CodexTokenRefresher.refresh(credentials)
    try CodexOAuthCredentialsStore.save(credentials)
}

// 3. 调用 ChatGPT API 获取使用情况
// 默认 URL: https://chatgpt.com/backend-api/wham/usage
// 或者: https://chatgpt.com/backend-api/api/codex/usage
let usage = try await CodexOAuthUsageFetcher.fetchUsage(
    accessToken: credentials.accessToken,
    accountId: credentials.accountId
)
```

**返回的数据结构**:
- `planType`: 计划类型（free, plus, pro, team, business, enterprise 等）
- `rateLimit`: 速率限制详情
  - `primaryWindow`: 主要时间窗口（5小时限制）
    - `usedPercent`: 已使用百分比
    - `resetAt`: 重置时间戳
  - `secondaryWindow`: 次要时间窗口（每周限制）
- `credits`: 积分详情
  - `balance`: 剩余积分余额
  - `hasCredits`: 是否有积分
  - `unlimited`: 是否无限制

**凭证文件格式** (`~/.codex/auth.json`):
```json
{
  "tokens": {
    "access_token": "...",
    "refresh_token": "...",
    "id_token": "...",
    "account_id": "..."
  },
  "last_refresh": "2024-01-01T00:00:00.000Z"
}
```

### 2. CLI RPC 方式

**文件位置**: `Sources/CodexBarCore/UsageFetcher.swift`

**工作原理**:
```swift
// 1. 启动 Codex CLI RPC 服务器
// 命令: codex -s read-only -a untrusted app-server

// 2. 通过 JSON-RPC 发送请求
// - initialize: 初始化连接
// - account/read: 读取账户信息
// - account/rateLimits/read: 读取速率限制

// 3. 解析返回的 JSON 响应
// 获取:
// - 使用率百分比
// - 重置时间戳
// - 账户邮箱
// - 计划类型
// - 积分余额
```

**优势**:
- 速度快（本地通信）
- 数据准确（直接从 Codex CLI 获取）
- 支持持久会话（Debug 模式下可保持连接）

### 3. CLI PTY 回退方式

**文件位置**: `Sources/CodexBarCore/Providers/Codex/CodexStatusProbe.swift`

**工作原理**:
```swift
// 1. 在伪终端中启动 Codex
let runner = TTYCommandRunner()
let script = "/status\n"
let result = try runner.run(
    binary: "codex",
    send: script,
    options: .init(
        rows: 60,
        cols: 200,
        timeout: 18.0,
        extraArgs: ["-s", "read-only", "-a", "untrusted"]
    )
)

// 2. 解析终端输出文本
let text = result.text
let snapshot = try CodexStatusProbe.parse(text: text)
```

**解析规则**:
- 从输出中查找 `Credits:` 行，提取积分数字
- 查找 `5h limit` 行，提取已使用百分比和重置时间
- 查找 `Weekly limit` 行，提取已使用百分比和重置时间

**示例输出解析**:
```
Credits: 1,234.56
5h limit: 45% used (resets in 2h 30m)
Weekly limit: 23% used (resets in 4 days)
```

**会话管理**:
- `CodexCLISession`: Actor 模式实现的持久会话管理器
- 支持保持连接以避免重复启动开销
- 自动处理更新提示（检测并跳过 "Update available" 对话框）

### 4. OpenAI Web 仪表板方式

**文件位置**: `Sources/CodexBarCore/Providers/Codex/CodexWebDashboardStrategy.swift`

**工作原理**:
```swift
// 1. 导入浏览器 Cookie
// 支持的浏览器:
// - Safari: ~/Library/Cookies/Cookies.binarycookies
// - Chrome: ~/Library/Application Support/Google/Chrome/*/Cookies
// - Firefox: ~/Library/Application Support/Firefox/Profiles/*/cookies.sqlite

// 2. 使用 WKWebView 加载 ChatGPT 仪表板
// URL: https://chatgpt.com/codex/settings/usage

// 3. 执行 JavaScript 脚本提取数据
// - 从页面 HTML 中提取使用率信息
// - 解析使用详情图表数据
// - 获取积分历史记录

// 4. 缓存 Cookie 到 Keychain
// 账户: cookie.codex
// 服务: com.steipete.codexbar.cache
```

**提供的额外数据**:
- 使用详情图表（按模型分类）
- 积分使用历史
- 代码审查剩余百分比
- 购买积分的 URL

**Cookie 导入模式**:
- **自动模式**: 自动从浏览器导入 Cookie
- **手动模式**: 用户手动粘贴 Cookie header

### 5. 本地成本使用扫描器

**文件位置**: `Sources/CodexBarCore/CostUsageFetcher.swift`

**工作原理**:
```swift
// 1. 扫描本地会话文件
// 路径:
// - ~/.codex/sessions/YYYY/MM/DD/*.jsonl
// - ~/.codex/archived_sessions/*.jsonl

// 2. 解析每行 JSONL 数据
// 提取:
// - event_msg: token 计数
// - turn_context: 模型标识
// - 输入/缓存/输出 token 数量

// 3. 计算成本
// 根据模型定价计算每次调用的成本

// 4. 缓存结果
// 位置: ~/Library/Caches/CodexBar/cost-usage/codex-v1.json
// 时间窗口: 最近 30 天
```

**扫描的数据**:
- Token 使用量（按模型分类）
- 预估成本
- 使用趋势

## 架构设计

### 策略模式实现

项目使用策略模式（Strategy Pattern）来实现多数据源的灵活切换：

```swift
protocol ProviderFetchStrategy {
    var id: String { get }
    var kind: ProviderFetchKind { get }
    
    func isAvailable(_ context: ProviderFetchContext) async -> Bool
    func fetch(_ context: ProviderFetchContext) async throws -> ProviderFetchResult
    func shouldFallback(on error: Error, context: ProviderFetchContext) -> Bool
}

// 实现的策略:
// 1. CodexOAuthFetchStrategy
// 2. CodexCLIUsageStrategy
// 3. CodexWebDashboardStrategy
```

### 数据流程

```
用户请求更新
    ↓
ProviderFetchPipeline
    ↓
按优先级尝试策略
    ↓
策略 1: OAuth API
    ├─ 成功 → 返回结果
    └─ 失败 → 下一个策略
    ↓
策略 2: CLI RPC
    ├─ 成功 → 返回结果
    └─ 失败 → 下一个策略
    ↓
策略 3: CLI PTY
    ├─ 成功 → 返回结果
    └─ 失败 → 报告错误
    ↓
（可选）并行加载 Web 仪表板附加功能
    ↓
合并所有数据源
    ↓
更新 UI 显示
```

## 关键类和文件

### 核心文件
1. **`CodexProviderDescriptor.swift`**
   - 定义 Codex provider 的元数据和策略选择逻辑

2. **`CodexOAuthUsageFetcher.swift`**
   - OAuth API 调用实现

3. **`CodexStatusProbe.swift`**
   - CLI PTY 方式的实现和输出解析

4. **`CodexCLISession.swift`**
   - 持久化 CLI 会话管理

5. **`CodexWebDashboardStrategy.swift`**
   - Web 仪表板数据提取

6. **`CodexOAuthCredentials.swift`**
   - OAuth 凭证的读取、保存和刷新

### 数据模型
- **`CodexUsageResponse`**: OAuth API 响应的数据结构
- **`CodexStatusSnapshot`**: CLI 状态快照
- **`UsageSnapshot`**: 统一的使用情况数据结构
- **`CreditsSnapshot`**: 积分信息快照

## 用户配置

### 设置路径
**应用程序**: Settings → Providers → Codex

### 可配置选项
1. **Usage source（使用情况来源）**
   - Auto: 自动选择最佳来源
   - OAuth: 仅使用 OAuth API
   - CLI: 仅使用 CLI

2. **OpenAI web extras（OpenAI Web 附加功能）**
   - 开关: 启用/禁用 Web 仪表板附加功能

3. **OpenAI cookies（OpenAI Cookie）**
   - Automatic: 自动从浏览器导入
   - Manual: 手动粘贴 Cookie header
   - Off: 禁用

## 错误处理

### 常见错误和处理
1. **`CodexStatusProbeError.codexNotInstalled`**
   - 错误: Codex CLI 未安装
   - 解决: 运行 `npm i -g @openai/codex` 安装

2. **`CodexStatusProbeError.updateRequired`**
   - 错误: Codex CLI 版本过旧
   - 解决: 运行 `bun install -g @openai/codex` 更新

3. **`CodexOAuthFetchError.unauthorized`**
   - 错误: OAuth token 过期或无效
   - 解决: 运行 `codex` 重新认证

4. **`CodexStatusProbeError.parseFailed`**
   - 错误: 解析 CLI 输出失败
   - 处理: 自动重试一次，使用更大的终端尺寸

### 回退机制
所有策略都实现了 `shouldFallback` 方法，当 `sourceMode == .auto` 时会自动尝试下一个策略。

## 性能优化

1. **缓存机制**
   - Cookie 缓存到 Keychain
   - Web 仪表板数据缓存
   - 成本使用数据缓存（60秒最小刷新间隔）

2. **并行加载**
   - Web 仪表板附加功能与主数据源并行加载
   - 不阻塞主要使用情况的显示

3. **持久会话**
   - Debug 模式下可保持 CLI 会话以减少启动开销
   - 使用 Actor 模式确保线程安全

4. **智能刷新**
   - OAuth token 只在需要时刷新（超过8天）
   - 本地文件扫描有最小刷新间隔

## 调试技巧

### 启用详细日志
在 Debug 菜单中启用:
- "Keep CLI sessions alive": 保持 CLI 会话以便调试
- "Dump OpenAI HTML": 保存 Web 仪表板的 HTML 用于调试解析问题

### 查看日志
应用日志位置: `~/Library/Logs/CodexBar/`

### 手动测试命令
```bash
# 测试 OAuth API
codex

# 测试 CLI RPC
codex -s read-only -a untrusted app-server

# 测试 CLI PTY
codex -s read-only -a untrusted
/status

# 查看本地会话文件
ls -la ~/.codex/sessions/
```

## 总结

CodexBar 通过以下方式实现了强大且可靠的 Codex token 使用情况监控：

1. **多数据源支持**: OAuth API、CLI RPC、CLI PTY、Web 仪表板
2. **智能回退**: 自动在数据源之间切换以确保可靠性
3. **丰富的数据**: 不仅提供基本使用率，还包括积分、历史和成本信息
4. **优秀的用户体验**: 自动选择最佳数据源，无需用户干预
5. **完善的错误处理**: 清晰的错误消息和自动恢复机制

这种设计使得 CodexBar 能够适应各种使用场景，从个人开发者到企业团队，都能获得准确的 token 使用情况监控。
