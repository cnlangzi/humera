# Humera 产品说明

## 1. 定位

Humera 是一个** Human 掌控的 AI 协作工具**。

核心原则：
- **Human 发起**：每个任务由 Human 显式发起（执行 slash command）
- **AI 执行**：Human 发起后，AI 执行任务的全部过程
- **Human 确认**：对于需要确认的任务，Human 确认结果后继续

---

## 2. Command Summary

| Command | Task | 说明 | Output |
|---------|------|------|--------|
| `/on <path>` | 切换项目 | 设置项目上下文，进入讨论模式 | 项目上下文 |
| `/new` | 创建 Issue | 把讨论结果创建为 GitHub Issue | GitHub Issue |
| `/fix <issue>` | 开发 | 读取 Issue，编码，push，创建 PR | GitHub PR |
| `/review <pr>` | 审查 | Review PR 代码，写入 GitHub Comment | PR Comment |
| `/revise <pr>` | 修正 | 读取 Review 反馈，修改代码，push | Updated PR |

---

## 3. Task: Discuss（需求讨论）

**起点：所有新功能和 Bug 都从这里开始**

### 3.1 Input / Output

| | 内容 |
|---|------|
| **Input** | Human 的消息（新功能需求或 Bug 描述） |
| **Output** | 确认文档（暂存），等待 `/new` |

### 3.2 流程

```
Human: /on ~/code/myapp
       │
       ▼
Project context set
自动进入 Discuss 模式
       │
       ▼
Human: 提出需求或问题
       │
       ▼
AI: 确认具体细节（功能点、范围、技术方案）
       │
       ▼
Human: 补充 / 修改
       │
       ▼
AI: 确认理解是否正确
       │
       ▼
← 循环直到双方确认 →
       │
       ▼
AI: 整合输出最终确认文档
       │
       ▼
Human: 确认最终文档
       │
       ▼
等待 Human 执行 /new
```

### 3.3 产物格式

```
═══════════════════════════════════════
需求确认文档
═══════════════════════════════════════

标题：[标题]

功能：
- [具体功能点]

范围：
- 包含：[包含哪些]
- 不包含：[不包含哪些]

技术方案：
- [实现方式]

验收标准：
- [ ] [标准 1]
- [ ] [标准 2]

═══════════════════════════════════════
确认后执行 /new 创建 Issue
═══════════════════════════════════════
```

### 3.4 边界

| | Human | AI |
|---|-------|-----|
| **发起** | 执行 `/on <path>` | |
| **需求提出** | 提出原始需求 | |
| **需求确认** | | 确认细节，输出确认文档 |
| **需求补充** | 补充 / 修改 | |
| **理解确认** | | 确认理解是否正确 |
| **最终确认** | 确认最终文档 | |
| **创建 Issue** | 执行 `/new` | 创建 GitHub Issue |

---

## 4. Task: Develop（编码开发）

### 4.1 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub Issue URL |
| **Output** | GitHub PR（代码已 push） |

### 4.2 流程

```
Human: /fix <issue-url>
       │
       ▼
AI: 读取 GitHub Issue 详情
       │
       ▼
AI: 分析需求
       │
       ▼
AI: 编码实现
       │
       ▼
AI: push 代码到分支
       │
       ▼
AI: 创建 GitHub PR
       │
       ▼
AI: 输出 PR URL
```

### 4.3 边界

| | Human | AI |
|---|-------|-----|
| **发起** | 执行 `/fix <issue-url>` | |
| **读取 Issue** | | 读取 GitHub Issue 详情 |
| **分析需求** | | 分析需求 |
| **编码** | | 编码实现 |
| **push 代码** | | push 代码到分支 |
| **创建 PR** | | 创建 GitHub PR |

### 4.4 说明

- Human 只需执行 `/fix`，之后 AI 自主完成全部过程
- 无中间确认环节
- 后续由 Human 在 GitHub 上 review PR

---

## 5. Task: Review（代码审查）

### 5.1 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub PR URL |
| **Output** | GitHub PR Comment（review 意见） |

### 5.2 流程

```
Human: /review <pr-url>
       │
       ▼
AI: 读取 PR 代码
       │
       ▼
AI: 专业方式 review
       │
       ▼
AI: 把 review 意见写入 GitHub PR Comment
       │
       ▼
AI: 输出 review 摘要
```

### 5.3 Review 维度

| 维度 | 说明 |
|------|------|
| **正确性** | 逻辑是否正确 |
| **安全性** | 是否有安全漏洞 |
| **性能** | 是否有性能问题 |
| **代码风格** | 是否符合规范 |
| **测试覆盖** | 是否有必要的测试 |

### 5.4 边界

| | Human | AI |
|---|-------|-----|
| **发起** | 执行 `/review <pr-url>` | |
| **读取代码** | | 读取 PR 代码 |
| **代码审查** | | 专业方式 review |
| **写入 Comment** | | 写入 GitHub PR Comment |

### 5.5 说明

- Human 只需执行 `/review`，之后 AI 自动完成
- Review 意见直接写入 GitHub PR，不在 IM 频道
- 后续由 Human 根据 review 决定是否 `/revise`

---

## 6. Task: Revise（修正）

### 6.1 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub PR URL |
| **Output** | 更新的 GitHub PR（代码已 push） |

### 6.2 流程

```
Human: /revise <pr-url>
       │
       ▼
AI: 读取 PR 的 Review Comments
       │
       ▼
AI: 分析所有 review 反馈
       │
       ▼
AI: 修改代码
       │
       ▼
AI: push 代码到远端
       │
       ▼
AI: 输出修正摘要
```

### 6.3 边界

| | Human | AI |
|---|-------|-----|
| **发起** | 执行 `/revise <pr-url>` | |
| **读取反馈** | | 读取 PR 的 Review Comments |
| **分析反馈** | | 分析所有 review 反馈 |
| **修改代码** | | 修改代码 |
| **push 代码** | | push 代码到远端 |

### 6.4 说明

- Human 只需执行 `/revise`，之后 AI 自动完成
- 读取此 PR 的所有 Review Comments 作为修改依据

---

## 7. 任务关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   /on <path> ──────────────────── 起点 + Discuss 模式            │
│      │                                                         │
│      │ Human 与 AI 迭代讨论                                     │
│      │                                                         │
│      ▼                                                         │
│   /new ─────────────────────────→ GitHub Issue                  │
│      │                                                         │
│      │ Human 执行                                               │
│      ▼                                                         │
│   /fix <issue> ──────────────────→ GitHub PR                    │
│                                              │                   │
│                                              │ Human 在 GitHub   │
│                                              │ review PR         │
│                                              ▼                   │
│                                        /review ──→ PR Comment   │
│                                              │                   │
│                                              │ Human 执行        │
│                                              ▼                   │
│                                        /revise ──→ Updated PR  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```
