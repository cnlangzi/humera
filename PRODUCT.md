# Humera 产品说明

## 1. 定位

Humera 是一个** Human 掌控的 AI 协作工具**。

Human 是唯一的掌控者和执行者。AI 是协作助手，负责确认、建议、执行过程，但不执行最终产物。

---

## 2. Human 与 AI 的关系

### 2.1 角色定义

| 角色 | 职责 |
|------|------|
| **Human** | 掌控者：发起任务、确认结果、执行所有 slash command |
| **AI** | 协作助手：确认细节、提供建议、执行过程、输出结果 |

### 2.2 边界

| AI 能做的 | AI 不能做的 |
|-----------|-------------|
| 确认需求细节 | 创建 Issue |
| 提供建议 | 提交代码 |
| 执行过程（编码、review） | 合并 PR |
| 输出结果到 GitHub | 执行 slash command |

---

## 3. 任务类型

Humera 定义了 4 种任务类型：

| Task | 发起 | 执行者 | 产物 |
|------|------|--------|------|
| **Discuss** | Human 执行 `/on` | AI + Human 迭代 | 确认文档（暂存） |
| **Create Issue** | Human | AI | GitHub Issue |
| **Develop** | Human | AI | GitHub PR |
| **Review** | Human | AI | PR Comment |
| **Revise** | Human | AI | Updated PR |

---

## 4. Task: Discuss（需求讨论）

### 4.1 描述

Human 执行 `/on <path>` 切换项目后，自动进入讨论模式。Human 提出需求或问题，AI 不断确认细节，Human 不断补充，直到双方确认无误。

### 4.2 Input / Output

| | 内容 |
|---|------|
| **Input** | Human 的任何消息（可以是新功能或 Bug） |
| **Output** | 确认文档（暂存），等待后续 `/create-issue` |

### 4.3 流程

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
Discuss 结束，等待 Human 执行 /create-issue
```

### 4.4 产物格式

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
确认后执行 /create-issue 创建 Issue
═══════════════════════════════════════
```

### 4.5 边界

| 边界 | 说明 |
|------|------|
| **起点** | Human 执行 `/on <path>` |
| **结束条件** | Human 确认最终文档 |
| **后续动作** | Human 执行 `/create-issue` |

---

## 5. Task: Create Issue（创建 Issue）

### 5.1 描述

Human 执行 `/create-issue`，AI 把当前讨论的结果创建为 GitHub Issue。

### 5.2 Input / Output

| | 内容 |
|---|------|
| **Input** | Discuss 阶段确认的需求文档 |
| **Output** | GitHub Issue（包含标题、描述） |

### 5.3 流程

```
Human: /create-issue
       │
       ▼
AI: 读取当前讨论的确认文档
       │
       ▼
AI: 创建 GitHub Issue
       │
       ▼
AI: 输出 Issue URL
```

### 5.4 边界

| 边界 | 说明 |
|------|------|
| **前置条件** | 必须有 Discuss 阶段的确认文档 |
| **执行前提** | Human 主动执行 `/create-issue` |
| **后续动作** | Human 可以执行 `/fix` 开发此 Issue |

---

## 6. Task: Develop（编码开发）

### 6.1 描述

Human 指定要开发的 Issue，AI 读取 Issue 内容，编码实现，push 代码，创建 PR。

### 6.2 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub Issue URL |
| **Output** | GitHub PR（代码已 push） |

### 6.3 流程

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

### 6.4 边界

| 边界 | 说明 |
|------|------|
| **前置条件** | Issue 已存在 |
| **Human 动作** | 只执行 `/fix <issue-url>`，之后 AI 自主完成 |
| **无中间确认** | Develop 开始后不等待 Human 确认 |
| **后续动作** | Human 在 GitHub 上 review PR，决定是否需要 `/revise` |

---

## 7. Task: Review（代码审查）

### 7.1 描述

Human 指定要 review 的 PR，AI 读取代码，用专业方式审查，把意见写入 GitHub PR Comment。

### 7.2 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub PR URL |
| **Output** | GitHub PR Comment（review 意见） |

### 7.3 流程

```
Human: /review <pr-url>
       │
       ▼
AI: 读取 PR 代码
       │
       ▼
AI: 专业方式 review（正确性、安全性、性能、风格、测试覆盖）
       │
       ▼
AI: 把 review 意见写入 GitHub PR Comment
       │
       ▼
AI: 输出 review 摘要
```

### 7.4 Review 维度

| 维度 | 说明 |
|------|------|
| **正确性** | 逻辑是否正确 |
| **安全性** | 是否有安全漏洞 |
| **性能** | 是否有性能问题 |
| **代码风格** | 是否符合规范 |
| **测试覆盖** | 是否有必要的测试 |

### 7.5 边界

| 边界 | 说明 |
|------|------|
| **前置条件** | PR 已存在 |
| **Human 动作** | 只执行 `/review <pr-url>`，之后 AI 自动完成 |
| **输出位置** | Review 意见写入 GitHub PR，不在 IM 频道 |
| **后续动作** | Human 根据 review 决定是否 `/revise` |

---

## 8. Task: Revise（修正）

### 8.1 描述

Human 指定要修正的 PR，AI 读取 PR 的 Review Comments，分析反馈，修改代码，push 到远端。

### 8.2 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub PR URL |
| **Output** | 更新的 GitHub PR（代码已 push） |

### 8.3 流程

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

### 8.4 边界

| 边界 | 说明 |
|------|------|
| **前置条件** | PR 已有 Review Comments |
| **Human 动作** | 只执行 `/revise <pr-url>`，之后 AI 自动完成 |
| **读取范围** | 读取此 PR 的所有 Review Comments |
| **结束条件** | 代码 push 到远端 |

---

## 9. 任务关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   /on <path> ──────────────────── 起点 + Discuss 模式            │
│      │                                                         │
│      │ Human 与 AI 迭代讨论                                     │
│      │                                                         │
│      ▼                                                         │
│   /create-issue ────────────────→ GitHub Issue                  │
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

---

## 10. Command Summary

| Command | Task | Input | Output |
|---------|------|-------|--------|
| `/on <path>` | 切换项目 + Discuss | 目录路径 | 项目上下文 + 讨论模式 |
| `/create-issue` | 创建 Issue | 确认文档 | GitHub Issue |
| `/fix <issue>` | 编码 | Issue URL | GitHub PR |
| `/review <pr>` | 审查 | PR URL | PR Comment |
| `/revise <pr>` | 修正 | PR URL | Updated PR |
