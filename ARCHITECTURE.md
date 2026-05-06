# Humera Technical Architecture

## 1. Architecture Overview

Humera 是一个 **Human 掌控的 AI Agent**，核心原则：
- Human 发起所有任务（通过 slash command）
- AI 执行具体工作
- 所有关键决策由 Human 做出

### 1.1 AI Agent 核心组件

一个完整的 AI Agent 必须包含以下组件：

| 组件 | 职责 | Humera 实现 |
|------|------|------------|
| **Channel** | 消息收发（IM 平台接入） | Telegram/Feishu/CLI/WebSocket |
| **Command Parser** | 解析 slash command | `/on`, `/new`, `/tdd`, `/fix`, `/review`, `/revise` |
| **Agent Loop** | 核心控制流：LLM → 工具 → LLM | 循环调用 LLM，执行 tool_calls |
| **Dispatcher** | 命令分发到 Handler | 根据 CommandType 路由 |
| **LLM Provider** | AI 推理能力 | OpenAI/Anthropic 等 |
| **Tool System** | 执行具体操作 | GitHub API, Coding Agent, File System |
| **Session Memory** | 当前对话上下文 | 讨论内容、讨论进度 |
| **Project Context** | 项目信息 | workdir, repo, branch |
| **Task Handlers** | 业务逻辑处理 | Discuss, TDD, Fix, Review, Revise |

### 1.2 System Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Human (IM Channel)                             │
│                                                                          │
│   /on ~/code/myapp                                                     │
│   [any message]  ← Discuss 模式                                         │
│   /new                                                            │
│   /tdd 123                                                         │
│   /fix 123                                                         │
│   /review 456                                                       │
│   /revise 456                                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Message
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              Channel Layer                               │
│                                                                          │
│   Responsibility:                                                        │
│   - 连接 IM 平台（Telegram/Feishu/Discord/WhatsApp/CLI/WebSocket）       │
│   - 消息格式标准化                                                       │
│   - 返回格式化结果给 Human                                               │
│                                                                          │
│   Interface:                                                             │
│   - Receive() *Message                                                   │
│   - Send(Display) error                                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Message
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Agent Core                                     │
│                                                                          │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │
│   │  Command Parser │  │   Dispatcher    │  │ Session Memory  │       │
│   │                 │  │                  │  │                 │       │
│   │ - Parse slash   │  │ - Route to       │  │ - Current msg   │       │
│   │   command       │  │   Handler        │  │ - Discuss hist  │       │
│   │ - Extract args  │  │ - No biz logic  │  │ - Project ctx   │       │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘       │
│                                    │                                    │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      Agent Loop                                  │   │
│   │                                                                  │   │
│   │   while (tool_calls exist):                                     │   │
│   │       1. LLM.Chat() → response + tool_calls                     │   │
│   │       2. Execute each tool_call                                 │   │
│   │       3. Append results to messages                             │   │
│   │       4. Continue                                               │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                        LLM Provider                              │   │
│   │                                                                  │   │
│   │   Responsibility:                                               │   │
│   │   - Chat (讨论、推理、生成)                                       │   │
│   │   - Code Review                                                  │   │
│   │                                                                  │   │
│   │   Interface:                                                     │   │
│   │   - Chat(ctx, messages) (string, error)                         │   │
│   │   - Review(ctx, diff) (*ReviewResult, error)                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                       Tool System                                │   │
│   │                                                                  │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │   │
│   │   │  GitHub API  │  │ Coding Agent │  │ File System  │         │   │
│   │   │              │  │              │  │              │         │   │
│   │   │ - Issue CRUD │  │ - Claude Code│  │ - Read/Write │         │   │
│   │   │ - PR CRUD    │  │ - OpenCode    │  │ - expandPath │         │   │
│   │   │ - Comments   │  │              │  │              │         │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     Task Handlers                                │   │
│   │                                                                  │   │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐│   │
│   │   │ Discuss │  │   TDD   │  │   Fix   │  │ Review  │  │ Revise ││   │
│   │   │         │  │         │  │         │  │         │  │        ││   │
│   │   │ - Chat  │  │ - Design│  │ - Impl  │  │ - Review│  │ - Fix  ││   │
│   │   │   with  │  │ - Stub  │  │   logic │  │   code  │  │   feed ││   │
│   │   │   LLM   │  │ - Test  │  │         │  │         │  │  back  ││   │
│   │   └─────────┘  └─────────┘  └─────────┘  └─────────┘  └────────┘│   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Result
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Display Layer                                  │
│                                                                          │
│   Responsibility:                                                        │
│   - 格式化输出给 Human                                                   │
│   - 支持多平台格式（Telegram/Feishu/Markdown/CLI）                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Agent Loop

Agent Loop 是 Agent 的核心控制流：

```go
// agent/loop.go

type AgentLoop struct {
    llm        LLMProvider
    tools      []Tool
    maxIter    int
}

func (a *AgentLoop) Run(ctx context.Context, messages []Message) (string, error) {
    iter := 0
    for iter < a.maxIter {
        // 1. LLM 生成回复 + tool_calls
        resp, err := a.llm.Chat(ctx, messages)
        if err != nil {
            return "", err
        }

        // 2. 如果没有 tool_calls，返回文本回复
        if len(resp.ToolCalls) == 0 {
            return resp.Content, nil
        }

        // 3. 执行 tool_calls
        for _, tc := range resp.ToolCalls {
            result, err := a.executeTool(tc)
            if err != nil {
                // 工具执行失败，返回错误信息
                messages = append(messages, Message{
                    Role:    "tool",
                    Content: fmt.Sprintf("Error: %v", err),
                })
            } else {
                messages = append(messages, Message{
                    Role:    "tool",
                    Content: result,
                })
            }
        }

        iter++
    }

    return "", ErrMaxIterations
}

func (a *AgentLoop) executeTool(tc ToolCall) (string, error) {
    for _, tool := range a.tools {
        if tool.Name() == tc.Name {
            return tool.Execute(tc.Args)
        }
    }
    return "", fmt.Errorf("unknown tool: %s", tc.Name)
}
```

**Tool 接口：**

```go
// agent/tool.go

type ToolCall struct {
    Name string
    Args map[string]any
}

type Tool interface {
    Name() string
    Description() string
    Execute(args map[string]any) (string, error)
}

// 注册到 AgentLoop 的工具：
// - GitHubIssueTool     (GetIssue, CreateIssue)
// - GitHubPRTool        (CreatePR, UpdatePR, GetPR)
// - GitHubCommentTool   (PostComment, GetComments)
// - CodingTool          (TDD, Implement, Revise)
// - FileSystemTool      (Read, Write, expandPath)
```

### 1.4 Nested Agent Loop (Supervisor Pattern)

当 Humera 委托 Claude Code / OpenCode 时，被委托的 CLI **本身也是一个 Agent Loop**。我们称之为 **Coder Agent**。

Humera 和 Coder Agent 的关系是 **Supervisor - Worker** 模式：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Humera Agent Loop                                 │
│                                                                          │
│   Handler (TDD/Fix/Revise)                                               │
│        │                                                                │
│        │ spawn Coder Agent (Claude Code / OpenCode)                     │
│        ▼                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                  Coder Agent Loop                                │   │
│   │                                                                  │   │
│   │   while (iter < max_iter):                                      │   │
│   │       1. LLM generates code                                     │   │
│   │       2. Execute tools (git, file, shell)                       │   │
│   │       3. Return output                                          │   │
│   │                                                                  │   │
│   │   Tools: Read, Write, Bash, Git, Search                         │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│        │                                                                │
│        │ output                                                          │
│        ▼                                                                │
│   Humera 检查结果是否符合要求                                             │
│        │                                                                │
│        ├── 符合 → 继续下一步                                              │
│        └── 不符合 → 反馈给 Coder Agent，重新生成                          │
│                    (继续 Coder Agent Loop)                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Coder 接口设计：**

```go
// coder/coder.go

// Coder 是对外部 Agent CLI 的封装
// 它内部有自己的 Agent Loop，但我们需要：
// 1. 验证输出是否符合要求
// 2. 如果不符合，迭代沟通

type Coder interface {
    // Run 执行一次完整的 coding 任务
    // 内部可能有多轮 LLM ↔ 工具 的循环
    // 返回最终结果
    Run(ctx context.Context, req *CodingRequest) (*CodingResult, error)
}

type CodingRequest struct {
    Task     string       // 任务描述
    Context  string       // 上下文（Issue 内容、讨论历史）
    Workdir  string       // 工作目录
    Mode     CodingMode   // stub | implement | revise
}

type CodingMode string

const (
    ModeStub      CodingMode = "stub"      // TDD: stub + 测试
    ModeImplement CodingMode = "implement" // 实现逻辑
    ModeRevise    CodingMode = "revise"    // 根据反馈修正
)

type CodingResult struct {
    Success   bool
    Output    string       // 最终输出
    Diff      string       // 代码变更
    Error     error
    IterCount int          // 迭代次数
}

// Validator 验证输出是否符合要求
type Validator interface {
    Validate(req *CodingRequest, result *CodingResult) *ValidationResult
}

type ValidationResult struct {
    Pass    bool
    Feedback string  // 不通过的原因，用于反馈给 Coder
}

// CoderAgent 包含嵌套的 Agent Loop
type CoderAgent struct {
    cli       CoderCLI           // Claude Code / OpenCode
    validator Validator
    maxRetry  int
}

func (c *CoderAgent) Run(ctx context.Context, req *CodingRequest) (*CodingResult, error) {
    // 1. 执行 Coder
    result, err := c.cli.Run(ctx, req)
    if err != nil {
        return nil, err
    }

    // 2. 验证结果
    validation := c.validator.Validate(req, result)
    if validation.Pass {
        return result, nil
    }

    // 3. 不通过，反馈给 Coder 重试
    for i := 0; i < c.maxRetry; i++ {
        result, err = c.cli.Revise(ctx, req, validation.Feedback)
        if err != nil {
            return nil, err
        }

        validation = c.validator.Validate(req, result)
        if validation.Pass {
            return result, nil
        }
    }

    // 4. 超过重试次数，返回失败
    return result, fmt.Errorf("validation failed after %d retries: %s", c.maxRetry, validation.Feedback)
}

// CoderCLI 是对外部 CLI 的封装
type CoderCLI interface {
    // Run 执行一次任务
    Run(ctx context.Context, req *CodingRequest) (*CodingResult, error)

    // Revise 根据反馈修正
    Revise(ctx context.Context, req *CodingRequest, feedback string) (*CodingResult, error)
}

// ClaudeCodeCLI 实现
type ClaudeCodeCLI struct {
    model string
}

func (c *ClaudeCodeCLI) Run(ctx context.Context, req *CodingRequest) (*CodingResult, error) {
    // 调用 claude --print 执行任务
    cmd := exec.CommandContext(ctx, "claude",
        "--print",
        "--model", c.model,
        "--prompt", buildPrompt(req),
    )
    cmd.Dir = req.Workdir

    output, err := cmd.CombinedOutput()
    return &CodingResult{
        Success:   err == nil,
        Output:    string(output),
        IterCount: 1, // Claude Code 内部自己有多轮 loop
    }, err
}

func (c *ClaudeCodeCLI) Revise(ctx context.Context, req *CodingRequest, feedback string) (*CodingResult, error) {
    // 调用 claude --print 执行修正
    cmd := exec.CommandContext(ctx, "claude",
        "--print",
        "--model", c.model,
        "--prompt", buildRevisePrompt(req, feedback),
    )
    cmd.Dir = req.Workdir

    output, err := cmd.CombinedOutput()
    return &CodingResult{
        Success:   err == nil,
        Output:    string(output),
        IterCount: 1,
    }, err
}
```

---

## 2. Module Boundaries

### 2.1 Module List

| Module | Layer | Responsibility | Public API |
|--------|-------|---------------|------------|
| **Channel** | Input/Output | IM 平台接入 | `Receive()`, `Send()` |
| **Command** | Core | 解析 slash command | `Parse(Message) *Command` |
| **Agent Loop** | Core | 核心控制流：LLM → 工具 → LLM | `Run(ctx, messages)` |
| **Tool System** | Core | 工具注册和执行 | `Tool`, `Register()` |
| **Dispatcher** | Core | 命令分发 | `Dispatch(Command, *Session) *Result` |
| **Session** | Core | 会话上下文管理 | `AddMessage()`, `GetHistory()`, `GetProject()` |
| **LLM** | Core | AI 推理 | `Chat()`, `Review()` |
| **DiscussHandler** | Handler | 需求讨论 | `Handle(ctx, msg, session) *Result` |
| **TDDHandler** | Handler | 架构设计 + stub + 测试 | `Handle(ctx, issue, session) *Result` |
| **FixHandler** | Handler | 逻辑实现 | `Handle(ctx, issue, session) *Result` |
| **ReviewHandler** | Handler | 代码审查 | `Handle(ctx, pr, session) *Result` |
| **ReviseHandler** | Handler | 代码修正 | `Handle(ctx, pr, session) *Result` |
| **GitHub** | Tool | GitHub API | `GetIssue()`, `CreateIssue()`, `CreatePR()`, etc. |
| **Coder** | Tool | 嵌套 Agent Loop | `Run()`, `Revise()`, `Validate()` |
| **PathResolver** | Tool | 路径解析 | `Resolve(path) string` |

### 2.2 Dependency Rules

```
Channel → Command → Dispatcher → [Handler Layer]
                              ↓
                         Agent Loop ← Tools
                              ↓
                         LLM Provider
```

**依赖方向规则：**
- 上层可以依赖下层
- 下层不能依赖上层
- 同层之间不允许循环依赖

**Tool 注册到 Agent Loop：**
- Handler 不直接调用 Tool
- Handler 通过 Agent Loop 调用 Tool（通过 messages）
- Tool 执行结果通过 tool result message 返回

### 2.3 数据流

```
Human Message
    │
    ▼
Channel.Receive()
    │
    ▼
Command.Parse(message)
    │
    ▼
Dispatcher.Dispatch(cmd, session)
    │
    ├── nil (普通消息) → DiscussHandler
    ├── /on           → ProjectHandler (设置上下文)
    ├── /new          → DiscussHandler (创建 Issue)
    ├── /tdd          → TDDHandler
    ├── /fix          → FixHandler
    ├── /review       → ReviewHandler
    └── /revise       → ReviseHandler
    │
    ▼
Handler.Handle()
    │
    ▼
Agent Loop.Run(ctx, messages)
    │
    ├── LLM.Chat() → response + tool_calls
    ├── Execute tool_calls
    ├── Append results to messages
    └── Loop until no tool_calls
    │
    ▼
Result.Display
    │
    ▼
Channel.Send(display)
    │
    ▼
Human
```

---

## 3. Module Interfaces

### 3.1 Channel

```go
// channel/channel.go

// Message 接收自 Human
type Message struct {
    Raw       string                 // 原始消息
    Platform  string                 // telegram, feishu, cli, ws
    ChatID    string                 // 聊天 ID
    UserID    string                 // 用户 ID
    Timestamp time.Time              // 时间戳
}

// Display 发送给 Human
type Display struct {
    Type    DisplayType              // text, code, image, file
    Content string
    Metadata map[string]string       // platform-specific
}

type DisplayType string

const (
    DisplayText  DisplayType = "text"
    DisplayCode  DisplayType = "code"
    DisplayImage DisplayType = "image"
    DisplayFile  DisplayType = "file"
)

type Channel interface {
    // Receive 接收 Human 消息
    Receive(ctx context.Context) (*Message, error)

    // Send 发送消息给 Human
    Send(ctx context.Context, display Display) error

    // Start 启动监听
    Start(ctx context.Context, handler MessageHandler) error

    // Stop 停止监听
    Stop(ctx context.Context) error
}

type MessageHandler interface {
    Handle(ctx context.Context, msg *Message) (*Result, error)
}
```

### 3.2 Command

```go
// command/command.go

type CommandType string

const (
    CmdOn      CommandType = "on"
    CmdNew     CommandType = "new"
    CmdTDD     CommandType = "tdd"
    CmdFix     CommandType = "fix"
    CmdReview  CommandType = "review"
    CmdRevise  CommandType = "revise"
)

type Command struct {
    Type   CommandType
    Raw    string                 // 原始命令
    Params string                  // 命令参数（issue/pr URL 或 number）
}

type Parser interface {
    // Parse 解析消息为命令
    // 如果不是 slash command，返回 nil（由 Discuss Handler 处理）
    Parse(msg *Message) *Command
}
```

### 3.3 Session

```go
// session/session.go

// DiscussMessage 讨论消息
type DiscussMessage struct {
    Role    string                 // "human" | "ai"
    Content string
    Time    time.Time
}

// Project 项目上下文
type Project struct {
    Workdir string                 // /Users/bin/code/myapp
    Owner   string                 // owner
    Repo    string                 // repo
    Branch  string                 // main
}

// DiscussContext 讨论上下文
type DiscussContext struct {
    Project *Project
    Messages []DiscussMessage       // 讨论历史
}

// Session 会话管理
type Session interface {
    // 项目上下文
    SetProject(project *Project)
    GetProject() *Project

    // 讨论上下文
    AddDiscussMessage(msg *DiscussMessage)
    GetDiscussHistory() []*DiscussMessage
    ClearDiscussHistory()

    // Session 持久化
    Save() error
    Load(chatID string) error
}
```

### 3.4 LLM Provider

```go
// llm/llm.go

// Message 对话消息
type Message struct {
    Role    string                 // "system", "user", "assistant"
    Content string
}

// ReviewResult 审查结果
type ReviewResult struct {
    Comments []*ReviewComment
    Summary  string
}

type LLMProvider interface {
    // Chat 对话
    Chat(ctx context.Context, messages []Message) (string, error)

    // Review 代码审查
    Review(ctx context.Context, diff string) (*ReviewResult, error)
}
```

### 3.5 Path Resolver

```go
// path/resolver.go

// 遵循 GTW expandPath 逻辑
// 支持格式：
//   /Users/bin/code/myapp     → 直接使用
//   ~/code/myapp              → 替换为 homedir
//   ~user/code/myapp           → 替换为该用户的 home
//   ./code/myapp               → 相对当前工作目录
//   ../code/myapp              → 相对当前工作目录
//   code/myapp                 → 默认视为 ~/code/myapp

type Resolver interface {
    Resolve(path string) (string, error)  // 返回绝对路径
}
```

### 3.6 GitHub Client

```go
// github/github.go

type Issue struct {
    Number    int
    Title     string
    Body      string
    State     string
    URL       string
    Labels    []string
}

type PR struct {
    Number     int
    Title      string
    Body       string
    State      string
    Head       string                 // branch name
    Base       string                 // target branch
    URL        string
    IsDraft    bool
}

type ReviewComment struct {
    ID        int
    Author    string
    Body      string
    Path      string
    Line      int
    CommitID  string
}

type GitHubClient interface {
    // Issue
    GetIssue(ctx context.Context, owner, repo string, number int) (*Issue, error)
    CreateIssue(ctx context.Context, owner, repo string, title, body string) (*Issue, error)

    // PR
    GetPR(ctx context.Context, owner, repo string, number int) (*PR, error)
    CreatePR(ctx context.Context, owner, repo, title, body, head, base string, draft bool) (*PR, error)
    UpdatePR(ctx context.Context, owner, repo string, number int, title, body string) error

    // Comments
    GetIssueComments(ctx context.Context, owner, repo string, number int) ([]*ReviewComment, error)
    GetPRComments(ctx context.Context, owner, repo string, number int) ([]*ReviewComment, error)
    PostPRComment(ctx context.Context, owner, repo string, number int, body string) error

    // Reviews
    CreateReview(ctx context.Context, owner, repo string, number int, comments []*ReviewComment, body string) error
}
```

### 3.7 Coder

```go
// coder/coder.go

// CodingAgent 编码代理接口
type CodingAgent interface {
    // TDD 架构设计 + stub + 测试
    // 返回: branch name, error
    TDD(ctx context.Context, workdir string, issue *Issue) (string, error)

    // Implement 逻辑实现
    // 返回: error
    Implement(ctx context.Context, workdir string, issue *Issue) error

    // Revise 根据反馈修正
    // 返回: error
    Revise(ctx context.Context, workdir string, prURL string, comments []*ReviewComment) error
}
```

### 3.8 Dispatcher

```go
// dispatch/dispatch.go

type Result struct {
    Display  *Display
    IssueURL string
    PRURL    string
    Error    error
}

type Dispatcher interface {
    // Dispatch 分发命令到对应 Handler
    Dispatch(ctx context.Context, cmd *Command, session *Session) (*Result, error)
}
```

### 3.9 Task Handlers

```go
// handler/handler.go

type TaskHandler interface {
    Handle(ctx context.Context, cmd *Command, session *Session) (*Result, error)
}
```

#### Discuss Handler

```go
// handler/discuss.go

// DiscussHandler 需求讨论
// 处理 /new 和普通消息
type DiscussHandler struct {
    llm    LLMProvider
    github GitHubClient
}

func (h *DiscussHandler) Handle(ctx context.Context, cmd *Command, session *Session) (*Result, error) {
    switch cmd.Type {
    case CmdNew:
        return h.createIssue(ctx, session)
    case nil:
        return h.discuss(ctx, cmd.Raw, session)
    }
    return nil, ErrUnknownCommand
}

func (h *DiscussHandler) discuss(ctx context.Context, msg string, session *Session) (*Result, error) {
    // 1. 添加 Human 消息到 session
    session.AddDiscussMessage(&DiscussMessage{
        Role:    "human",
        Content: msg,
        Time:    time.Now(),
    })

    // 2. 获取讨论历史
    history := session.GetDiscussHistory()

    // 3. 调用 LLM 讨论
    messages := buildLLMMessages(history)
    response, err := h.llm.Chat(ctx, messages)
    if err != nil {
        return nil, err
    }

    // 4. 添加 AI 响应到 session
    session.AddDiscussMessage(&DiscussMessage{
        Role:    "ai",
        Content: response,
        Time:    time.Now(),
    })

    // 5. 保存 session
    session.Save()

    return &Result{
        Display: &Display{
            Type:    DisplayText,
            Content: response,
        },
    }, nil
}

func (h *DiscussHandler) createIssue(ctx context.Context, session *Session) (*Result, error) {
    // 1. 获取项目上下文
    project := session.GetProject()
    if project == nil {
        return nil, ErrNoProjectContext
    }

    // 2. 从 session 获取讨论内容
    history := session.GetDiscussHistory()
    if len(history) == 0 {
        return nil, ErrNoDiscussContent
    }

    // 3. 调用 LLM 整理成 Issue 格式
    messages := buildIssuePrompt(history)
    body, err := h.llm.Chat(ctx, messages)
    if err != nil {
        return nil, err
    }

    // 4. 提取标题（第一行）
    lines := strings.SplitN(body, "\n", 2)
    title := strings.Trim(lines[0], "# ")
    if len(lines) > 1 {
        body = lines[1]
    }

    // 5. 创建 GitHub Issue
    issue, err := h.github.CreateIssue(ctx, project.Owner, project.Repo, title, body)
    if err != nil {
        return nil, err
    }

    // 6. 清除讨论历史
    session.ClearDiscussHistory()
    session.Save()

    return &Result{
        Display: &Display{
            Type:    DisplayText,
            Content: fmt.Sprintf("Issue created: %s", issue.URL),
        },
        IssueURL: issue.URL,
    }, nil
}
```

#### TDD Handler

```go
// handler/tdd.go

// TDDHandler 架构设计 + stub + 测试
type TDDHandler struct {
    github GitHubClient
    coder  CodingAgent
}

func (h *TDDHandler) Handle(ctx context.Context, cmd *Command, session *Session) (*Result, error) {
    project := session.GetProject()
    if project == nil {
        return nil, ErrNoProjectContext
    }

    // 1. 解析 Issue URL/number
    owner, repo, number, err := parseURL(cmd.Params, project)
    if err != nil {
        return nil, err
    }

    // 2. 获取 Issue 内容
    issue, err := h.github.GetIssue(ctx, owner, repo, number)
    if err != nil {
        return nil, err
    }

    // 3. 获取讨论内容作为上下文
    history := session.GetDiscussHistory()

    // 4. 调用 Coder 执行 TDD
    branch, err := h.coder.TDD(ctx, project.Workdir, issue)
    if err != nil {
        return nil, err
    }

    // 5. 创建 Draft PR
    pr, err := h.github.CreatePR(ctx, owner, repo,
        fmt.Sprintf("TDD: %s", issue.Title),
        fmt.Sprintf("Issue: %s\n\n%s", issue.URL, formatDiscuss(history)),
        branch,
        "main",
        true,  // draft
    )
    if err != nil {
        return nil, err
    }

    return &Result{
        Display: &Display{
            Type:    DisplayText,
            Content: fmt.Sprintf("TDD PR created (draft): %s", pr.URL),
        },
        PRURL: pr.URL,
    }, nil
}
```

#### Fix Handler

```go
// handler/fix.go

// FixHandler 逻辑实现
type FixHandler struct {
    github GitHubClient
    coder  CodingAgent
}

func (h *FixHandler) Handle(ctx context.Context, cmd *Command, session *Session) (*Result, error) {
    project := session.GetProject()
    if project == nil {
        return nil, ErrNoProjectContext
    }

    // 1. 解析 Issue URL/number
    owner, repo, number, err := parseURL(cmd.Params, project)
    if err != nil {
        return nil, err
    }

    // 2. 获取 Issue 内容
    issue, err := h.github.GetIssue(ctx, owner, repo, number)
    if err != nil {
        return nil, err
    }

    // 3. 调用 Coder 实现逻辑
    err = h.coder.Implement(ctx, project.Workdir, issue)
    if err != nil {
        return nil, err
    }

    // 4. 查找对应的 Draft PR
    // TODO: 需要记录 TDD 创建的 PR 信息
    prs, err := h.github.GetPRsByBranch(ctx, owner, repo, issue.HeadBranch)
    if err != nil {
        return nil, err
    }

    if len(prs) > 0 {
        // 如果有 Draft PR，转换为正式 PR
        if prs[0].IsDraft {
            err = h.github.UpdatePR(ctx, owner, repo, prs[0].Number, "", "")
            if err != nil {
                return nil, err
            }
        }

        return &Result{
            Display: &Display{
                Type:    DisplayText,
                Content: fmt.Sprintf("PR updated: %s", prs[0].URL),
            },
            PRURL: prs[0].URL,
        }, nil
    }

    return &Result{
        Display: &Display{
            Type:    DisplayText,
            Content: "Implementation complete",
        },
    }, nil
}
```

#### Review Handler

```go
// handler/review.go

// ReviewHandler 代码审查
type ReviewHandler struct {
    llm    LLMProvider
    github GitHubClient
}

func (h *ReviewHandler) Handle(ctx context.Context, cmd *Command, session *Session) (*Result, error) {
    project := session.GetProject()
    if project == nil {
        return nil, ErrNoProjectContext
    }

    // 1. 解析 PR URL/number
    owner, repo, number, err := parseURL(cmd.Params, project)
    if err != nil {
        return nil, err
    }

    // 2. 获取 PR diff
    pr, err := h.github.GetPR(ctx, owner, repo, number)
    if err != nil {
        return nil, err
    }

    diff, err := h.github.GetPRDiff(ctx, owner, repo, number)
    if err != nil {
        return nil, err
    }

    // 3. 调用 LLM 审查
    result, err := h.llm.Review(ctx, diff)
    if err != nil {
        return nil, err
    }

    // 4. 写入 GitHub PR Comment
    if len(result.Comments) > 0 {
        err = h.github.CreateReview(ctx, owner, repo, number, result.Comments, result.Summary)
        if err != nil {
            return nil, err
        }
    } else {
        err = h.github.PostPRComment(ctx, owner, repo, number, result.Summary)
        if err != nil {
            return nil, err
        }
    }

    return &Result{
        Display: &Display{
            Type:    DisplayText,
            Content: fmt.Sprintf("Review posted: %s\n\n%s", pr.URL, result.Summary),
        },
        PRURL: pr.URL,
    }, nil
}
```

#### Revise Handler

```go
// handler/revise.go

// ReviseHandler 代码修正
type ReviseHandler struct {
    github GitHubClient
    coder  CodingAgent
}

func (h *ReviseHandler) Handle(ctx context.Context, cmd *Command, session *Session) (*Result, error) {
    project := session.GetProject()
    if project == nil {
        return nil, ErrNoProjectContext
    }

    // 1. 解析 PR URL/number
    owner, repo, number, err := parseURL(cmd.Params, project)
    if err != nil {
        return nil, err
    }

    // 2. 获取 PR Review Comments
    comments, err := h.github.GetPRComments(ctx, owner, repo, number)
    if err != nil {
        return nil, err
    }

    // 3. 调用 Coder 修正代码
    err = h.coder.Revise(ctx, project.Workdir, fmt.Sprintf("%s/%s#%d", owner, repo, number), comments)
    if err != nil {
        return nil, err
    }

    return &Result{
        Display: &Display{
            Type:    DisplayText,
            Content: "Code revised based on review feedback",
        },
        PRURL: fmt.Sprintf("https://github.com/%s/%s/pull/%d", owner, repo, number),
    }, nil
}
```

---

## 4. Data Models

### 4.1 讨论文档格式（LLM 生成）

```markdown
## 标题
[标题]

## 功能
- [具体功能点]

## 范围
- 包含：[包含哪些]
- 不包含：[不包含哪些]

## 技术方案
[实现方式]

## 验收标准
- [ ] [标准 1]
- [ ] [标准 2]
```

### 4.2 PR Body 格式

```markdown
Issue: https://github.com/owner/repo/issues/123

## 讨论摘要
[从 session 中提取的讨论内容]

## 变更内容
[stub 实现 / 逻辑实现说明]
```

---

## 5. Error Handling

| Error | Module | Description | Recovery |
|-------|--------|-------------|----------|
| `ErrNoProjectContext` | Session | 没有设置项目上下文 | Human 执行 `/on <path>` |
| `ErrNoDiscussContent` | Discuss | 讨论内容为空 | Human 补充讨论 |
| `ErrInvalidPath` | PathResolver | 路径不存在或无效 | Human 确认路径 |
| `ErrGitRemoteNotFound` | Project | 目录不是 git 仓库 | Human 确认目录 |
| `ErrIssueNotFound` | GitHub | Issue 不存在 | Human 确认 URL |
| `ErrPRNotFound` | GitHub | PR 不存在 | Human 确认 URL |
| `ErrLLM` | LLM | LLM 调用失败 | 自动重试 3 次 |
| `ErrCoder` | Coder | 编码执行失败 | Human 介入 |

---

## 6. Security

### 6.1 Credentials

- GitHub Token 从环境变量 `GITHUB_TOKEN` 读取
- 不持久化存储 token
- 不打印 token 到日志

### 6.2 Project Isolation

- 每个项目有独立的工作目录
- Git 操作限制在项目目录内
- 禁止跨项目操作

### 6.3 Human-in-the-Loop

- 所有 slash command 由 Human 执行
- AI 无法绕过 Human 执行关键操作
- AI 执行结果通过 Display 返回给 Human

---

## 7. Project Structure

```
humera/
├── cmd/
│   └── humera/
│       └── main.go              # 入口
│
├── channel/                       # IM 平台接入
│   ├── channel.go                # Channel interface
│   ├── telegram.go
│   ├── feishu.go
│   ├── discord.go
│   ├── cli.go
│   └── ws.go                     # WebSocket
│
├── agent/                         # Agent 核心
│   ├── loop.go                   # AgentLoop: LLM → 工具 → LLM
│   ├── tool.go                   # Tool interface
│   └── tools/                    # Tool 实现
│       ├── github_issue.go
│       ├── github_pr.go
│       ├── github_comment.go
│       ├── coding.go
│       └── filesystem.go
│
├── command/                       # 命令解析
│   └── parser.go                 # slash command parser
│
├── dispatch/                      # 命令分发
│   └── dispatcher.go
│
├── session/                       # 会话管理
│   ├── session.go                # Session interface
│   └── storage.go                 # SQLite storage
│
├── llm/                           # LLM 调用
│   ├── provider.go                # LLMProvider interface
│   ├── openai.go
│   └── anthropic.go
│
├── handler/                       # 任务处理器
│   ├── handler.go                # TaskHandler interface
│   ├── discuss.go
│   ├── tdd.go
│   ├── fix.go
│   ├── review.go
│   └── revise.go
│
├── github/                        # GitHub API
│   └── client.go                 # GitHubClient interface
│
├── coder/                         # 编码代理（嵌套 Agent Loop）
│   ├── coder.go                  # Coder interface + Validator
│   ├── agent.go                  # CoderAgent (Supervisor Loop)
│   ├── claude_code.go            # ClaudeCodeCLI implementation
│   └── opencode.go               # OpenCode CLI implementation
│
├── path/                          # 路径解析
│   └── resolver.go               # expandPath logic
│
├── display/                       # 输出格式化
│   └── formatter.go
│
├── errors/                        # 错误定义
│   └── errors.go
│
└── tests/                         # 测试
    └── ...
```

---

## 8. Implementation Notes

### 8.1 编码代理选择

优先使用 Claude Code：
```bash
claude --print --model sonnet-4-6-20250514 --system-prefill "<system>" --prefill "<user>" < prompt.txt
```

备选 OpenCode：
```bash
opencode -m "model" -c "指令"
```

### 8.2 Session 持久化

使用 SQLite 存储 session：
- 表：sessions (chat_id, project_json, discuss_json, updated_at)
- 表：discuss_messages (session_id, role, content, time)

### 8.3 TDD PR 流程

1. `/tdd` → Coder.TDD() → 创建 stub 分支
2. GitHub 创建 Draft PR
3. `/fix` → Coder.Implement() → 继续同一分支
4. 如果有 Draft PR，转为正式 PR

### 8.4 路径解析实现

参考 GTW `utils/path.js` 的 `expandPath` 逻辑：
```go
func expandPath(path string) string {
    // ~ → homedir
    // ~user → user homedir
    // / → absolute
    // ./ → relative to cwd
    // bare → ~/bare
}
```
