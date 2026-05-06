# Humera

Human 掌控的 AI 协作工具。

## Core Concept

Human 发起所有任务，执行所有 slash command。AI 负责协助确认和建议。

```
Human 执行 /on <path> → 设置项目 + 进入讨论模式
Human 与 AI 迭代讨论需求
Human 执行 /create-issue → 创建 GitHub Issue
Human 执行 /fix <issue> → AI 编码、push、PR
Human 执行 /review <pr> → AI review 代码
Human 执行 /revise <pr> → AI 修正代码
```

## 5 Commands

| Command | Description | Output |
|---------|-------------|--------|
| `/on <path>` | 切换项目 + 进入讨论 | 项目上下文 |
| `/create-issue` | 创建 Issue | GitHub Issue |
| `/fix <issue>` | 开发 Issue | GitHub PR |
| `/review <pr>` | Review PR | PR Comment |
| `/revise <pr>` | 修正 PR | Updated PR |

## Quick Start

```bash
# 切换项目，自动进入讨论模式
/on ~/code/myapp

# Human 与 AI 讨论需求...
# AI 不断确认细节，Human 不断补充
# 直到双方确认

# 创建 Issue
/create-issue

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

## Architecture

```
humera/
├── channel/      # IM 平台
├── command/      # slash command 解析
├── dispatch/     # 命令分发
├── handler/      # 任务处理器（Discuss/Develop/Review/Revise）
├── project/      # 项目上下文
├── github/       # GitHub API
├── provider/     # LLM
├── coder/        # 编码执行
└── storage/      # 本地存储
```

## License

MIT
