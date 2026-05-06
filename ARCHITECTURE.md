# Humera Technical Architecture

## 1. Architecture Overview

### 1.1 System Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Human (IM Channel)                             │
│                                                                          │
│   /on ~/code/myapp                                                     │
│   [any message]  ← Discuss 模式                                         │
│   /new                                                         │
│   /fix <issue>                                                           │
│   /review <pr>                                                          │
│   /revise <pr>                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Message
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              Channel                                     │
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
│                              Command                                     │
│                                                                          │
│   Responsibility:                                                        │
│   - 解析 slash command                                                   │
│   - 识别命令类型                                                         │
│   - 提取参数                                                             │
│                                                                          │
│   Interface:                                                             │
│   - Parse(Message) *Command                                              │
│                                                                          │
│   Command Types:                                                         │
│   - CmdOn          → Project                                             │
│   - CmdNew         → Discuss                                             │
│   - CmdFix         → Develop                                             │
│   - CmdReview      → Review                                              │
│   - CmdRevise      → Revise                                              │
│                                                                          │
│   Note: 无 slash command 的消息 → Discuss Handler                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Command
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                             Dispatch                                     │
│                                                                          │
│   Responsibility:                                                        │
│   - 根据 Command.Type 分发到对应的 Handler                                │
│   - 无 Command（普通消息）→ Discuss Handler                              │
│                                                                          │
│   Interface:                                                             │
│   - Dispatch(Command, *Project) *Result                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
┌─────────────────────────┐ ┌─────────────────────────┐ ┌─────────────────────────┐
│        Discuss          │ │        Develop          │ │        Review           │
│                         │ │                         │ │                         │
│   Context:              │ │   Dependencies:          │ │   Dependencies:         │
│   - 当前讨论内容         │ │   - GitHub              │ │   - GitHub              │
│   - 确认文档（暂存）     │ │   - Coder               │ │   - LLM Provider        │
│                         │ │                         │ │                         │
│   Output:                │ │   Output:                │ │   Output:                │
│   - 确认文档（内存）      │ │   - GitHub PR           │ │   - PR Comment          │
└─────────────────────────┘ └─────────────────────────┘ └─────────────────────────┘
                                        │
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              Revise                                     │
│                                                                          │
│   Dependencies:                                                          │
│   - GitHub                                                              │
│   - Coder                                                               │
│                                                                          │
│   Output:                                                                │
│   - Updated GitHub PR                                                    │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                            Infrastructure                               │
│                                                                          │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐       │
│   │     Project      │  │    GitHub API    │  │   LLM Provider   │       │
│   │                  │  │                  │  │                  │       │
│   │  Responsibility: │  │  Responsibility: │  │  Responsibility: │       │
│   │  - 读取 git remote│  │  - Issue CRUD    │  │  - Chat          │       │
│   │  - 项目路径解析    │  │  - PR CRUD       │  │  - Review        │       │
│   │  - 设置讨论模式    │  │  - Comments      │  │                  │       │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘       │
│                                                                          │
│   ┌──────────────────┐  ┌──────────────────┐                           │
│   │      Coder       │  │     Storage      │                           │
│   │                  │  │                  │                           │
│   │  Responsibility: │  │  Responsibility: │                           │
│   │  - 代码实现        │  │  - Discuss 暂存  │                           │
│   │  - 代码修改        │  │  - 项目配置      │                           │
│   └──────────────────┘  └──────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Module Boundaries

### 2.1 Module List

| Module | Responsibility | Boundary |
|--------|---------------|----------|
| **Channel** | IM 消息收发 | 输入/输出的唯一通道 |
| **Command** | 解析 slash command | 只解析，不执行 |
| **Dispatch** | 分发命令到 Handler | 路由，无业务逻辑 |
| **Discuss** | 需求讨论 | 业务：讨论流程 |
| **Develop** | 编码开发 | 业务：开发流程 |
| **Review** | 代码审查 | 业务：review 流程 |
| **Revise** | 代码修正 | 业务：修正流程 |
| **Project** | 项目上下文 | 数据：从 git remote 读取 |
| **GitHub** | GitHub API | 外部依赖：GitHub |
| **LLM** | LLM 调用 | 外部依赖：LLM Provider |
| **Coder** | 编码执行 | 外部依赖：Claude Code / OpenCode |
| **Storage** | 本地存储 | 数据持久化 |

### 2.2 Dependency Rules

```
Channel → Command → Dispatch → [Discuss | Develop | Review | Revise]
                                       ↓
                              [Project | GitHub | LLM | Coder | Storage]
```

**依赖方向（只能依赖下游）：**
- Channel 可以被 Command 依赖（消息输入）
- Command 可以被 Dispatch 依赖（命令输入）
- Dispatch 可以依赖所有 Handler
- Handler 可以依赖 Project、GitHub、LLM、Coder、Storage

**禁止反向依赖：**
- Handler 不能依赖 Dispatch
- Command 不能依赖 Channel
- 上游模块不能依赖下游模块

---

## 3. Module Interfaces

### 3.1 Channel

```go
// channel/channel.go
type Channel interface {
    // 接收 Human 消息
    Receive(ctx context.Context) (*Message, error)

    // 发送消息给 Human
    Send(ctx context.Context, display Display) error

    // 启动监听
    Start(ctx context.Context, handler MessageHandler) error

    // 停止监听
    Stop(ctx context.Context) error
}

// 支持的实现：
// - TelegramChannel
// - FeishuChannel
// - DiscordChannel
// - WhatsAppChannel
// - CLIChannel
// - WebSocketChannel
```

### 3.2 Command

```go
// command/command.go
type CommandType string

const (
    CmdOn          CommandType = "on"           // 切换项目
    CmdNew         CommandType = "new"            // 创建 Issue
    CmdFix         CommandType = "fix"          // 编码开发
    CmdReview      CommandType = "review"       // 代码审查
    CmdRevise      CommandType = "revise"       // 修正
)

type Command struct {
    Type   CommandType
    Raw    string
    Params map[string]string
}

type Parser interface {
    // 解析消息为命令
    // 如果不是 slash command，返回 nil（由 Discuss Handler 处理）
    Parse(msg *Message) *Command
}
```

### 3.3 Dispatch

```go
// dispatch/dispatch.go
type Dispatcher interface {
    // 分发命令到对应 Handler
    // cmd 为 nil 时，默认分发到 Discuss Handler
    Dispatch(ctx context.Context, cmd *Command, project *Project) (*Result, error)
}

type Result struct {
    Display  Display
    IssueURL string
    PRURL    string
    Error    error
}
```

### 3.4 Project

```go
// project/project.go
type Project struct {
    Path   string
    Remote Remote
    Branch string
}

type Remote struct {
    URL   string  // 例如 github.com/owner/repo
    Owner string
    Repo  string
}

type Resolver interface {
    // 根据路径解析项目
    Resolve(path string) (*Project, error)

    // 设置当前项目上下文
    SetCurrent(project *Project)

    // 获取当前项目上下文
    GetCurrent() *Project
}
```

### 3.5 Discuss Handler

```go
// handler/discuss.go
type DiscussHandler struct {
    llm     LLMProvider
    storage Storage
    github  GitHubClient
    context *DiscussContext  // 当前讨论状态（内存）
}

type DiscussContext struct {
    Project  *Project
    Messages []Message
    Document *ConfirmDocument  // 确认文档（暂存）
}

type ConfirmDocument struct {
    Title     string
    Features  []string
    Scope     Scope
    Solution  string
    Criteria  []string
}

// Handle 实现 TaskHandler 接口
// 注意：Discuss Handler 处理两类输入：
// 1. CmdNew         → 创建 Issue
// 2. 普通消息（cmd == nil）→ 讨论迭代
func (h *DiscussHandler) Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error)

// 子方法：
// - discuss(msg, context)     → 迭代确认
// - createIssue(context)     → 创建 Issue
```

### 3.6 Develop Handler

```go
// handler/develop.go
type DevelopHandler struct {
    github GitHubClient
    coder  Coder
}

// Handle 实现 TaskHandler 接口
func (h *DevelopHandler) Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error)
```

### 3.7 Review Handler

```go
// handler/review.go
type ReviewHandler struct {
    llm    LLMProvider
    github GitHubClient
}

// Handle 实现 TaskHandler 接口
func (h *ReviewHandler) Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error)
```

### 3.8 Revise Handler

```go
// handler/revise.go
type ReviseHandler struct {
    github GitHubClient
    coder  Coder
}

// Handle 实现 TaskHandler 接口
func (h *ReviseHandler) Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error)
```

### 3.9 GitHub Client

```go
// github/github.go
type GitHubClient interface {
    // Issue
    GetIssue(ctx context.Context, url string) (*Issue, error)
    CreateIssue(ctx context.Context, project *Project, doc *ConfirmDocument) (*Issue, error)

    // PR
    GetPR(ctx context.Context, url string) (*PR, error)
    CreatePR(ctx context.Context, project *Project, branch string, doc *ConfirmDocument) (*PR, error)
    GetReviewComments(ctx context.Context, pr *PR) ([]*ReviewComment, error)

    // Comments
    PostReviewComment(ctx context.Context, pr *PR, review *ReviewResult) error

    // Push
    PushBranch(ctx context.Context, project *Project, branch string) error
}
```

### 3.10 LLM Provider

```go
// provider/llm.go
type LLMProvider interface {
    // 对话
    Chat(ctx context.Context, prompt string) (string, error)

    // 代码审查
    Review(ctx context.Context, diff string) (*ReviewResult, error)
}
```

### 3.11 Coder

```go
// coder/coder.go
type Coder interface {
    // 根据 Issue 实现代码
    Implement(ctx context.Context, project *Project, issue *Issue) error

    // 根据 Review 反馈修正代码
    Revise(ctx context.Context, project *Project, pr *PR, comments []*ReviewComment) error
}
```

### 3.12 Storage

```go
// storage/storage.go
type Storage interface {
    // Discuss 上下文
    SaveDiscussContext(ctx *DiscussContext) error
    GetDiscussContext(projectID string) (*DiscussContext, error)

    // 项目配置
    SaveProject(project *Project) error
    GetProject(path string) (*Project, error)
}
```

---

## 4. Data Models

### 4.1 Issue

```go
type Issue struct {
    Number    int
    Title     string
    Body      string
    State     string
    URL       string
    Project   *Project
}
```

### 4.2 PR

```go
type PR struct {
    Number    int
    Title     string
    Body      string
    State     string
    Head      string  // branch name
    Base      string  // target branch
    URL       string
    Diff      string
    Project   *Project
}
```

### 4.3 ReviewComment

```go
type ReviewComment struct {
    ID        int
    Author    string
    Body      string
    Path      string
    Line      int
    CreatedAt time.Time
}
```

### 4.4 ReviewResult

```go
type ReviewResult struct {
    Comments []*ReviewComment
    Summary  string
}
```

---

## 5. Error Handling

| Error | Module | Description | Recovery |
|-------|--------|-------------|----------|
| `ErrProjectNotFound` | Project | 目录不存在 | Human 确认路径 |
| `ErrGitRemoteNotFound` | Project | 无 git remote | Human 确认目录 |
| `ErrGitHubAuth` | GitHub | GitHub 认证失败 | 检查 GITHUB_TOKEN |
| `ErrIssueNotFound` | GitHub | Issue 不存在 | Human 确认 URL |
| `ErrPRNotFound` | GitHub | PR 不存在 | Human 确认 URL |
| `ErrLLM` | LLM | LLM 调用失败 | 自动重试 3 次 |
| `ErrCoder` | Coder | 编码失败 | Human 介入 |

---

## 6. Security

### 6.1 Credentials

- GitHub Token 从环境变量读取（`GITHUB_TOKEN`）
- 不持久化存储 token
- 不打印 token 到日志

### 6.2 Project Isolation

- 每个项目有独立的工作目录
- Git 操作限制在项目目录内
- 禁止跨项目操作

### 6.3 Human-in-the-Loop

- 所有 slash command 由 Human 执行
- AI 无法绕过 Human 执行关键操作
- AI 执行结果需要 Human 确认后才能继续

---

## 7. Project Structure

```
humera/
├── cmd/
│   └── humera/
│       └── main.go
│
├── internal/
│   ├── channel/          # IM 平台接入
│   │   ├── channel.go
│   │   ├── telegram.go
│   │   ├── feishu.go
│   │   ├── discord.go
│   │   ├── cli.go
│   │   └── websocket.go
│   │
│   ├── command/          # 命令解析
│   │   ├── command.go
│   │   └── parser.go
│   │
│   ├── dispatch/          # 命令分发
│   │   └── dispatch.go
│   │
│   ├── handler/           # 任务处理器
│   │   ├── handler.go     # 接口定义
│   │   ├── discuss.go
│   │   ├── develop.go
│   │   ├── review.go
│   │   └── revise.go
│   │
│   ├── project/           # 项目上下文
│   │   └── project.go
│   │
│   ├── github/            # GitHub API
│   │   └── github.go
│   │
│   ├── provider/          # LLM 接入
│   │   └── llm.go
│   │
│   ├── coder/             # 编码执行
│   │   └── coder.go
│   │
│   └── storage/           # 本地存储
│       └── storage.go
│
├── pkg/
│   └── display/           # 格式化输出
│       └── display.go
│
└── test/
    └── integration/        # 集成测试
```

---

## 8. External Dependencies

### 8.1 GitHub API

- Issue CRUD
- PR CRUD
- Review Comments
- 认证：Personal Access Token

### 8.2 LLM Provider

- Chat API（确认、建议生成）
- 支持：OpenAI、Anthropic、Ollama、本地模型

### 8.3 Coder

- Claude Code CLI
- OpenCode CLI
- 支持自定义 coder 实现
