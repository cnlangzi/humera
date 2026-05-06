# Humera Architecture

## 1. Design Philosophy

**Humera = Human 的智能协作工具**

Humera 不是一个 AI 自动化平台，而是一个增强 Human 工程能力的工具。Human 是唯一的掌控者和执行者，AI 只负责协助确认和输出建议，最终执行必须由 Human 完成。

### 1.1 Core Principles

| Principle | Description |
|-----------|-------------|
| **Human in Control** | 所有关键操作（创建 Issue、提交代码、合并 PR）由 Human 执行 |
| **AI Assists Only** | AI 负责具体化、确认、建议，不负责最终产出 |
| **Task Isolation** | 每个任务是独立的，没有跨任务的流程或状态关联 |
| **Project from Git** | 项目上下文从本地 git remote 读取，不需要额外配置 |
| **Explicit Commands** | Human 通过 slash command 明确指定要执行的操作 |

### 1.2 Comparison

| Dimension | Auto Agent (Symphony) | Humera |
|-----------|----------------------|--------|
| Human Role | Observer | Controller + Executor |
| AI Role | Executor | Assistant |
| Task Flow | Pipeline / State Machine | Independent Tasks |
| Final Output | AI creates directly | Human confirms then executes |
| Concurrency | Multiple agents | Single task |

---

## 2. Core Model

### 2.1 Task Lifecycle

每个任务遵循完全相同的三阶段模式：

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 1. Human 提出                                            │   │
│   │    - 原始想法                                             │   │
│   │    - 指定任务类型                                         │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │                                     │
│   ┌─────────────────────────▼───────────────────────────────┐   │
│   │ 2. AI 协助                                               │   │
│   │    - 具体化需求                                           │   │
│   │    - 输出确认文档                                         │   │
│   │    - 给出建议方案                                         │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │                                     │
│   ┌─────────────────────────▼───────────────────────────────┐   │
│   │ 3. Human 确认 + 执行                                      │   │
│   │    - "可以" 或 "改一下..."                                │   │
│   │    - 执行 slash command                                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**关键约束：AI 永远不执行最终产物。所有 slash command 由 Human 执行。**

### 2.2 Task Types

Humera 支持 4 种独立的任务类型：

| Task | AI Output | Human Execution | Artifact |
|------|-----------|-----------------|----------|
| **Discuss** | 确认后的需求文档（方案 + 细节） | `/create-issue` | GitHub Issue |
| **Develop** | 代码修改建议 + 实现细节 | `/start-dev` | GitHub PR (pushed code) |
| **Review** | review 结果和建议 | 无（AI 输出到 PR） | GitHub PR Comment |
| **Revise** | 修正建议 | `/revise` | GitHub PR (updated code) |

### 2.3 Project Context

项目上下文在开始任务时通过 `on <path>` 命令确定：

```
Human: /on ~/code/myapp
       │
       ▼
Humera reads:
  - Local directory: ~/code/myapp
  - Git remote: github.com/owner/repo
  - Current branch
       │
       ▼
Project context established
```

**项目切换本质：切换工作目录 + 读取 git remote**

项目信息存储在本地，不依赖外部数据库。

---

## 3. Module Definitions

### 3.1 Module Overview

```
humera/
├── channel/          # 消息入口和出口（IM 平台）
├── command/          # slash command 解析
├── assist/           # AI 协助能力
│   ├── discuss.go       # 需求讨论协助
│   ├── review.go        # 代码审查协助
│   └── suggest.go        # 修正建议协助
├── project/          # 项目上下文管理
├── artifact/         # 产物生成
└── infra/           # 基础设施（存储、日志）
```

---

### 3.2 Channel

**Purpose**: 接收 Human 消息，返回 AI 协助结果。

```go
// channel/channel.go
type Channel interface {
    // 发送消息给 Human
    Send(ctx context.Context, msg Display) error

    // 接收 Human 输入
    Receive(ctx context.Context) (*Message, error)

    // 启动监听
    Start(ctx context.Context, handler Handler) error

    // 停止监听
    Stop(ctx context.Context) error
}
```

**Supported Channels**:
- Telegram
- Feishu (Lark)
- WhatsApp
- Discord
- CLI
- WebSocket

**Responsibilities**:
- 连接各 IM 平台
- 消息格式标准化
- 返回格式化结果（markdown、card、buttons）

---

### 3.3 Command

**Purpose**: 解析 Human 输入，识别要执行的任务类型。

```go
// command/command.go
type CommandType string

const (
    CmdOn          CommandType = "on"          // 切换项目
    CmdDiscuss     CommandType = "discuss"     // 需求讨论
    CmdCreateIssue CommandType = "create-issue" // 创建 Issue
    CmdStartDev    CommandType = "start-dev"  // 开始开发
    CmdReview      CommandType = "review"     // PR Review
    CmdRevise      CommandType = "revise"     // 修正
    CmdMerge       CommandType = "merge"      // 合并 PR
)

type Command struct {
    Type CommandType

    // 原始输入
    Raw string

    // 解析后的参数
    Params map[string]string
}
```

**Command Parsing Rules**:

| Input | Command | Notes |
|-------|---------|-------|
| `/on <path>` | CmdOn | 切换项目上下文 |
| `/discuss <idea>` | CmdDiscuss | 开始需求讨论 |
| `/create-issue` | CmdCreateIssue | 创建 GitHub Issue（当前讨论结果） |
| `/start-dev <issue-url>` | CmdStartDev | 开始开发指定 issue |
| `/review <pr-url>` | CmdReview | Review 指定 PR |
| `/revise <pr-url>` | CmdRevise | 修正指定 PR |
| `/merge <pr-url>` | CmdMerge | 合并指定 PR |
| 其他 | CmdDiscuss | 作为需求讨论处理 |

**Intent Recognition**:
- Slash command 优先匹配
- 无 slash command → 作为讨论内容处理
- 不存在"理解意图"，只有"解析命令"

---

### 3.4 Project

**Purpose**: 管理项目上下文，从 git remote 读取项目信息。

```go
// project/project.go
type Project struct {
    // 本地路径
    Path string

    // Git remote 信息
    Remote Remote

    // 当前分支
    Branch string
}

type Remote struct {
    // 例如 "github.com/owner/repo"
    URL string

    // 解析后的 owner 和 repo
    Owner string
    Repo string
}
```

**Project Resolution**:

```
/on ~/code/myapp
      │
      ▼
cd ~/code/myapp
      │
      ▼
git remote -v
      │
      ▼
Parse: github.com/owner/repo
      │
      ▼
Set as current project context
```

**Responsibilities**:
- 读取本地 git remote
- 解析 GitHub owner/repo
- 管理当前项目路径
- 提供 GitHub API 访问凭证（从环境变量或 git config）

---

### 3.5 Assist

**Purpose**: AI 协助能力，根据任务类型提供不同的协助。

```go
// assist/assist.go
type Assister interface {
    // 任务类型
    TaskType() TaskType

    // 执行协助
    Assist(ctx context.Context, input string, project *Project) (*AssistResult, error)
}

type AssistResult struct {
    // AI 的输出内容
    Output Display

    // 是否需要 Human 确认
    NeedsConfirmation bool

    // 确认后的产物（如果已完成）
    Artifact *Artifact
}
```

#### 3.5.1 Discuss Assister

**Input**: Human 的原始需求想法
**Output**: 具体化后的需求文档

```go
// assist/discuss.go
type DiscussAssister struct {
    llm llm.Provider
}

func (a *DiscussAssister) Assist(ctx context.Context, input string, project *Project) (*AssistResult, error) {
    // 1. AI 分析原始需求
    analysis := a.analyze(input)

    // 2. 输出结构化确认文档
    //    - 功能点
    //    - 范围边界
    //    - 技术方案
    //    - 验收标准
    output := a.formatAnalysis(analysis)

    return &AssistResult{
        Output: output,
        NeedsConfirmation: true,
    }
}
```

**Display Template**:

```
需求确认文档
═══════════════════════════════════

功能：
- [具体功能点]

范围：
- [包含哪些]
- [不包含哪些]

技术方案：
- [实现方式]

验收标准：
- [如何验证]

───────────────────────────────────
确认后输入 /create-issue 创建 Issue
```

#### 3.5.2 Review Assister

**Input**: PR URL
**Output**: Review 意见（输出到 GitHub PR Comment）

```go
// assist/review.go
type ReviewAssister struct {
    llm    llm.Provider
    github GitHubTool
}

func (a *ReviewAssister) Assist(ctx context.Context, prURL string, project *Project) (*AssistResult, error) {
    // 1. 获取 PR diff
    diff := a.github.GetPRDiff(prURL)

    // 2. AI 分析代码
    review := a.llm.Review(diff)

    // 3. 输出到 GitHub PR
    a.github.PostReviewComment(prURL, review)

    return &AssistResult{
        Output: formatReviewSummary(review),
        NeedsConfirmation: false,  // 直接输出到 GitHub
    }
}
```

#### 3.5.3 Suggest Assister

**Input**: Human 指出的修改点
**Output**: 修正建议

```go
// assist/suggest.go
type SuggestAssister struct {
    llm llm.Provider
}

func (a *SuggestAssister) Assist(ctx context.Context, feedback string, project *Project) (*AssistResult, error) {
    // 1. AI 分析 Human 的反馈
    suggestions := a.analyzeFeedback(feedback)

    // 2. 输出修正建议
    output := a.formatSuggestions(suggestions)

    return &AssistResult{
        Output: output,
        NeedsConfirmation: true,
    }
}
```

---

### 3.6 Artifact

**Purpose**: 管理任务产物（Issue、Code、PR Comment）。

```go
// artifact/artifact.go
type ArtifactType string

const (
    ArtifactIssue    ArtifactType = "issue"    // GitHub Issue
    ArtifactCode     ArtifactType = "code"     // 代码修改
    ArtifactReview   ArtifactType = "review"   // PR Review
    ArtifactDocument ArtifactType = "document" // 确认文档
)

type Artifact struct {
    ID   ArtifactID
    Type ArtifactType

    // 内容
    Title   string
    Content string

    // 关联的 Project
    ProjectID ProjectID

    // 创建时间
    CreatedAt time.Time
}
```

---

### 3.7 Infra

**Purpose**: 基础设施支持。

```go
// infra/infra.go

// Storage: SQLite 本地存储
type Storage interface {
    // 项目信息
    SaveProject(project *Project) error
    GetProject(path string) (*Project, error)
    ListProjects() ([]*Project, error)

    // 讨论历史
    SaveDiscussion(discussion *Discussion) error
    GetDiscussion(id DiscussionID) (*Discussion, error)
}

// GitHub API
type GitHubClient interface {
    // Issues
    CreateIssue(owner, repo string, issue *Issue) (*Issue, error)

    // PRs
    CreatePR(owner, repo string, pr *PR) (*PR, error)
    PostPRComment(prURL string, comment string) error

    // Reviews
    PostReview(prURL string, review *Review) error
}

// LLM Provider
type LLMProvider interface {
    Chat(ctx context.Context, prompt string) (string, error)
    Review(ctx context.Context, diff string) (*ReviewResult, error)
}
```

---

## 4. Interaction Flow

### 4.1 Task Flow: Discuss

```
Human: /on ~/code/myapp
       │
       ▼
Project context set: ~/code/myapp → github.com/owner/repo
       │
       ▼
Human: 我要给 API 加 rate limiting
       │
       ▼
AI (DiscussAssister):
```
需求确认文档
═══════════════════════════════════

功能：
- 给所有 API 添加 rate limiting

范围：
- 包含：REST API
- 不包含：GraphQL

技术方案：
- Redis + sliding window
- 限制：100 req/min per user

验收标准：
- [ ] 返回 429 当超过限制
- [ ] 计数器正确重置
───────────────────────────────────
确认后输入 /create-issue 创建 Issue
```
       │
       ▼
Human: 改成 200 req/min
       │
       ▼
AI: 更新文档，限制改为 200
       │
       ▼
Human: 可以
       │
       ▼
Human: /create-issue
       │
       ▼
GitHub Issue #123 created
```

### 4.2 Task Flow: Develop

```
Human: /start-dev https://github.com/owner/repo/issues/123
       │
       ▼
AI (DevAssister):
- 分析 Issue 需求
- 建议实现方案
- 输出到 display
       │
       ▼
Human: 可以
       │
       ▼
Human: /start-dev (confirm)
       │
       ▼
代码被 push 到 GitHub
PR #456 created
       │
       ▼
Human 在 GitHub 上 review PR
```

### 4.3 Task Flow: Review

```
Human: /review https://github.com/owner/repo/pull/456
       │
       ▼
AI (ReviewAssister):
- 获取 PR diff
- 分析代码
- 输出 review 到 GitHub PR Comment
       │
       ▼
Human 在 GitHub 上查看 review
Human 决定是否需要 /revise
```

### 4.4 Task Flow: Revise

```
Human: /revise https://github.com/owner/repo/pull/456
       │
       ▼
Human: review 提的问题需要修正
       │
       ▼
AI (SuggestAssister):
- 分析 review 反馈
- 输出修正建议
       │
       ▼
Human: 确认修正方案
       │
       ▼
Human: /revise (confirm)
       │
       ▼
修正代码 push 到 GitHub
```

---

## 5. Command Reference

| Command | Description | Execution |
|---------|-------------|-----------|
| `/on <path>` | 切换项目上下文 | 读取 git remote |
| `/discuss <idea>` | 开始需求讨论 | AI 协助确认 |
| `/create-issue` | 创建 Issue | GitHub API |
| `/start-dev <issue>` | 开始开发 | push + PR |
| `/review <pr>` | Review PR | GitHub PR Comment |
| `/revise <pr>` | 修正 PR | push 修正代码 |
| `/merge <pr>` | 合并 PR | GitHub API |

---

## 6. Error Handling

### 6.1 Error Types

| Error | Description | Recovery |
|-------|-------------|----------|
| `project_not_found` | 指定目录不存在 | Human 确认目录路径 |
| `git_remote_not_found` | 目录无 git remote | Human 确认目录 |
| `github_auth_failed` | GitHub 认证失败 | 检查 GITHUB_TOKEN |
| `issue_not_found` | 指定的 issue 不存在 | Human 确认 URL |
| `pr_not_found` | 指定的 PR 不存在 | Human 确认 URL |
| `llm_error` | LLM 调用失败 | 重试 |

### 6.2 Recovery Strategy

- **项目错误**: Human 重新输入正确路径
- **GitHub 错误**: 检查凭证或网络
- **LLM 错误**: 自动重试 3 次

---

## 7. Security Considerations

### 7.1 Credentials

- GitHub Token 从环境变量读取，不存储
- 不打印 token 到日志

### 7.2 Project Isolation

- 每个项目独立的工作目录
- Git 操作在项目目录内执行
- 防止误操作其他项目

### 7.3 Human-in-the-Loop

- AI 不直接执行任何 Git 操作
- 所有关键操作需要 Human 显式执行 slash command
