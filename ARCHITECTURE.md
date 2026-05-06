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
| **Iterative Refinement** | AI presents understanding, Human corrects, repeat until confirmed. |

### 1.2 Comparison with General-Purpose Agents

| Dimension | General Agent | Humera |
|-----------|-------------|--------|
| **Goal** | One-shot complete task | Engineering-grade precision |
| **Interaction** | Chat until done | Structured commands with checkpoints |
| **Flow** | AI decides path | Human-defined workflow |
| **Error Handling** | "Try again" | Checkpoint + rollback |
| **Output** | Final result | Versioned artifacts with history |
| **Requirements** | AI infers from chat | AI presents, Human confirms |

### 1.3 The Human-AI Collaboration Loop

The core interaction pattern is **iterative refinement**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Human: "我要给 API 加 rate limiting"                                      │
│                    │                                                         │
│                    ▼                                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │              AI presents understanding (paraphrase)                  │   │
│   │                                                                     │   │
│   │   功能：给所有 API 添加 rate limiting                                │   │
│   │   限制：100 req/min per user                                        │   │
│   │   实现：Redis + sliding window                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                    │                                                         │
│                    ▼                                                         │
│   Human: "限制改成 200 req/min"                                            │
│                    │                                                         │
│                    ▼                                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │              AI presents updated understanding                        │   │
│   │                                                                     │   │
│   │   功能：给所有 API 添加 rate limiting                                │   │
│   │   限制：200 req/min per user  ✓ (updated)                           │   │
│   │   实现：Redis + sliding window                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                    │                                                         │
│                    ▼                                                         │
│   Human: "确认"                                                             │
│                    │                                                         │
│                    ▼                                                         │
│              [Requirements Confirmed]                                        │
│                    │                                                         │
│                    ▼                                                         │
│              /create-issue                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Insight**: The AI **presents** its understanding, the Human **corrects** until satisfied. This is not a chatbot — it's a **structured review loop**.

### 1.4 Design Philosophy

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
    Discussion *Discussion
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
    IntentDiscuss       IntentType = "discuss"       // Discuss requirements
    IntentCreateIssue   IntentType = "create-issue"  // Create GitHub Issue
    IntentStartDev      IntentType = "start-dev"     // Start development
    IntentAIReview      IntentType = "ai-review"    // AI Code Review
    IntentRevise        IntentType = "revise"        // Fix issues
    IntentMerge         IntentType = "merge"         // Merge PR
    IntentSwitchProject IntentType = "switch-project" // Switch project
    IntentCancel        IntentType = "cancel"        // Cancel current flow
)

type Intent struct {
    Type IntentType
    
    // Raw input
    RawText string
    
    // Parsed parameters
    Params map[string]string
}

// Intent recognition
// - Slash commands: explicit (e.g., /create-issue)
// - Natural language: all treated as discuss
```

**Intent Recognition Rules**:

| Input | Intent | Notes |
|-------|--------|-------|
| `/create-issue` | IntentCreateIssue | Explicit command |
| `/start-dev` or `/start` | IntentStartDev | Explicit command |
| `/ai-review` | IntentAIReview | Explicit command |
| `/review` | IntentHumanReview | Human begins manual review |
| `/revise` | IntentRevise | Human requests fixes |
| `/merge` | IntentMerge | Human approves merge |
| `/cancel` | IntentCancel | Cancel current flow |
| `/switch <project>` | IntentSwitchProject | Switch project context |
| Natural language (requirements) | IntentDiscuss | AI infers from content |
| Natural language (correction) | IntentDiscuss (correction) | AI recognizes as modification |
| "确认" or "OK" | IntentApprove | Approve current state |

**Intent Parser Behavior**:

```go
func (p *Parser) Parse(input string) *Intent {
    // 1. Check for slash command
    if strings.HasPrefix(input, "/") {
        return p.parseSlashCommand(input)
    }
    
    // 2. Check for approval phrases
    if isApprovalPhrase(input) {
        return &Intent{Type: IntentApprove, Confidence: 0.95}
    }
    
    // 3. Check for correction phrases
    if isCorrectionPhrase(input) {
        return &Intent{Type: IntentDiscuss, Params: map[string]string{"correction": "true"}}
    }
    
    // 4. Default to discuss intent
    return &Intent{Type: IntentDiscuss, Confidence: 0.7}
}

func isCorrectionPhrase(input string) bool {
    // "不对", "不是", "改成", "修改", "错了", "但是..."
    patterns := []string{"不对", "不是", "改成", "修改", "错了", "但是", "再", "还", "要加"}
    for _, p := range patterns {
        if strings.Contains(input, p) {
            return true
        }
    }
    return false
}
```

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

#### PM Agent: Requirements Discussion Workflow

The PM Agent implements the **iterative refinement loop**:

```go
// team/pm.go
type PMAgent struct {
    llm      llm.Provider
    display  Display
    github   GitHubTool
}

// Discuss implements the iterative refinement loop:
// 1. Human states requirement
// 2. AI paraphrases and presents understanding
// 3. Human corrects if needed
// 4. Repeat until confirmed
func (a *PMAgent) Discuss(ctx context.Context, input string, discussion *Discussion) (*Result, error) {
    intent := parseIntent(input)
    
    switch intent.Type {
    case IntentDiscuss:
        // Add human statement to discussion
        discussion.AddMessage(Human, input, Statement)
        
        // AI analyzes and presents understanding
        understanding := a.analyzeRequirements(discussion)
        
        // Display to Human for review
        return &Result{
            Display: a.display.Requirements(understanding),
            Continue: true,  // Wait for human feedback
        }
        
    case IntentApprove:
        // Mark discussion as confirmed
        discussion.Confirm()
        return &Result{
            Display: a.display.Confirmed(discussion),
            Continue: false,
        }
        
    case IntentCreateIssue:
        // Create GitHub Issue from confirmed discussion
        issue := a.createIssue(discussion)
        return &Result{
            Artifact: issue,
            Continue: false,
        }
    }
}

// analyzeRequirements generates a structured understanding
func (a *PMAgent) analyzeRequirements(discussion *Discussion) *RequirementsUnderstanding {
    prompt := fmt.Sprintf(`
Based on the following discussion, extract and structure the requirements:

%s

Format the output as:
- 功能：
- 范围：
- 限制：
- 实现方式：
- 验收标准：
`, discussion.Format())
    
    response := a.llm.Chat(ctx, prompt)
    return parseRequirements(response)
}
```

#### Dev Agent: Development Workflow

```go
// team/dev.go
type DevAgent struct {
    llm         llm.Provider
    claudeCode  ClaudeCodeTool
    terminal    TerminalTool
    file        FileTool
    github      GitHubTool
}

// StartDev begins development from an Issue
func (a *DevAgent) StartDev(ctx context.Context, issue *Issue) (*Result, error) {
    // 1. Create feature branch
    branch := fmt.Sprintf("feature/%d-%s", issue.Number, slug(issue.Title))
    a.git.CreateBranch(branch)
    
    // 2. Analyze Issue requirements
    tasks := a.decomposeIssue(issue)
    
    // 3. Execute each task (using Claude Code)
    for _, task := range tasks {
        result := a.executeTask(ctx, task)
        if result.Error != nil {
            return result, result.Error
        }
    }
    
    // 4. Create PR
    pr := a.github.CreatePR(&PR{
        Title:     issue.Title,
        Body:      a.generatePRDescription(issue),
        Head:      branch,
        Base:      "main",
        IssueRefs: []int{issue.Number},
    })
    
    return &Result{
        Artifact: pr,
        State:    FlowPROpen,
    }
}
```

#### Reviewer Agent: Code Review Workflow

```go
// team/reviewer.go
type ReviewerAgent struct {
    llm       llm.Provider
    github    GitHubTool
    codeAnalysis CodeAnalysisTool
}

// Review performs AI code review on a PR
func (a *ReviewerAgent) Review(ctx context.Context, pr *PullRequest) (*Result, error) {
    // 1. Get PR diff
    diff := a.github.GetPRDiff(pr)
    
    // 2. Analyze code changes
    analysis := a.codeAnalysis.Analyze(diff)
    
    // 3. Generate review comments
    review := a.llm.Chat(ctx, fmt.Sprintf(`
Review the following code changes for:
1. Correctness
2. Security issues
3. Performance problems
4. Code style violations
5. Missing tests

Changes:
%s
`, diff))
    
    // 4. Post review to GitHub
    a.github.PostReview(pr, review)
    
    return &Result{
        Display: a.display.ReviewResult(review),
    }
}
```

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

#### Discussion Artifact

The Discussion artifact tracks the **iterative refinement loop**:

```go
// artifact/discussion.go
type Discussion struct {
    ID        DiscussionID
    ProjectID ProjectID
    
    // Messages in the discussion
    Messages []DiscussionMessage
    
    // Current understanding (latest AI paraphrase)
    CurrentUnderstanding *RequirementsUnderstanding
    
    // Version history
    Versions []DiscussionVersion
    
    // Status
    Status DiscussionStatus  // active | confirmed | cancelled
    
    // Confirmed version number (if Status == confirmed)
    ConfirmedVersion *int
}

type DiscussionMessage struct {
    ID        string
    Role      string    // "human" | "ai"
    Type      string    // "statement" | "paraphrase" | "correction" | "confirmation"
    Content   string
    Timestamp time.Time
}

type RequirementsUnderstanding struct {
    Version int
    
    // Structured fields
    Features     []string
    Scope        string
    Constraints  []string
    Implementation string
    AcceptanceCriteria []string
    
    // Raw summary
    Summary string
}

// AddMessage adds a message and updates understanding
func (d *Discussion) AddMessage(role, content, msgType string) {
    d.Messages = append(d.Messages, DiscussionMessage{
        ID:        generateID(),
        Role:      role,
        Type:      msgType,
        Content:   content,
        Timestamp: time.Now(),
    })
    
    // If this is a human statement, AI should update understanding
    if role == "human" {
        d.CurrentUnderstanding = nil  // Will be regenerated by AI
    }
}

// CreateVersion snapshots current state
func (d *Discussion) CreateVersion() *DiscussionVersion {
    d.Version++
    v := &DiscussionVersion{
        Version:   d.Version,
        Understanding: d.CurrentUnderstanding,
        Timestamp: time.Now(),
    }
    d.Versions = append(d.Versions, *v)
    return v
}

// Confirm marks the discussion as confirmed
func (d *Discussion) Confirm() {
    d.Status = DiscussionStatusConfirmed
    d.ConfirmedVersion = &d.Version
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

**Purpose**: Format and present AI results to Human. This is the **Result Display Page** mentioned in the workflow.

```go
// display/display.go
type Display interface {
    // Render requirements understanding (for discussion loop)
    Requirements(content *RequirementsUnderstanding) DisplayBlock
    
    // Render Issue draft
    IssueDraft(issue *Artifact) DisplayBlock
    
    // Render PR description
    PRDescription(pr *Artifact) DisplayBlock
    
    // Render Code Review result
    ReviewResult(review *Review) DisplayBlock
    
    // Render confirmation
    Confirmed(artifact *Artifact) DisplayBlock
    
    // Render error
    Error(err error) DisplayBlock
}

type DisplayBlock struct {
    Type    string      // "markdown" | "card" | "buttons" | "table"
    Content interface{}
}
```

**Display Templates**:

#### Requirements Understanding Display

```
┌────────────────────────────────────────────────────────────────┐
│  📋 需求理解结果 (v2)                                           │
│  ─────────────────────────────────────────────────────────────  │
│                                                                │
│  功能：给所有 API 添加 rate limiting                           │
│                                                                │
│  范围：                                                         │
│    • POST /api/v1/users                                       │
│    • POST /api/v1/orders                                       │
│    • GET /api/v1/products                                       │
│                                                                │
│  限制：                                                         │
│    • 200 req/min per user  ✓ (updated from 100)               │
│    • 1000 req/min per IP                                       │
│                                                                │
│  实现方式：Redis + sliding window                              │
│                                                                │
│  验收标准：                                                     │
│    • [ ] 超过限制返回 429                                      │
│    • [ ] 有 X-RateLimit-* header                               │
│    • [ ] 单元测试覆盖率 > 80%                                   │
│                                                                │
│  ─────────────────────────────────────────────────────────────  │
│  [确认 ✓]  [修改 ✎]                                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### Issue Draft Display

```
┌────────────────────────────────────────────────────────────────┐
│  📝 Issue 草稿                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                │
│  Title: [feature] API Rate Limiting                           │
│                                                                │
│  Labels: [feature] [api]                                      │
│                                                                │
│  Body:                                                         │
│  ## 功能                                                        │
│  给所有 API 添加 rate limiting...                              │
│                                                                │
│  ## 验收标准                                                    │
│  - [ ] 超过限制返回 429                                        │
│  - [ ] 有 X-RateLimit-* header                                 │
│                                                                │
│  ─────────────────────────────────────────────────────────────  │
│  [创建 Issue ✓]  [取消]                                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### Code Review Result Display

```
┌────────────────────────────────────────────────────────────────┐
│  🔍 Code Review 结果                                            │
│  ─────────────────────────────────────────────────────────────  │
│                                                                │
│  PR: #456 feature/rate-limiting → main                        │
│  Status: ✅ Approved / ⚠️ Changes Requested                     │
│                                                                │
│  AI Review:                                                    │
│  ─────────────────────────────────                            │
│  ✅ 逻辑正确，Redis key 设计合理                                │
│  ⚠️ line 23: 建议添加熔断逻辑                                   │
│  ⚠️ line 45: 缺少错误日志                                       │
│  ❌ line 67: 硬编码的阈值应该配置化                             │
│                                                                │
│  手动 Review:                                                   │
│  ─────────────────────────────────                            │
│  ⬜ 请检查业务逻辑是否满足需求                                   │
│  ⬜ 请确认错误处理                                              │
│                                                                │
│  ─────────────────────────────────────────────────────────────  │
│  [确认并合并 ✓]  [要求修改 ✎]  [关闭 PR]                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
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
│  "改成 200" → IntentDiscuss (correction)                                        │
│  "确认" → IntentApprove                                                          │
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
│  │  Execute: Requirements analysis, Issue drafting, Display results        │   │
│  │                                                                           │   │
│  │  Discuss Loop:                                                           │   │
│  │    1. Human: "我要加 rate limiting"                                       │   │
│  │    2. AI → Display.Requirements(understanding)  ← Presented             │   │
│  │    3. Human: "限制改成 200"                                              │   │
│  │    4. AI → Display.Requirements(updated)  ← Presented                    │   │
│  │    5. Human: "确认"                                                      │   │
│  │    6. AI → /create-issue                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                            Dev Agent                                      │   │
│  │  Role: dev | Tools: terminal, file, github, claude-code                    │   │
│  │  Execute: Code implementation, PR creation                                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         Reviewer Agent                                    │   │
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
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        Result Display Page                               │   │
│  │                                                                          │   │
│  │  ┌──────────────────────────────────────────────────────────────────┐   │   │
│  │  │  📋 需求理解结果 (v2)                                            │   │   │
│  │  │  ────────────────────────────────────────────────────────────    │   │   │
│  │  │  功能：给所有 API 添加 rate limiting                             │   │   │
│  │  │  限制：200 req/min per user ✓ (updated)                        │   │   │
│  │  │  ────────────────────────────────────────────────────────────    │   │   │
│  │  │  [确认 ✓]  [修改 ✎]                                            │   │   │
│  │  └──────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  Markdown / Card / Buttons                                                       │
│  Back to Human via Channel                                                      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Command Specification

### 5.1 Slash Commands

| Command | Intent | Action | Notes |
|---------|--------|--------|-------|
| `/switch <project>` | switch-project | Bind session to another project | |
| `/create-issue` | create-issue | Create GitHub Issue from discussion | |
| `/start-dev` 或 `/start` | start-dev | Start development (PM → Dev handoff) | |
| `/ai-review` | ai-review | Run AI code review | 在 PR 上运行 AI 审查 |
| `/revise` | revise | Return PR to dev for fixes | |
| `/merge` | merge | Merge PR to main | |
| `/cancel` | cancel | Cancel current flow | |

**注意**：
- Human 的 code review 直接在 GitHub PR 页面进行，不需要 slash command
- `/merge` 是确认 Human 审查通过后执行的合并操作

### 5.2 Intent Recognition

**核心原则**：所有关键节点必须通过明确的 slash command 触发，自然语言只用于讨论需求。

```go
func (p *Parser) Parse(input string) *Intent {
    // 1. 检查 slash command（精确匹配）
    if strings.HasPrefix(input, "/") {
        return p.parseSlashCommand(input)
    }
    
    // 2. 其他全部视为需求讨论
    return &Intent{Type: IntentDiscuss, Confidence: 1.0}
}
```

**自然语言的唯一作用**：讨论需求（Discuss）。Human 和 AI 之间的所有对话都是需求澄清/确认的过程，直到 Human 决定发起下一个 slash command。

**注意**：`/review` 和 `/merge` 等命令需要 Human 在 GitHub/Web 上亲自操作或确认，而不是通过"可以"、"没问题"这类自然语言。

### 5.3 Discussion Loop Flow

核心交互模式：Human 和 AI 进行需求讨论，直到 Human 发起明确的 slash command。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Requirements Discussion Loop                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Human: "我要给 API 加 rate limiting"                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                          │
│                                    ▼                                          │
│                         Intent: IntentDiscuss                                │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  PM Agent.AnalyzeRequirements()                                     │    │
│  │                                                                     │    │
│  │  → RequirementsUnderstanding{                                       │    │
│  │      Features: ["API rate limiting"],                               │    │
│  │      Constraints: ["100 req/min per user"],                         │    │
│  │      Implementation: "Redis + sliding window"                      │    │
│  │    }                                                                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Display.Requirements() → [Result Display Page]                    │    │
│  │                                                                     │    │
│  │  ┌───────────────────────────────────────────────────────────────┐  │    │
│  │  │  功能：给所有 API 添加 rate limiting                          │  │    │
│  │  │  限制：100 req/min per user                                   │  │    │
│  │  │  实现：Redis + sliding window                                 │  │    │
│  │  └───────────────────────────────────────────────────────────────┘  │    │
│  │                                                                     │    │
│  │  Human 确认理解正确后，发起下一个命令                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                          │
│                     ┌──────────────┴──────────────┐                         │
│                     │                               │                         │
│          Human: "限制改成 200"          Human: /create-issue                 │
│                     │                               │                         │
│                     ▼                               ▼                         │
│          Intent: IntentDiscuss            Intent: IntentCreateIssue           │
│          (correction=true)                                                  │
│                     │                               │                         │
│                     ▼                               ▼                         │
│          ┌─────────────────────┐          ┌─────────────────────────┐    │
│          │ Update + Re-display │          │ Create GitHub Issue      │    │
│          └─────────────────────┘          │ Flow → issue-created     │    │
│                                             └─────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 Full Command Flow Examples

**Requirements Discussion Flow**:
```
1. Human: "我要给 API 加 rate limiting"
   → Intent: discuss
   → PM Agent analyzes requirements
   → Display: AI presents understanding

2. Human: "限制改成 200 req/min"
   → Intent: discuss
   → PM Agent updates understanding
   → Display: AI shows updated understanding

3. Human: /create-issue
   → Intent: create-issue
   → PM Agent creates GitHub Issue
   → Flow: discussing → issue-created
```

**Development Flow**:
```
1. Human: /start-dev
   → Intent: start-dev
   → Flow: issue-created → in-dev
   → Dev Agent starts implementation
   → Dev Agent creates PR
   → Flow: in-dev → pr-open → reviewing
   → Display: PR opened for Human review

2. Human: (在 GitHub 上进行 code review)

3. Human: /revise
   → Intent: revise
   → Flow: reviewing → in-dev
   → Dev Agent fixes issues
   → Flow: in-dev → reviewing

4. Human: /merge
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
                    │           │ Human: /create-issue                         │
                    │           ▼                                              │
                    │     ┌───────────────┐                                    │
                    │     │ issue-created │ ←── GitHub Issue created          │
                    │     └───────┬───────┘                                    │
                    │             │                                             │
                    │             │ Human: /start-dev                          │
                    │             ▼                                              │
                    │     ┌─────────────┐                                      │
                    │     │  in-dev     │ ←── Dev Agent working                │
                    │     └──────┬──────┘                                      │
                    │            │                                              │
                    │            │ Dev Agent: PR created                        │
                    │            ▼                                              │
                    │     ┌─────────────┐                                      │
                    │     │  pr-open   │ ←── PR submitted                      │
                    │     └──────┬──────┘                                      │
                    │            │                                              │
                    │            │ AI Review complete (auto or /ai-review)     │
                    │            ▼                                              │
                    │     ┌─────────────┐                                      │
    Human:          │     │ reviewing   │                                      │
    /revise ────────┼──→  └──────┬──────┘                                      │
                        │            │                                             │
                        │            │ Human: /merge                              │
                        │            ▼                                             │
                        │     ┌─────────────┐                                      │
                        │     │  merged    │ ←── Code merged to main             │
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

**触发类型说明**：
- `human`：必须通过 slash command 触发
- `ai`：系统在满足条件时自动执行

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
    
    // Current understanding (regenerated on each iteration)
    CurrentUnderstanding *RequirementsUnderstanding
    
    // Status
    Status   DiscussionStatus  // active | confirmed | cancelled
    ConfirmedVersion *int       // Pointer to confirmed version number
    
    CreatedAt time.Time
    UpdatedAt time.Time
}

type DiscussionMessage struct {
    Role    string    // "human" | "ai"
    Type    string    // "statement" | "paraphrase" | "correction" | "confirmation"
    Content string
    Timestamp time.Time
}

type RequirementsUnderstanding struct {
    Version int
    
    // Structured fields
    Features           []string
    Scope              string
    Constraints        []string
    Implementation     string
    AcceptanceCriteria []string
    
    // Raw summary
    Summary string
    
    // Change tracking
    UpdatedFields []string  // Which fields were updated in this version
}

type DiscussionVersion struct {
    Version         int
    Understanding  *RequirementsUnderstanding
    CreatedAt       time.Time
    IsConfirmed     bool
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
    
    // Requirements (from Discussion)
    Requirements *RequirementsUnderstanding
    
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

### 9.2 Why Iterative Refinement?

AI cannot perfectly understand requirements in one pass. The iterative loop:

```
Human states → AI paraphrases → Human corrects → AI updates → ... → Confirm
```

This ensures:
- **Accuracy**: Requirements are precisely captured
- **Alignment**: AI's understanding matches Human's intent
- **Audit Trail**: Every correction is tracked

### 9.3 Why Structured Commands?

Natural language is ambiguous. "start the thing" could mean:
- Start development
- Start a new discussion
- Resume a paused task

Slash commands (`/start-dev`) make intent explicit and unambiguous.

### 9.4 Why Milestone-Based?

Not every step needs human approval (that defeats the purpose of automation). But critical milestones do:
- Requirements confirmed → prevents wasted development
- Code reviewed → maintains quality
- Deployment approved → prevents broken main

### 9.5 Why Artifact Versioning?

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
