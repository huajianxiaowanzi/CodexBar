# Codex Token 使用情况获取 - 项目总结

## 问题回答

**问题**: 我想知道这个项目怎么实现的获取codex的token的使用情况和剩余情况

**简短回答**: 

CodexBar 通过 **四种数据源** 获取 Codex 的 token 使用情况，并实现了智能的自动回退机制：

1. **OAuth API**（首选）- 从 `~/.codex/auth.json` 读取凭证，调用 ChatGPT API
2. **CLI RPC** - 启动本地 Codex CLI 的 JSON-RPC 服务器通信
3. **CLI PTY** - 在虚拟终端运行 `codex /status` 并解析文本输出
4. **Web Dashboard**（可选附加）- 使用 WebKit 加载 ChatGPT 网页提取额外信息

当一种方式失败时，自动尝试下一种，确保可靠性。

## 详细文档

我已经为您创建了完整的中文文档，详细解释了实现原理：

### 📚 文档列表

1. **[文档索引](./codex-documentation-index-cn.md)** - 从这里开始，根据您的需求选择合适的文档

2. **[快速指南](./codex-usage-quick-guide-cn.md)** (~5分钟阅读)
   - 三种主要方式的简单说明
   - 代码示例
   - 数据流程图
   - 常见问题解答

3. **[实现详解](./codex-implementation-cn.md)** (~20分钟阅读)
   - 每种数据源的详细工作原理
   - 完整代码分析
   - 错误处理机制
   - 性能优化技巧

4. **[架构图](./codex-architecture-cn.md)** (~15分钟阅读)
   - 系统架构总览
   - 详细的 ASCII 架构图
   - 数据模型和流程
   - 并发和线程模型

## 核心实现概览

### 数据获取流程

```
用户请求 → UsageStore → ProviderFetchPipeline
              ↓
    ┌─────────┴─────────┐
    │ 按优先级尝试策略   │
    └─────────┬─────────┘
              ↓
    ┌─────────────────┐
    │ 1. OAuth API    │ ← 最快，首选
    │    (auth.json)  │
    └────┬────────────┘
         │ 失败？
         ↓
    ┌─────────────────┐
    │ 2. CLI RPC      │ ← 本地通信
    │    (JSON-RPC)   │
    └────┬────────────┘
         │ 失败？
         ↓
    ┌─────────────────┐
    │ 3. CLI PTY      │ ← 最可靠
    │    (/status)    │
    └────┬────────────┘
         │
         ↓
    返回使用情况数据
```

### 关键代码文件

```
Sources/CodexBarCore/Providers/Codex/
├── CodexOAuth/
│   ├── CodexOAuthUsageFetcher.swift    ← OAuth API 实现
│   ├── CodexOAuthCredentials.swift     ← 凭证读取/保存
│   └── CodexTokenRefresher.swift       ← Token 刷新
├── CodexStatusProbe.swift              ← CLI PTY 解析
├── CodexCLISession.swift               ← 持久会话管理
├── CodexWebDashboardStrategy.swift     ← Web 仪表板
├── CodexProviderDescriptor.swift       ← 策略选择逻辑
└── CodexUsageDataSource.swift          ← 数据源定义
```

### OAuth API 实现示例

```swift
// 1. 读取凭证
let credentials = try CodexOAuthCredentialsStore.load()
// 从 ~/.codex/auth.json 读取 access_token 和 refresh_token

// 2. 刷新 token（如果需要）
if credentials.needsRefresh {  // 超过8天
    credentials = try await CodexTokenRefresher.refresh(credentials)
    try CodexOAuthCredentialsStore.save(credentials)
}

// 3. 调用 API
let usage = try await CodexOAuthUsageFetcher.fetchUsage(
    accessToken: credentials.accessToken,
    accountId: credentials.accountId
)
// GET https://chatgpt.com/backend-api/wham/usage
// Authorization: Bearer <access_token>

// 4. 返回的数据包含：
// - rateLimit.primaryWindow: 5小时窗口使用情况
// - rateLimit.secondaryWindow: 每周窗口使用情况
// - credits.balance: 剩余积分
// - planType: 计划类型（free, plus, pro, team, etc.）
```

### CLI PTY 实现示例

```swift
// 1. 在伪终端中启动 Codex
let runner = TTYCommandRunner()
let result = try runner.run(
    binary: "codex",
    send: "/status\n",
    options: .init(rows: 60, cols: 200, timeout: 18.0)
)

// 2. 解析输出文本
let text = result.text
// 示例输出:
// Credits: 1,234.56
// 5h limit: 45% used (resets in 2h 30m)
// Weekly limit: 23% used (resets in 4 days)

// 3. 提取数据
let snapshot = try CodexStatusProbe.parse(text: text)
// snapshot.credits = 1234.56
// snapshot.fiveHourPercentLeft = 55
// snapshot.weeklyPercentLeft = 77
```

## 设计特点

### 1. 多层次回退机制
- 每个数据源都是独立的策略（Strategy Pattern）
- 失败时自动尝试下一个策略
- 确保在各种环境下都能工作

### 2. 性能优化
- OAuth API 最快（直接 HTTP 请求）
- Token 只在需要时刷新（8天周期）
- 可选的持久 CLI 会话（减少启动开销）
- 缓存机制（Keychain, 文件系统）

### 3. 用户友好
- 自动选择最佳数据源（Auto 模式）
- 清晰的错误消息和自动恢复
- 灵活的配置选项（Settings → Providers → Codex）

### 4. 架构清晰
- Protocol-oriented design
- 每个组件职责单一
- 易于测试和维护

## 如何使用

### 对于终端用户
1. 安装 CodexBar
2. 运行 `codex` 命令登录（会自动创建 auth.json）
3. CodexBar 会自动选择最佳方式获取使用情况
4. （可选）在设置中启用 "OpenAI web extras" 获取更多信息

### 对于开发者
1. 阅读[快速指南](./codex-usage-quick-guide-cn.md)了解基础
2. 查看[实现详解](./codex-implementation-cn.md)了解细节
3. 参考[架构图](./codex-architecture-cn.md)理解设计
4. 在相关文件中查找代码实现

## 常见使用场景

### 场景 1: 仅使用 OAuth API
```bash
# 确保已登录
codex

# CodexBar 会自动使用 ~/.codex/auth.json
# 在应用中选择: Settings → Providers → Codex → Usage source: OAuth
```

### 场景 2: 仅使用 CLI
```bash
# 安装 Codex CLI
npm install -g @openai/codex

# CodexBar 会自动检测并使用
# 在应用中选择: Settings → Providers → Codex → Usage source: CLI
```

### 场景 3: 自动模式（推荐）
```bash
# 设置为 Auto（默认）
# Settings → Providers → Codex → Usage source: Auto

# 会按优先级尝试：
# 1. OAuth API（如果 auth.json 存在）
# 2. CLI RPC
# 3. CLI PTY
```

### 场景 4: 添加 Web 仪表板附加功能
```bash
# Settings → Providers → Codex
# 1. 启用 "OpenAI web extras"
# 2. OpenAI cookies: Automatic (自动从浏览器导入)

# 提供额外信息：
# - 使用详情图表（按模型分类）
# - 积分使用历史
# - 代码审查剩余次数
```

## 调试技巧

### 检查当前使用的数据源
点击菜单栏的 Codex 图标，底部会显示数据源标签：
- `oauth` - 使用 OAuth API
- `codex-cli` - 使用 CLI
- `oauth + openai-web` - OAuth + Web 仪表板

### 查看日志
```bash
# macOS 日志位置
tail -f ~/Library/Logs/CodexBar/codexbar.log
```

### 手动测试各个数据源
```bash
# 测试 OAuth 凭证
cat ~/.codex/auth.json

# 测试 CLI
codex -s read-only -a untrusted
# 在 CLI 中输入: /status

# 测试 CLI RPC
codex -s read-only -a untrusted app-server
# 在另一个终端通过 stdin 发送 JSON-RPC 请求
```

### 强制刷新
- 点击 Codex 图标
- 选择 "Refresh now"
- 或使用快捷键（在设置中查看）

## 总结

CodexBar 的 Codex token 使用情况获取实现是一个精心设计的系统，具有：

✅ **可靠性** - 多层次回退机制确保在各种环境下都能工作  
✅ **性能** - 优先使用最快的 OAuth API  
✅ **灵活性** - 支持多种数据源和配置选项  
✅ **用户友好** - 自动选择最佳方式，无需用户干预  
✅ **可维护性** - 清晰的架构，易于理解和扩展

希望这些文档能帮助您完全理解 CodexBar 如何实现 Codex token 使用情况的监控！

## 更多资源

- [官方英文文档](./codex.md)
- [Provider 开发指南](./provider.md)
- [项目架构](./architecture.md)

---

**文档作者**: GitHub Copilot  
**最后更新**: 2024-02-04  
**文档语言**: 简体中文
