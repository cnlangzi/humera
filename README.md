# Humera

Human 掌控的 AI 协作工具。

## Core Concept

Human 发起所有任务，执行所有 slash command。AI 负责协助确认和建议。

```
Human 提出 → AI 确认 → Human 确认 → Human 执行 slash command
```

## 4 Tasks

| Task | Command | Output |
|------|---------|--------|
| **Discuss** | `/discuss` + `/create-issue` | GitHub Issue |
| **Develop** | `/start-dev <issue>` | GitHub PR |
| **Review** | `/review <pr>` | PR Comment |
| **Revise** | `/revise <pr>` | Updated PR |

## Quick Start

```bash
# 切换项目
/on ~/code/myapp

# 需求讨论
/discuss 给 API 加 rate limiting
AI 确认细节...
/create-issue

# 开始开发
/start-dev https://github.com/owner/repo/issues/123
AI 自动编码、push、PR

# Review
/review https://github.com/owner/repo/pull/456
AI 输出 review 到 PR

# 修正
/revise https://github.com/owner/repo/pull/456
AI 读取反馈、修改、push
```

## Architecture

```
humera/
├── channel/      # IM 平台
├── command/      # slash command 解析
├── task/         # 任务处理器
├── project/      # 项目上下文
├── provider/     # LLM
├── github/       # GitHub API
└── infra/       # 存储、日志
```

## License

MIT
