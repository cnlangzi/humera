# Humera

Humera is a **Human-controlled AI collaboration tool** for software engineering. Unlike autonomous agent systems, Humera keeps Human in the driver's seat — AI assists with confirmation and suggestions, but all final actions are executed by Human.

## Core Concept

```
Human 提出 → AI 具体化确认 → Human 确认 → Human 执行 slash command
```

**AI never executes final actions.** AI only:
1. Confirms and specifies what Human proposed
2. Provides suggestions

**Human controls everything:**
- Creating Issues
- Pushing Code
- Merging PRs

## Task Types

| Task | AI Output | Human Action |
|------|-----------|--------------|
| **Discuss** | Confirmed requirements document | `/create-issue` |
| **Develop** | Code implementation suggestions | `/start-dev` → push code |
| **Review** | PR review comments | Output to GitHub PR |
| **Revise** | Fix suggestions | `/revise` → push fixes |

## Quick Start

```bash
# Start Humera
humera

# Switch project (reads git remote automatically)
> /on ~/code/myapp
Project: github.com/owner/repo

# Discuss requirements
> 我要给 API 加 rate limiting
[AI assists with confirmation...]

# Create issue (Human executes)
> /create-issue

# Start development
> /start-dev https://github.com/owner/repo/issues/123
[AI assists with implementation...]

# Review PR
> /review https://github.com/owner/repo/pull/456
[AI outputs review to GitHub PR]
```

## Architecture

```
humera/
├── channel/          # IM platforms (Telegram/Feishu/CLI/...)
├── command/          # Slash command parsing
├── assist/           # AI assistance
│   ├── discuss.go       # Requirements discussion
│   ├── review.go        # Code review
│   └── suggest.go        # Fix suggestions
├── project/          # Project context (from git remote)
├── artifact/         # Artifacts (Issue/Code/PR Comment)
└── infra/            # Storage, logging
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
