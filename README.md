# Humera

Human 掌控的 AI 协作工具。

## Core Concept

Human 发起每个任务，AI 执行任务全部过程。

```
/on ~/code/myapp     → 切换项目 + 进入讨论模式
/new                 → 创建 GitHub Issue
/fix <issue>         → 编码 + push + PR
/review <pr>         → Review 代码 → PR Comment
/revise <pr>         → 读取反馈 + 修改 + push
```

## 4 Tasks

| Task | Command | AI 执行 |
|------|---------|---------|
| **Discuss** | `/on <path>` | 确认需求细节，输出确认文档 |
| **Create Issue** | `/new` | 创建 GitHub Issue |
| **Develop** | `/fix <issue>` | 编码、push、PR |
| **Review** | `/review <pr>` | Review，写入 PR Comment |
| **Revise** | `/revise <pr>` | 修改代码，push |

## Quick Start

```bash
# 切换项目，进入讨论模式
/on ~/code/myapp

# Human 与 AI 讨论需求
AI 确认细节，Human 补充
直到双方确认

# 创建 Issue
/new

# 开始开发
/fix https://github.com/owner/repo/issues/123

# Review
/review https://github.com/owner/repo/pull/456

# 修正
/revise https://github.com/owner/repo/pull/456
```

## Supported Channels

- Telegram
- Feishu (Lark)
- Discord
- WhatsApp
- CLI
- WebSocket

## License

MIT
