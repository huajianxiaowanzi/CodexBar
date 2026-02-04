# Codex Token 使用情况获取 - 快速指南

这是一个简化版的指南，快速说明 CodexBar 如何获取 Codex 的 token 使用情况。

## 三种主要方式

### 1. OAuth API（推荐，速度最快）

**原理**: 读取本地的 OAuth 凭证文件，调用 ChatGPT API

**实现位置**: `Sources/CodexBarCore/Providers/Codex/CodexOAuth/CodexOAuthUsageFetcher.swift`

**步骤**:
1. 读取 `~/.codex/auth.json` 文件
2. 如果 token 过期（超过8天），自动刷新
3. 调用 `https://chatgpt.com/backend-api/wham/usage`
4. 解析返回的 JSON 数据

**获取的信息**:
- 5小时限制使用百分比和重置时间
- 每周限制使用百分比和重置时间
- 剩余积分
- 账户计划类型

**代码示例**:
```swift
// 加载凭证
let credentials = try CodexOAuthCredentialsStore.load()

// 获取使用情况
let usage = try await CodexOAuthUsageFetcher.fetchUsage(
    accessToken: credentials.accessToken,
    accountId: credentials.accountId
)

// usage 包含:
// - rateLimit.primaryWindow (5小时窗口)
// - rateLimit.secondaryWindow (每周窗口)  
// - credits.balance (剩余积分)
```

### 2. CLI RPC（备选方案）

**原理**: 启动本地 Codex CLI 的 RPC 服务器，通过 JSON-RPC 协议通信

**实现位置**: `Sources/CodexBarCore/UsageFetcher.swift`

**步骤**:
1. 运行命令: `codex -s read-only -a untrusted app-server`
2. 发送 JSON-RPC 请求: `account/rateLimits/read`
3. 解析 JSON 响应

**优点**:
- 速度快（本地通信）
- 数据准确（直接从 Codex CLI 获取）

### 3. CLI PTY（最后备选）

**原理**: 在虚拟终端中运行 Codex，发送 `/status` 命令，解析文本输出

**实现位置**: `Sources/CodexBarCore/Providers/Codex/CodexStatusProbe.swift`

**步骤**:
1. 在伪终端(PTY)中启动 `codex`
2. 发送 `/status` 命令
3. 解析终端输出的文本

**解析示例**:
```
输入: /status

输出:
Credits: 1,234.56
5h limit: 45% used (resets in 2h 30m)
Weekly limit: 23% used (resets in 4 days)
```

**代码会提取**:
- Credits 后面的数字 → 剩余积分
- 5h limit 行中的百分比 → 5小时窗口使用率
- Weekly limit 行中的百分比 → 每周使用率

## 附加功能：OpenAI Web 仪表板

**原理**: 使用 WebKit 加载 ChatGPT 网页，提取额外信息

**实现位置**: `Sources/CodexBarCore/Providers/Codex/CodexWebDashboardStrategy.swift`

**提供**:
- 使用详情图表（按模型分类）
- 积分使用历史
- 代码审查剩余次数

**需要**:
- 在设置中启用 "OpenAI web extras"
- 配置浏览器 Cookie（自动或手动）

## 自动回退机制

当设置为 "Auto" 模式时，按以下顺序尝试：

```
应用程序模式:
OAuth API → CLI RPC → CLI PTY

命令行模式:
Web 仪表板 → CLI RPC → CLI PTY
```

如果某个方式失败，自动尝试下一个。

## 关键代码文件

| 文件 | 功能 |
|------|------|
| `CodexOAuthUsageFetcher.swift` | OAuth API 调用 |
| `CodexStatusProbe.swift` | CLI PTY 解析 |
| `CodexCLISession.swift` | CLI 会话管理 |
| `CodexWebDashboardStrategy.swift` | Web 数据提取 |
| `CodexProviderDescriptor.swift` | 策略选择逻辑 |

## 数据流程图

```
用户界面
    ↓
UsageStore.refresh()
    ↓
ProviderFetchPipeline
    ↓
┌─────────────────────────────────┐
│ 1. 尝试 OAuth API               │
│    ├─ 读取 ~/.codex/auth.json   │
│    ├─ 刷新 token (如果需要)      │
│    ├─ 调用 ChatGPT API          │
│    └─ 解析 JSON                 │
└─────────────────────────────────┘
    ↓ (如果失败)
┌─────────────────────────────────┐
│ 2. 尝试 CLI RPC                 │
│    ├─ 启动 codex RPC 服务器      │
│    ├─ 发送 JSON-RPC 请求         │
│    └─ 解析响应                   │
└─────────────────────────────────┘
    ↓ (如果失败)
┌─────────────────────────────────┐
│ 3. 尝试 CLI PTY                 │
│    ├─ 在 PTY 中运行 codex        │
│    ├─ 发送 /status              │
│    └─ 解析文本输出               │
└─────────────────────────────────┘
    ↓ (可选并行)
┌─────────────────────────────────┐
│ 附加: OpenAI Web 仪表板          │
│    ├─ 导入浏览器 Cookie          │
│    ├─ 加载 chatgpt.com          │
│    └─ 提取额外数据               │
└─────────────────────────────────┘
    ↓
更新 UI 显示
```

## 快速调试

### 查看 OAuth 凭证
```bash
cat ~/.codex/auth.json
```

### 手动测试 CLI
```bash
# 启动 Codex 并输入 /status
codex -s read-only -a untrusted
```

### 查看应用日志
```bash
# macOS 日志位置
tail -f ~/Library/Logs/CodexBar/codexbar.log
```

## 常见问题

**Q: 为什么有多种获取方式？**

A: 提供冗余和可靠性。如果一种方式失败（如网络问题、CLI 未安装等），自动切换到备用方式。

**Q: 哪种方式最快？**

A: OAuth API 最快，因为是直接的 HTTP 请求。CLI RPC 次之，PTY 最慢（需要启动完整的终端会话）。

**Q: 数据更新频率是多少？**

A: 默认每 10 秒刷新一次，可在设置中调整。

**Q: 如何知道当前使用的是哪种方式？**

A: 在菜单栏点击 Codex 图标，底部会显示数据源标签（如 "oauth"、"codex-cli"、"openai-web"）。

## 总结

CodexBar 通过多层次的数据获取策略，确保始终能够获取到 Codex 的 token 使用情况：

1. **首选 OAuth API** - 快速、准确、官方支持
2. **备用 CLI RPC** - 本地通信，不依赖网络
3. **兜底 PTY** - 最基础但最可靠的方式
4. **可选 Web** - 提供额外的详细信息

这种设计保证了即使在各种环境和网络条件下，都能正常监控 token 使用情况。
