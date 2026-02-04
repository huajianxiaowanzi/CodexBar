# Codex Token 使用情况获取 - 文档索引

本目录包含关于 CodexBar 如何获取 Codex token 使用情况的完整文档。

## 📚 文档列表

### 1. [快速指南](./codex-usage-quick-guide-cn.md) ⚡
**适合人群**: 想要快速了解基本原理的开发者

**内容概要**:
- 三种主要获取方式的简单说明
- 代码示例和数据流程图
- 常见问题解答
- 快速调试技巧

**阅读时间**: ~5 分钟

### 2. [实现详解](./codex-implementation-cn.md) 📖
**适合人群**: 需要深入理解实现细节的开发者

**内容概要**:
- 每种数据源的详细工作原理
- 完整的代码分析
- 错误处理和回退机制
- 性能优化技巧
- 用户配置选项说明

**阅读时间**: ~20 分钟

### 3. [架构图](./codex-architecture-cn.md) 🏗️
**适合人群**: 想要理解系统架构的开发者和架构师

**内容概要**:
- 系统架构总览
- 各组件的详细架构图（ASCII 图）
- 数据模型和转换流程
- 并发和线程模型
- 缓存机制设计

**阅读时间**: ~15 分钟

### 4. [原始英文文档](./codex.md) 🇬🇧
**适合人群**: 英语开发者或需要最新官方信息的读者

**内容概要**:
- 官方维护的英文文档
- 可能包含最新的更新信息

## 🎯 根据场景选择文档

### 场景 1: 我是新手，想快速了解
**推荐阅读顺序**:
1. [快速指南](./codex-usage-quick-guide-cn.md) - 了解基础概念
2. [实现详解](./codex-implementation-cn.md) 的"概述"和"数据源优先级"部分

### 场景 2: 我要修复 bug 或添加功能
**推荐阅读顺序**:
1. [快速指南](./codex-usage-quick-guide-cn.md) - 快速复习
2. [实现详解](./codex-implementation-cn.md) - 查找相关实现
3. [架构图](./codex-architecture-cn.md) - 理解组件关系

### 场景 3: 我要重构或优化代码
**推荐阅读顺序**:
1. [架构图](./codex-architecture-cn.md) - 理解整体架构
2. [实现详解](./codex-implementation-cn.md) - 深入了解每个组件
3. [原始英文文档](./codex.md) - 参考官方设计意图

### 场景 4: 我在排查问题
**推荐阅读顺序**:
1. [快速指南](./codex-usage-quick-guide-cn.md) 的"常见问题"部分
2. [实现详解](./codex-implementation-cn.md) 的"错误处理"和"调试技巧"部分
3. 根据具体错误查看相关组件的详细说明

## 🔑 关键概念速查

### 数据源类型
| 类型 | 速度 | 准确性 | 依赖 | 默认优先级 (App) |
|------|------|--------|------|-----------------|
| OAuth API | ⚡⚡⚡ | ✅✅✅ | auth.json | 1 |
| CLI RPC | ⚡⚡ | ✅✅✅ | codex CLI | 2 |
| CLI PTY | ⚡ | ✅✅ | codex CLI | 3 |
| Web Dashboard | ⚡ | ✅✅ | Browser cookies | 可选 |

### 文件位置速查
| 内容 | 路径 |
|------|------|
| OAuth 凭证 | `~/.codex/auth.json` |
| 会话日志 | `~/.codex/sessions/YYYY/MM/DD/*.jsonl` |
| 归档会话 | `~/.codex/archived_sessions/*.jsonl` |
| 成本缓存 | `~/Library/Caches/CodexBar/cost-usage/codex-v1.json` |
| 应用日志 | `~/Library/Logs/CodexBar/` |

### 代码文件速查
| 功能 | 文件路径 |
|------|---------|
| OAuth API | `Sources/CodexBarCore/Providers/Codex/CodexOAuth/CodexOAuthUsageFetcher.swift` |
| CLI PTY | `Sources/CodexBarCore/Providers/Codex/CodexStatusProbe.swift` |
| CLI Session | `Sources/CodexBarCore/Providers/Codex/CodexCLISession.swift` |
| Web Dashboard | `Sources/CodexBarCore/Providers/Codex/CodexWebDashboardStrategy.swift` |
| Provider Descriptor | `Sources/CodexBarCore/Providers/Codex/CodexProviderDescriptor.swift` |
| Usage Fetcher | `Sources/CodexBarCore/UsageFetcher.swift` |

## 💡 快速参考

### 测试命令
```bash
# 测试 Codex CLI
codex -s read-only -a untrusted

# 在 CLI 中输入
/status

# 查看 OAuth 凭证
cat ~/.codex/auth.json

# 查看会话文件
ls -la ~/.codex/sessions/

# 查看应用日志
tail -f ~/Library/Logs/CodexBar/codexbar.log
```

### 常见错误代码
```swift
// Codex CLI 未安装
CodexStatusProbeError.codexNotInstalled

// 需要更新 Codex CLI
CodexStatusProbeError.updateRequired

// OAuth token 无效
CodexOAuthFetchError.unauthorized

// 解析失败
CodexStatusProbeError.parseFailed
```

### 数据结构
```swift
// OAuth API 响应
CodexUsageResponse {
    planType: PlanType?
    rateLimit: RateLimitDetails?
    credits: CreditDetails?
}

// 统一快照
UsageSnapshot {
    primary: RateWindow        // 5小时窗口
    secondary: RateWindow?     // 每周窗口
    tertiary: RateWindow?      // 第三窗口（如有）
    updatedAt: Date
    identity: ProviderIdentitySnapshot
}
```

## 🔗 相关资源

### 外部链接
- [Codex CLI 官方文档](https://platform.openai.com/docs/guides/codex)
- [ChatGPT Dashboard](https://chatgpt.com/codex/settings/usage)
- [OpenAI 状态页](https://status.openai.com/)

### 项目内部文档
- [Provider 开发指南](./provider.md)
- [开发环境设置](./DEVELOPMENT.md)
- [发布流程](./RELEASING.md)

## 📝 文档更新

这些文档是根据代码库在 2024 年的状态编写的。如果代码有重大更新，文档可能需要相应更新。

### 如何贡献
如果你发现文档中有错误或需要补充的内容，请：
1. 在 GitHub 上提交 Issue
2. 或者直接提交 Pull Request 更新文档

### 最后更新
- 快速指南: 2024-02-04
- 实现详解: 2024-02-04
- 架构图: 2024-02-04

---

**注意**: 这些文档是中文翻译和扩展版本，原始的英文文档请参考 [codex.md](./codex.md)。
