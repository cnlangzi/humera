# Humera

Humera is a **team-style AI-assisted software engineering platform** for solo developers. It transforms a single developer into a complete engineering team by coordinating multiple AI agents through structured workflows.

## Overview

Humera follows a **Human-in-the-Loop** architecture where the Human acts as Team Lead and AI agents execute specific roles:

```
Human (Team Lead)
    │
    │ 1. Discuss requirements → AI paraphrases → Human confirms
    │ 2. /create-issue → GitHub Issue created
    │ 3. /start-dev → Dev Agent implements → PR created
    │ 4. AI Review → Human Review
    │ 5. /merge → Merged to main
    │
    ▼
AI Team Members (PM Agent, Dev Agent, Reviewer Agent)
```

## Key Features

- **Structured Workflow**: Project-level milestones with human approval checkpoints
- **Team Simulation**: PM, Dev, and Reviewer agents work together
- **Versioned Artifacts**: Every requirement, issue, and PR is versioned
- **Multi-Channel**: Telegram, Feishu, WhatsApp, Discord, CLI support
- **Engineering Precision**: Every step has clear input, output, and verification

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed architecture documentation.

## Quick Start

```bash
# Clone and build
go build ./cmd/humera

# Run with CLI channel
./humera --channel=cli

# Or connect to Telegram
./humera --channel=telegram --telegram-token=YOUR_TOKEN
```

## Commands

| Command | Description |
|---------|-------------|
| `/switch <project>` | Switch to another project |
| `/create-issue` | Create GitHub Issue from discussion |
| `/start-dev` | Start development for current issue |
| `/ai-review` | Run AI code review |
| `/merge` | Merge current PR |
| `/revise` | Request fixes from Dev Agent |

## License

MIT
