# Humera

Human 掌控的 AI 协作工具。

## Core Concept

Human 发起所有任务，执行所有 slash command。AI 负责协助确认和建议。

```
Human 提出 → AI 确认 → Human 确认 → Human 执行 slash command
```

## 4 Tasks

| Task | Command | Description | Output |
|------|---------|-------------|--------|
| **Discuss** | `/discuss <idea>` | Human 提出需求，AI 确认细节 | - |
| **Create Issue** | `/create-issue` | 把讨论结果创建为 GitHub Issue | GitHub Issue |
| **Develop** | `/start-dev <issue>` | AI 根据 Issue 编码、push、PR | GitHub PR |
| **Review** | `/review <pr>` | AI review 代码 | PR Comment |
| **Revise** | `/revise <pr>` | AI 根据 review 反馈修正代码 | Updated PR |

## Workflow

### Discuss + Create Issue

```
Human: /discuss 给 API 加 rate limiting
AI:    确认具体细节...
Human: 补充/修改
AI:    确认理解
← 循环直到双方确认 →
Human: /create-issue
→ GitHub Issue 创建
```

### Develop

```
Human: /start-dev https://github.com/owner/repo/issues/123
AI:    读取 Issue → 编码 → push → 创建 PR
→ GitHub PR 创建
```

### Review

```
Human: /review https://github.com/owner/repo/pull/456
AI:    读取代码 → review → 写入 PR Comment
→ Review 意见在 GitHub PR 上
```

### Revise

```
Human: /revise https://github.com/owner/repo/pull/456
AI:    读取 review 反馈 → 修改代码 → push
→ 修正代码 push 到 GitHub
```

## Project

项目通过 `/on <path>` 切换，从 git remote 读取 GitHub 地址。

```bash
/on ~/code/myapp
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
