# Humera 产品说明

## 1. 定位

Humera 是一个** Human 掌控的 AI 协作工具**。

核心原则：
- **Human 发起**：每个任务由 Human 显式发起（执行 slash command）
- **AI 执行**：Human 发起后，AI 执行任务的全部过程

---

## 2. Command Summary

| Command | Task | 说明 | Output |
|---------|------|------|--------|
| `/on <path>` | 切换项目 | 设置项目上下文，进入讨论模式 | 项目上下文 |
| `/new` | 创建 Issue | 把讨论结果创建为 GitHub Issue | GitHub Issue |
| `/fix <issue>` | 开发 | 读取 Issue，编码，push，创建 PR | GitHub PR |
| `/review <pr>` | 审查 | Review PR 代码，写入 GitHub Comment | — |
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
Human: 确认（回复"确认"或类似明确答复）
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

### 3.4 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/on <path>` | |
| 讨论迭代 | 提出需求 / 补充 / 修改 | 确认细节 / 确认理解 |
| 最终确认 | 回复"确认"或类似明确答复 | 输出确认文档 |
| 创建 Issue | 执行 `/new` | 创建 GitHub Issue |

---

## 4. Task: Create Issue

### 4.1 Input / Output

| | 内容 |
|---|------|
| **Input** | Discuss 阶段确认的需求文档 |
| **Output** | GitHub Issue |

### 4.2 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/new` | |
| 创建 | | 调用 GitHub API 创建 Issue |

---

## 5. Task: Develop（编码开发）

### 5.1 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub Issue URL |
| **Output** | GitHub PR |

### 5.2 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/fix <issue-url>` | |
| 读取 Issue | | 读取 GitHub Issue 详情 |
| 分析需求 | | 分析需求 |
| 编码实现 | | 编码实现 |
| push 代码 | | push 代码到分支 |
| 创建 PR | | 创建 GitHub PR |

---

## 6. Task: Review（代码审查）

### 6.1 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub PR URL |
| **Output** | 写入 GitHub PR Comment |

### 6.2 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/review <pr-url>` | |
| 读取代码 | | 读取 PR 代码 |
| 代码审查 | | 专业方式 review |
| 写入 Comment | | 写入 GitHub PR Comment |

### 6.3 Review 维度

| 维度 | 说明 |
|------|------|
| **正确性** | 逻辑是否正确 |
| **安全性** | 是否有安全漏洞 |
| **性能** | 是否有性能问题 |
| **代码风格** | 是否符合规范 |
| **测试覆盖** | 是否有必要的测试 |

---

## 7. Task: Revise（修正）

### 7.1 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub PR URL |
| **Output** | 更新的 GitHub PR |

### 7.2 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/revise <pr-url>` | |
| 读取反馈 | | 读取 PR 的 Review Comments |
| 分析反馈 | | 分析所有 review 反馈 |
| 修改代码 | | 修改代码 |
| push 代码 | | push 代码到远端 |

### 7.3 迭代说明

Review 和 Revise 可以重复迭代：
- Review 后可以 Revise
- Revise 后可以再次 Review
- 直到 Human 满意为止

---

## 8. 任务关系图

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
│                                              │                   │
│                                              │ 可以循环迭代      │
│                                              ▼                   │
│                                        /review ──→ ...         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```
