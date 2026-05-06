# Humera Architecture

## 1. Design Philosophy

**Humera = Human 掌控的 AI 协作工具**

Human 是掌控者和执行者，AI 负责协助确认和建议。所有 slash command 由 Human 执行，AI 只负责过程协助。

### 1.1 Core Principles

| Principle | Description |
|-----------|-------------|
| **Human in Control** | Human 发起所有任务，执行所有 slash command |
| **AI Assists Only** | AI 负责确认、建议、输出，不执行最终产物 |
| **Task Isolation** | 每个任务独立，没有跨任务的状态关联 |
| **Project from Git** | 项目上下文从 git remote 读取 |
| **Explicit Commands** | 所有操作通过 slash command 显式指定 |

### 1.2 Comparison

| Dimension | Auto Agent | Humera |
|-----------|------------|--------|
| Human Role | Observer | Controller + Executor |
| AI Role | Executor | Assistant |
| Output | AI creates directly | Human confirms, then executes |
| Flow | Pipeline / State Machine | Independent Tasks |

---

## 2. Task Definitions

Humera 定义了 4 种独立任务：

| Task | Human Action | AI Action | Output |
|------|-------------|-----------|--------|
| **Discuss** | 提出需求 + 执行 `/create-issue` | 确认细节 + 整合文档 | GitHub Issue |
| **Develop** | 执行 `/start-dev <issue>` | 编码 + push + PR | GitHub PR |
| **Review** | 执行 `/review <pr>` | review + 写 comment | PR Comment |
| **Revise** | 执行 `/revise <pr>` | 读反馈 + 修改 + push | Updated PR |

---

## 3. Task 1: Discuss（需求讨论）

**起点：所有功能开发和 Bug 修改都从这里开始**

### 3.1 Flow

| Stage | Human | AI |
|-------|-------|-----|
| 1 | 提出简单需求 | |
| 2 | | 确认具体细节 |
| 3 | 补充 / 修改 | |
| 4 | | 确认理解是否正确 |
| 5 | ← 循环直到双方确认 → | |
| 6 | | 整合输出最终确认文档 |
| 7 | 确认最终文档 | |
| 8 | 执行 `/create-issue` | |
| 9 | | 把确认文档写入 GitHub Issue |
| 10 | | 创建 Issue |
| Output | **包含最终讨论结论的 GitHub Issue** |

### 3.2 AI Output Example

```
需求确认文档
═══════════════════════════════════

标题：给 API 添加 rate limiting

功能：
- 给所有 REST API 添加 rate limiting

范围：
- 包含：REST API
- 不包含：GraphQL

技术方案：
- Redis + sliding window
- 限制：200 req/min per user

验收标准：
- [ ] 超过限制返回 429
- [ ] 计数器正确重置

───────────────────────────────────
确认后执行 /create-issue
```

### 3.3 Command

```bash
# 开始讨论
/discuss 我要给 API 加 rate limiting

# 创建 Issue（讨论结束后）
/create-issue
```

---

## 4. Task 2: Develop（编码）

### 4.1 Flow

| Stage | Human | AI |
|-------|-------|-----|
| 1 | 执行 `/start-dev <issue-url>` | |
| 2 | | 读取 GitHub Issue 详情 |
| 3 | | 分析需求，开始编码 |
| 4 | | push 代码到分支 |
| 5 | | 创建 PR |
| Output | **代码 push 到 GitHub，PR 创建** |

### 4.2 Notes

- Human 指定要开发的 issue，AI 自主完成
- 没有中间的确认环节
- AI 负责：编码、push、PR 创建

### 4.3 Command

```bash
/start-dev https://github.com/owner/repo/issues/123
```

---

## 5. Task 3: Review（代码审查）

### 5.1 Flow

| Stage | Human | AI |
|-------|-------|-----|
| 1 | 执行 `/review <pr-url>` | |
| 2 | | 读取 PR 代码 |
| 3 | | 专业方式 review 代码 |
| 4 | | 把修改意见写入 GitHub PR Comment |
| Output | **Review 意见在 GitHub PR 上** |

### 5.2 Command

```bash
/review https://github.com/owner/repo/pull/456
```

---

## 6. Task 4: Revise（修正）

### 6.1 Flow

| Stage | Human | AI |
|-------|-------|-----|
| 1 | 执行 `/revise <pr-url>` | |
| 2 | | 读取 PR 及 Review Comments |
| 3 | | 分析 Review 反馈 |
| 4 | | 修改代码 |
| 5 | | push 代码到远端 |
| Output | **修正代码 push 到 GitHub** |

### 6.2 Command

```bash
/revise https://github.com/owner/repo/pull/456
```

---

## 7. Module Architecture

### 7.1 Module Overview

```
humera/
├── channel/          # 消息入口和出口
├── command/          # slash command 解析
├── task/             # 任务处理器
│   ├── discuss.go       # 需求讨论
│   ├── develop.go       # 编码
│   ├── review.go        # 代码审查
│   └── revise.go        # 修正
├── project/          # 项目上下文
├── provider/         # LLM 调用
├── github/           # GitHub API
└── infra/            # 存储、日志
```

### 7.2 Channel

消息入口和出口，支持多种 IM 平台。

```go
type Channel interface {
    Send(ctx context.Context, msg Display) error
    Receive(ctx context.Context) (*Message, error)
    Start(ctx context.Context, handler Handler) error
    Stop(ctx context.Context) error
}

// Supported: Telegram, Feishu, Discord, WhatsApp, CLI, WebSocket
```

### 7.3 Command

解析 slash command，识别任务类型。

```go
type CommandType string

const (
    CmdOn          CommandType = "on"           // 切换项目
    CmdDiscuss     CommandType = "discuss"      // 需求讨论
    CmdCreateIssue CommandType = "create-issue" // 创建 Issue
    CmdStartDev    CommandType = "start-dev"   // 开始开发
    CmdReview      CommandType = "review"      // 代码审查
    CmdRevise      CommandType = "revise"      // 修正
)

type Command struct {
    Type   CommandType
    Raw    string
    Params map[string]string
}
```

| Input | Command | Handler |
|-------|---------|---------|
| `/on <path>` | CmdOn | Project |
| `/discuss <idea>` | CmdDiscuss | Discuss |
| `/create-issue` | CmdCreateIssue | Discuss |
| `/start-dev <issue>` | CmdStartDev | Develop |
| `/review <pr>` | CmdReview | Review |
| `/revise <pr>` | CmdRevise | Revise |
| 其他 | CmdDiscuss | Discuss |

### 7.4 Project

项目上下文从 git remote 读取。

```go
type Project struct {
    Path   string   // 本地路径
    Remote Remote   // Git remote 信息
    Branch string   // 当前分支
}

type Remote struct {
    URL   string   // 例如 github.com/owner/repo
    Owner string
    Repo  string
}
```

**Project Resolution:**

```
/on ~/code/myapp
      │
      ▼
cd ~/code/myapp
git remote -v
      │
      ▼
github.com/owner/repo
```

### 7.5 Task Handlers

每个任务类型对应一个 handler。

```go
type TaskHandler interface {
    TaskType() CommandType
    Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error)
}
```

#### 7.5.1 Discuss Handler

```go
type DiscussHandler struct {
    llm      LLMProvider
    github   GitHubClient
    discuss  *DiscussContext  // 当前讨论状态
}

func (h *DiscussHandler) Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error) {
    switch cmd.Type {
    case CmdDiscuss:
        // 接收 Human 的需求，开始或继续讨论
        return h.discuss(cmd.Raw, project)

    case CmdCreateIssue:
        // 创建 Issue
        return h.createIssue(project)
    }
}
```

#### 7.5.2 Develop Handler

```go
type DevelopHandler struct {
    llm     LLMProvider
    github  GitHubClient
    coder   Coder  // Claude Code / OpenCode
}

func (h *DevelopHandler) Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error) {
    // 1. 读取 Issue 详情
    issue := h.github.GetIssue(cmd.Params["issue_url"])

    // 2. 编码
    diff := h.coder.Implement(issue)

    // 3. push + PR
    pr := h.github.CreatePR(project, diff)

    return &Result{PR: pr}
}
```

#### 7.5.3 Review Handler

```go
type ReviewHandler struct {
    llm    LLMProvider
    github GitHubClient
}

func (h *ReviewHandler) Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error) {
    // 1. 读取 PR
    pr := h.github.GetPR(cmd.Params["pr_url"])

    // 2. review 代码
    review := h.llm.Review(pr.Diff)

    // 3. 写入 PR Comment
    h.github.PostReviewComment(pr, review)

    return &Result{Review: review}
}
```

#### 7.5.4 Revise Handler

```go
type ReviseHandler struct {
    llm    LLMProvider
    github GitHubClient
    coder  Coder
}

func (h *ReviseHandler) Handle(ctx context.Context, cmd *Command, project *Project) (*Result, error) {
    // 1. 读取 PR 及其 Review Comments
    pr := h.github.GetPR(cmd.Params["pr_url"])
    comments := h.github.GetReviewComments(pr)

    // 2. 分析反馈
    feedback := h.parseFeedback(comments)

    // 3. 修改代码
    diff := h.coder.Revise(pr, feedback)

    // 4. push
    h.github.PushAndUpdatePR(pr, diff)

    return &Result{Updated: true}
}
```

### 7.6 Provider

LLM 调用抽象。

```go
type LLMProvider interface {
    Chat(ctx context.Context, prompt string) (string, error)
    Review(ctx context.Context, diff string) (*ReviewResult, error)
}
```

### 7.7 GitHub

GitHub API 抽象。

```go
type GitHubClient interface {
    // Issue
    GetIssue(url string) (*Issue, error)
    CreateIssue(project *Project, title, body string) (*Issue, error)

    // PR
    GetPR(url string) (*PR, error)
    CreatePR(project *Project, diff *Diff) (*PR, error)
    GetReviewComments(pr *PR) ([]*ReviewComment, error)
    PostReviewComment(pr *PR, review *ReviewResult) error
    PushAndUpdatePR(pr *PR, diff *Diff) error
}
```

### 7.8 Coder

编码能力抽象。

```go
type Coder interface {
    // 根据 Issue 实现代码
    Implement(ctx context.Context, issue *Issue) (*Diff, error)

    // 根据 Review 反馈修正代码
    Revise(ctx context.Context, pr *PR, feedback []*ReviewComment) (*Diff, error)
}
```

---

## 8. Command Reference

| Command | Description | Handler |
|---------|-------------|---------|
| `/on <path>` | 切换项目上下文 | Project |
| `/discuss <idea>` | 开始需求讨论 | Discuss |
| `/create-issue` | 创建 Issue | Discuss |
| `/start-dev <issue>` | 开发指定 issue | Develop |
| `/review <pr>` | Review 指定 PR | Review |
| `/revise <pr>` | 修正指定 PR | Revise |

---

## 9. Error Handling

| Error | Description | Recovery |
|-------|-------------|----------|
| `project_not_found` | 目录不存在 | Human 确认路径 |
| `git_remote_not_found` | 无 git remote | Human 确认目录 |
| `github_auth_failed` | GitHub 认证失败 | 检查 GITHUB_TOKEN |
| `issue_not_found` | Issue 不存在 | Human 确认 URL |
| `pr_not_found` | PR 不存在 | Human 确认 URL |
| `llm_error` | LLM 调用失败 | 自动重试 3 次 |

---

## 10. Security

- GitHub Token 从环境变量读取，不存储
- 不打印 token 到日志
- Git 操作在项目目录内执行
- 所有关键操作由 Human 执行，AI 无法直接修改代码
