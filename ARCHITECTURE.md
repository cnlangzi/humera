# Humera Architecture

## 1. Design Philosophy

Humera is a **team-style AI-assisted software engineering platform** for solo developers. It transforms a single developer into a complete engineering team by coordinating multiple AI agents through structured workflows.

### 1.1 Core Principles

| Principle | Description |
|-----------|-------------|
| **Engineering Precision** | Every step has clear input, output, and verification. No ambiguity. |
| **Structured Workflow** | Not "one command does everything" — each action is explicit and traceable. |
| **Human-in-the-Loop** | Human is the Team Lead; AI agents execute. Critical milestones require human approval. |
| **Version Control** | All artifacts (requirements, issues, PRs) are versioned and auditable. |
| **Separation of Concerns** | Different agents have different responsibilities (PM, Dev, Reviewer). |

### 1.2 Comparison with General-Purpose Agents

| Dimension | General Agent | Humera |
|-----------|-------------|--------|
| **Goal** | One-shot complete task | Engineering-grade precision |
| **Interaction** | Chat until done | Structured commands with checkpoints |
| **Flow** | AI decides path | Human-defined workflow |
| **Error Handling** | "Try again" | Checkpoint + rollback |
| **Output** | Final result | Versioned artifacts with history |

### 1.3 Design Philosophy

```
Human (Team Lead) + AI Team Members
         │
         │ 1. Discuss requirements (AI paraphrases, Human confirms)
         │ 2. /create-issue (Create GitHub Issue)
         │ 3. /start (Dev Agent implements)
         │ 4. AI Review → Human Review
         │ 5. /merge (Merge to main)
```

**Key Insight**: The workflow controls **project-level milestones**, not individual task execution. Each milestone is a **checkpoint** where human approval is required.

---

## 2. Core Modules

```
humera/
├── channel/          # Message ingress/egress (Telegram/Feishu/CLI/WebSocket)
├── session/          # Session management (current project, context)
├── intent/           # Intent recognition (slash commands, natural language)
├── team/             # Team orchestration (PM/Dev/Reviewer agents)
├── artifact/         # Artifact management (Issue, PR, Discussion, Spec)
├── flow/             # Flow control (state machine, milestones)
├── memory/           # Memory system (discussion context, project knowledge)
├── tool/             # Tool system (GitHub, terminal, Claude Code)
├── display/          # Result display (formatted output to Human)
└── infra/            # Infrastructure (storage, config, logging)
```

---

## 3. Module Definitions

### 3.1 Channel

**Purpose**: Receive Human instructions, send formatted responses.

```go
// channel/interface.go
type Channel interface {
    Name() string
    
    // Send a message to Human
    Send(ctx context.Context, to string, msg Display) error
    
    // Receive message from Human
    Receive(ctx context.Context) (*Message, error)
    
    // Start listening for messages
    Start(ctx context.Context, handler MessageHandler) error
    
    // Stop listening
    Stop(ctx context.Context) error
}

// Supported channels
// - Telegram
// - Feishu (Lark)
// - WhatsApp
// - Discord
// - CLI
// - WebSocket
```

**Responsibilities**:
- Connect to various IM platforms
- Normalize messages to internal format
- Route messages to Session
- Send formatted results (cards, markdown, buttons)

---

### 3.2 Session

**Purpose**: Manage current working context.

```go
// session/session.go
type Session struct {
    ID        SessionID
    ChannelID ChannelID
    UserID    UserID
    
    // Current project binding
    ProjectID ProjectID
    
    // Current workflow state
    FlowState FlowState
    
    // Current artifact being discussed
    CurrentArtifact ArtifactID
    
    // Discussion history (for this session)
    Discussion []DiscussionEntry
}
```

**Responsibilities**:
- Bind Human ↔ Project
- Track current workflow state
- Route messages to correct Team member
- Manage discussion context

---

### 3.3 Intent

**Purpose**: Understand what Human wants to do.

```go
// intent/intent.go
type IntentType string

const (
    IntentDiscuss      IntentType = "discuss"      // Discuss requirements
    IntentCreateIssue  IntentType = "create-issue" // Create GitHub Issue
    IntentStartDev     IntentType = "start-dev"    // Start development
    IntentAIReview     IntentType = "ai-review"    // AI Code Review
    IntentHumanReview  IntentType = "human-review" // Human Code Review
    IntentRevise       IntentType = "revise"       // Fix issues
    IntentMerge        IntentType = "merge"        // Merge PR
    IntentSwitchProject IntentType = "switch-project" // Switch project
)

type Intent struct {
    Type IntentType
    
    // Raw input
    RawText string
    
    // Parsed parameters
    Params map[string]string
    
    // Confidence score
    Confidence float64
}

// Intent recognition
// - Slash commands: explicit (e.g., /create-issue)
// - Natural language: AI-inferred
```

**Responsibilities**:
- Parse slash commands
- Infer intent from natural language
- Route to correct Flow handler

---

### 3.4 Flow

**Purpose**: Control project-level state transitions.

```go
// flow/flow.go
type FlowState string

const (
    FlowDiscussing    FlowState = "discussing"    // Requirements discussion
    FlowIssueCreated  FlowState = "issue-created"  // Issue created
    FlowInDev         FlowState = "in-dev"        // Development in progress
    FlowPROpen        FlowState = "pr-open"        // PR opened
    FlowReviewing     FlowState = "reviewing"     // Code review
    FlowMerged        FlowState = "merged"        // Merged
    FlowClosed        FlowState = "closed"        // Closed
)

type Transition struct {
    From    FlowState
    To      FlowState
    Command string      // Command that triggers this
    Trigger TriggerType // "human" | "ai" | "auto"
}

type Milestone struct {
    State     FlowState
    RequireApproval bool
    Approver  string  // "human" | "ai"
}
```

**State Machine**:

```
 discussing ──[Human OK + /create-issue]──▶ issue-created
      │
      │ [Human OK + /start-dev]
      ▼
   in-dev ──[PR Created]──▶ pr-open ──[AI Review Done]──▶ reviewing
      │                                                       │
      │                    ◀──[Human Revise]──┐              │
      │                                                       │
      │              ◀──[/merge]──────────────────────────────┘
      │
      ▼
   merged ──[Deployed]──▶ closed
```

**Milestones (Human Approval Required)**:
| From | To | Command | Description |
|------|-----|---------|-------------|
| discussing | issue-created | /create-issue | Requirements confirmed |
| in-dev | pr-open | (auto) | Development complete |
| reviewing | merged | /merge | Code review approved |

---

### 3.5 Team

**Purpose**: Manage multiple AI agents simulating team members.

```go
// team/team.go
type Role string

const (
    RolePM       Role = "pm"        // Product Manager
    RoleDev      Role = "dev"        // Developer
    RoleReviewer Role = "reviewer"   // Code Reviewer
)

type Agent interface {
    Role() Role
    Name() string
    
    // Execute a task
    Execute(ctx context.Context, task Task) (*Result, error)
    
    // Hand off to another agent
    Handoff(ctx context.Context, to Role, context map[string]any) error
}

// Team composition
type Team struct {
    pm       Agent
    dev      Agent
    reviewer Agent
    
    // Current active agent
    activeAgent Agent
    
    // Communication bus for agent-to-agent messaging
    bus *CommBus
}
```

**Agent Responsibilities**:

| Agent | Role | Responsibilities | Tools |
|-------|------|-----------------|-------|
| PM Agent | pm | Requirements analysis, Issue drafting, Spec writing | github, display |
| Dev Agent | dev | Code implementation, PR creation | terminal, file, github, claude-code |
| Reviewer Agent | reviewer | Code review, Review result display | github, terminal, code-analysis |

**Handoff Protocol**:
```
PM Agent ──handoff(context)──▶ Dev Agent
                                    │
                                    │ handoff(result) ──▶ Reviewer Agent
                                    │                          │
                                    │ ◀──handoff_back(review)──┘
                                    │
                                    ▼
                              (Done or Reopen)
```

---

### 3.6 Artifact

**Purpose**: Manage project artifacts (Issue, PR, Discussion, Spec).

```go
// artifact/artifact.go
type ArtifactType string

const (
    ArtifactDiscussion ArtifactType = "discussion"  // Requirements discussion
    ArtifactIssue      ArtifactType = "issue"        // GitHub Issue
    ArtifactPR         ArtifactType = "pr"           // Pull Request
    ArtifactSpec       ArtifactType = "spec"        // Design specification
)

type Artifact struct {
    ID       ArtifactID
    Type     ArtifactType
    ProjectID ProjectID
    
    // Version control
    Version   int
    History   []ArtifactVersion
    
    // Content
    Title    string
    Body     string    // Markdown
    Metadata map[string]string
    
    // State
    Status   ArtifactStatus
    CreatedAt time.Time
    UpdatedAt time.Time
}

type ArtifactVersion struct {
    Version   int
    Title     string
    Body      string
    CreatedBy string
    CreatedAt time.Time
    Diff      string  // What changed from previous version
}
```

**Artifact Lifecycle**:

```
Discussion (Requirements)
    │
    │ /create-issue
    ▼
Issue #123 (GitHub Issue)
    │
    │ /start-dev
    ▼
PR #456 (Pull Request)
    │
    │ /merge
    ▼
Merged Code
```

---

### 3.7 Memory

**Purpose**: Store context and knowledge.

```go
// memory/memory.go
type Memory struct {
    Session *SessionMemory   // Current discussion
    Project *ProjectMemory   // Project-level knowledge
}

// Session memory: current discussion context
type SessionMemory struct {
    SessionID SessionID
    
    // Human's original statements
    HumanStatements []Statement
    
    // AI's paraphrased understanding
    AIParaphrases []Paraphrase
    
    // Corrections made
    Corrections []Correction
    
    // Final confirmed version
    ConfirmedVersion *ConfirmedContent
}

// Project memory: project knowledge
type ProjectMemory struct {
    ProjectID ProjectID
    
    // Previous issues and their solutions
    ClosedIssues []IssueSummary
    
    // Team conventions
    Conventions []string
    
    // Code patterns
    Patterns []Pattern
    
    // Known issues
    KnownBugs []BugSummary
}
```

---

### 3.8 Tool

**Purpose**: Capabilities for agents to execute operations.

```go
// tool/tool.go
type Tool interface {
    Name() string
    Description() string
    
    Execute(ctx context.Context, args map[string]any) ([]byte, error)
}

type Registry struct {
    tools map[string]Tool
}
```

**Core Tools**:

| Tool | Agent | Description |
|------|-------|-------------|
| github | pm, dev, reviewer | GitHub API operations (create-issue, create-pr, merge, comment) |
| terminal | dev | Execute shell commands |
| file | dev | Read/write files |
| claude-code | dev | Invoke Claude Code for code generation |
| display | all | Render results to Human |
| http | all | Send HTTP requests |

---

### 3.9 Display

**Purpose**: Format and present AI results to Human.

```go
// display/display.go
type Display interface {
    // Render requirements understanding
    RequirementsUnderstanding(content *Requirements) DisplayBlock
    
    // Render Issue draft
    IssueDraft(issue *Artifact) DisplayBlock
    
    // Render PR description
    PRDescription(pr *Artifact) DisplayBlock
    
    // Render Code Review result
    ReviewResult(review *Review) DisplayBlock
    
    // Render confirmation buttons
    Confirmation(buttons []Button) DisplayBlock
}

type DisplayBlock struct {
    Type    string      // "markdown" | "card" | "buttons" | "table"
    Content interface{}
}
```

**Supported Formats**:
- Markdown (for CLI, Telegram)
- Card (for Feishu, Discord)
- Interactive buttons (for confirmation actions)

---

### 3.10 Infra

**Purpose**: Core infrastructure services.

```go
// infra/infra.go
type Infra struct {
    Storage   *Storage     // SQLite + GitHub API
    Config    *Config      // Configuration management
    Logger    *Logger      // Structured logging
    Creds     *CredVault   // API key management
    Telemetry *Telemetry   // Tracing, metrics
}
```

---

## 4. Component Relationships

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Human (Team Lead)                                  │
│                                                                                 │
│  Commands: /create-issue, /start, /review, /merge                             │
│  Natural Language: Requirements discussions                                     │
│                                                                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Message
                                         ▼
┌────────────────────────────────────────▼────────────────────────────────────────┐
│                                    Channel                                       │
│                                                                                 │
│  Telegram / Feishu / WhatsApp / Discord / CLI / WebSocket                      │
│  Responsibility: Normalize messages, route to Session                          │
│                                                                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Route to Session
                                         ▼
┌────────────────────────────────────────▼────────────────────────────────────────┐
│                                    Session                                       │
│                                                                                 │
│  Current Project + Current State + Current Artifact                            │
│  Responsibility: Context binding, message routing                              │
│                                                                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Parse Intent
                                         ▼
┌────────────────────────────────────────▼────────────────────────────────────────┐
│                                     Intent                                       │
│                                                                                 │
│  Slash Command: /create-issue → IntentCreateIssue                              │
│  Natural Language: "add rate limiting" → IntentDiscuss                          │
│  Responsibility: Understand what Human wants                                    │
│                                                                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Determine action
                                         ▼
┌────────────────────────────────────────▼────────────────────────────────────────┐
│                                     Flow                                          │
│                                                                                 │
│  State Machine: discussing → issue-created → in-dev → ...                       │
│  Milestones: Human approval required at key transitions                          │
│  Responsibility: Control project-level workflow                                 │
│                                                                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Handoff to Agent
                                         ▼
┌────────────────────────────────────────▼────────────────────────────────────────┐
│                                     Team                                          │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                            PM Agent                                       │   │
│  │  Role: pm | Tools: github, display                                       │   │
│  │  Execute: Requirements analysis, Issue drafting, Display results         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                            Dev Agent                                      │   │
│  │  Role: dev | Tools: terminal, file, github, claude-code                   │   │
│  │  Execute: Code implementation, PR creation                                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         Reviewer Agent                                   │   │
│  │  Role: reviewer | Tools: github, terminal, code-analysis                  │   │
│  │  Execute: Code review, Output review results                              │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Create/Update
                                         ▼
┌────────────────────────────────────────▼────────────────────────────────────────┐
│                                    Artifact                                       │
│                                                                                 │
│  Discussion ──▶ Issue ──▶ PR ──▶ Merged                                        │
│  Versioned: v1, v2, v3... (every change is tracked)                            │
│                                                                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Store
                                         ▼
┌────────────────────────────────────────▼────────────────────────────────────────┐
│                                    Memory                                         │
│                                                                                 │
│  Session Memory: Current discussion context                                     │
│  Project Memory: Closed issues, patterns, conventions                          │
│                                                                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Render results
                                         ▼
┌────────────────────────────────────────▼────────────────────────────────────────┐
│                                    Display                                       │
│                                                                                 │
│  Markdown / Card / Buttons                                                       │
│  Back to Human via Channel                                                      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Command Specification

### 5.1 Slash Commands

| Command | Intent | Action | Approver |
|---------|--------|--------|----------|
| `/switch <project>` | switch-project | Bind session to another project | - |
| `/create-issue` | create-issue | Create GitHub Issue from discussion | Human |
| `/start-dev` | start-dev | Start development (PM → Dev handoff) | Human |
| `/start` | start-dev | Alias for /start-dev | Human |
| `/ai-review` | ai-review | Run AI code review | - |
| `/review` | human-review | Human begins manual review | Human |
| `/revise` | revise | Fix review issues | Human |
| `/merge` | merge | Merge PR to main | Human |
| `/cancel` | cancel | Cancel current flow | Human |

### 5.2 Natural Language Mapping

| Phrase | Intent | Notes |
|--------|--------|-------|
| "我想加一个功能..." | discuss | Start requirements discussion |
| "帮我看看这个 bug" | discuss | Start bug discussion |
| "确认" | approve | Approve current state |
| "我来 review" | human-review | Human starts review |
| "有问题" | revise | Need fixes |
| "没问题了" | approve | Approval after fixes |

### 5.3 Command Flow Examples

**Requirements Discussion Flow**:
```
1. Human: "我要给 API 加 rate limiting"
   → Intent: discuss
   → Display: AI paraphrases understanding
   
2. Human: "对，但是限制改成 200 req/min"
   → Intent: discuss (with correction)
   → Display: AI shows updated understanding
   
3. Human: "确认，/create-issue"
   → Intent: create-issue
   → PM Agent creates GitHub Issue
   → Flow: discussing → issue-created
```

**Development Flow**:
```
1. Human: "/start-dev"
   → Intent: start-dev
   → Flow: issue-created → in-dev
   → Dev Agent starts implementation
   → Dev Agent creates PR
   → Flow: in-dev → pr-open → reviewing
   
2. (AI runs automated review)
   → Reviewer Agent outputs review results
   → Display shows review to Human
   
3. Human: "/review"
   → Intent: human-review
   → Human reviews code
   
4. Human: "有几个问题，需要修改"
   → Intent: revise
   → Flow: reviewing → in-dev
   → Dev Agent fixes issues
   → Flow: in-dev → reviewing (again)
   
5. Human: "/merge"
   → Intent: merge
   → Flow: reviewing → merged
```

---

## 6. State Transitions

### 6.1 Full State Diagram

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                                                         │
                    │     ┌───────────┐                                       │
                    │     │ discussing │ ←── Human starts requirements        │
                    │     └─────┬─────┘                                       │
                    │           │                                              │
                    │           │ Human: "/create-issue"                       │
                    │           │ Human: "confirm"                            │
                    │           ▼                                              │
                    │     ┌───────────────┐                                    │
                    │     │ issue-created │ ←── GitHub Issue created         │
                    │     └───────┬───────┘                                    │
                    │             │                                             │
                    │             │ Human: "/start-dev"                        │
                    │             │ Human: "confirm"                            │
                    │             ▼                                             │
                    │     ┌─────────────┐                                      │
                    │     │  in-dev     │ ←── Dev Agent working                │
                    │     └──────┬──────┘                                      │
                    │            │                                              │
                    │            │ Dev Agent: PR created                       │
                    │            ▼                                              │
                    │     ┌─────────────┐                                      │
                    │     │  pr-open   │ ←── PR submitted                      │
                    │     └──────┬──────┘                                      │
                    │            │                                              │
                    │            │ AI Reviewer: review complete                 │
                    │            ▼                                              │
                    │     ┌─────────────┐                                      │
    Human:          │     │ reviewing   │                                      │
    "/revise" ──────┼──→  └──────┬──────┘                                      │
                        │            │                                             │
                        │            │ Human: "/merge"                           │
                        │            │ Human: "LGTM"                              │
                        │            ▼                                             │
                        │     ┌─────────────┐                                      │
                        │     │  merged    │ ←── Code merged to main            │
                        │     └─────────────┘                                      │
                        │            │                                             │
                        │            │ Auto: deployed                             │
                        │            ▼                                             │
                        │     ┌─────────────┐                                      │
                        │     │  closed    │ ←── Issue closed                    │
                        │     └─────────────┘                                      │
                        │                                                           │
                        └─────────────────────────────────────────────────────────┘
```

### 6.2 Transition Rules

| Current State | Command | Target State | Trigger | Action |
|--------------|---------|--------------|---------|--------|
| discussing | /create-issue | issue-created | human | PM creates Issue |
| discussing | /cancel | - | human | Clear discussion |
| issue-created | /start-dev | in-dev | human | Dev starts work |
| issue-created | /cancel | - | human | Cancel Issue |
| in-dev | (PR created) | pr-open | ai | Auto-transition |
| in-dev | /cancel | - | human | Stop dev, revert |
| pr-open | (AI review done) | reviewing | ai | Auto-transition |
| reviewing | /revise | in-dev | human | Return to dev |
| reviewing | /merge | merged | human | Merge PR |
| reviewing | /cancel | - | human | Close PR |
| merged | (deployed) | closed | ai | Auto-transition |

---

## 7. Data Models

### 7.1 Project

```go
type Project struct {
    ID       ProjectID
    Name     string
    RepoURL  string          // e.g., "github.com/cnlangzi/xun-web"
    Language string           // e.g., "Go", "TypeScript"
    Framework string          // e.g., "Echo", "React"
    
    // GitHub integration
    GitHubRepo  string
    GitHubToken string        // Encrypted
    
    // Project-specific config
    Config map[string]string
    
    // Memory
    Memory *ProjectMemory
    
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

### 7.2 Discussion

```go
type Discussion struct {
    ID        DiscussionID
    ProjectID ProjectID
    SessionID SessionID
    
    // Content
    Title    string
    Messages []DiscussionMessage
    
    // Versions (each confirmation creates a version)
    Versions []DiscussionVersion
    
    // Status
    Status   DiscussionStatus  // active | confirmed | cancelled
    ConfirmedVersion *int       // Pointer to confirmed version number
    
    CreatedAt time.Time
    UpdatedAt time.Time
}

type DiscussionMessage struct {
    Role    string    // "human" | "ai"
    Content string
    Type    string    // "statement" | "paraphrase" | "correction" | "confirmation"
}
```

### 7.3 Issue

```go
type Issue struct {
    ID        IssueID
    ProjectID ProjectID
    GitHubNumber int    // GitHub Issue number
    
    // Content
    Title   string
    Body    string    // Markdown
    
    // Requirements
    Requirements []Requirement
    
    // Design
    Design DesignSpec
    
    // Status
    Status IssueStatus  // draft | confirmed | in-progress | done | closed
    
    // Workflow
    FlowState FlowState
    
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

### 7.4 PullRequest

```go
type PullRequest struct {
    ID        PRID
    IssueID   IssueID
    ProjectID ProjectID
    GitHubNumber int
    
    // Content
    Title   string
    Body    string
    Head    string    // Branch name
    Base    string    // Target branch
    
    // Review
    Reviews []Review
    
    // CI Status
    CIStatus CIStatus
    
    // Status
    Status PRStatus  // open | approved | changes-requested | merged | closed
    
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

---

## 8. Technical Architecture

### 8.1 Project Structure

```
humera/
├── cmd/
│   └── humera/
│       └── main.go
├── internal/
│   ├── channel/          # Channel implementations
│   │   ├── gateway.go
│   │   ├── message.go
│   │   ├── cli.go
│   │   ├── telegram.go
│   │   └── feishu.go
│   ├── session/
│   │   └── session.go
│   ├── intent/
│   │   ├── intent.go
│   │   └── parser.go
│   ├── team/
│   │   ├── team.go
│   │   ├── pm.go
│   │   ├── dev.go
│   │   └── reviewer.go
│   ├── artifact/
│   │   ├── artifact.go
│   │   ├── issue.go
│   │   ├── pr.go
│   │   └── discussion.go
│   ├── flow/
│   │   ├── flow.go
│   │   └── state.go
│   ├── memory/
│   │   ├── memory.go
│   │   ├── session.go
│   │   └── project.go
│   ├── tool/
│   │   ├── registry.go
│   │   ├── github.go
│   │   ├── terminal.go
│   │   └── claude_code.go
│   ├── display/
│   │   └── display.go
│   └── infra/
│       ├── storage.go
│       ├── config.go
│       ├── logger.go
│       └── creds.go
├── pkg/
│   └── wasm/             # Wasm plugin host (future)
├── ARCHITECTURE.md
└── README.md
```

### 8.2 Runtime Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          humera main                                │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      Runtime                                 │   │
│  │                                                             │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │   │
│  │  │Channel  │  │Session  │  │ Intent  │  │  Flow   │      │   │
│  │  │Gateway  │──│Manager  │──│ Parser  │──│Machine  │      │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └────┬────┘      │   │
│  │                                                  │           │   │
│  │                                                  ▼           │   │
│  │  ┌─────────────────────────────────────────────────────────┐ │   │
│  │  │                      Team                              │ │   │
│  │  │                                                         │ │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                │ │   │
│  │  │  │   PM    │  │   Dev   │  │Reviewer │                │ │   │
│  │  │  │ Agent   │  │ Agent   │  │ Agent   │                │ │   │
│  │  │  └────┬────┘  └────┬────┘  └────┬────┘                │ │   │
│  │  │       │            │            │                      │ │   │
│  │  │       └────────────┴────────────┘                      │ │   │
│  │  │                      │                                 │ │   │
│  │  │              ┌───────┴───────┐                         │ │   │
│  │  │              │  Comm Bus     │                         │ │   │
│  │  │              └───────────────┘                         │ │   │
│  │  └─────────────────────────────────────────────────────────┘ │   │
│  │                                                             │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │   │
│  │  │Artifact │  │ Memory  │  │  Tool   │  │Display  │      │   │
│  │  │Manager  │  │Manager  │  │Registry │  │ Engine  │      │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                       Infra                                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │   │
│  │  │Storage  │  │ Config  │  │ Logger  │  │ Creds   │      │   │
│  │  │(SQLite) │  │         │  │         │  │ Vault   │      │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Design Rationale

### 9.1 Why Team-Based?

A solo developer needs to act as:
- **PM**: Define requirements, prioritize
- **Dev**: Write code
- **QA**: Test
- **Reviewer**: Code review

By modeling these as separate agents with clear responsibilities:
- Each agent can be specialized (different system prompts, tools)
- Handoffs create natural checkpoints
- Parallel work is possible (e.g., AI review while Human reviews)

### 9.2 Why Structured Commands?

Natural language is ambiguous. "start the thing" could mean:
- Start development
- Start a new discussion
- Resume a paused task

Slash commands (`/start-dev`) make intent explicit and unambiguous.

### 9.3 Why Milestone-Based?

Not every step needs human approval (that defeats the purpose of automation). But critical milestones do:
- Requirements confirmed → prevents wasted development
- Code reviewed → maintains quality
- Deployment approved → prevents broken main

### 9.4 Why Artifact Versioning?

Requirements evolve. The first draft might be incomplete. By tracking versions:
- Every change is visible
- Rollback is possible
- Audit trail exists for future reference

---

## 10. Future Considerations

### 10.1 Wasm Plugin System

For future extensibility, tools can be loaded as Wasm plugins:
- Isolated execution environment
- Version management
- Crash isolation

### 10.2 Multiple Human Support

Currently designed for solo developer. Future version could support:
- Team with multiple developers
- Role-based access control
- Collaborative review

### 10.3 LLM Provider Abstraction

Currently assumes Claude API. Future could support:
- OpenAI
- Local models (Ollama)
- Model routing based on task type
