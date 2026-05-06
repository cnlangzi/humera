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

### 2.1 前置条件

| Command | 前置条件 |
|---------|---------|
| `/on` | 无 |
| Discuss（普通消息） | 如果没有 project context，AI 要求 Human 先执行 `/on` |
| `/new` | 必须先执行 `/on`（依赖项目上下文中的 repo 信息） |
| `/fix` | 必须先执行 `/on` |
| `/review` | 必须先执行 `/on` |
| `/revise` | 必须先执行 `/on` |

### 2.2 路径格式

`/on <path>` 中的 path 支持以下格式：

| 格式 | 示例 | 解析 |
|------|------|------|
| 绝对路径 | `/Users/bin/code/myapp` | 直接使用 |
| `~` 开头 | `~/code/myapp` | `~/` 替换为 homedir |
| `~user` 开头 | `~bob/code/myapp` | `~user` 替换为该用户的 home |
| `./` 或 `../` 开头 | `./code/myapp` | 相对当前工作目录 |
| 非以上格式 | `code/myapp` | 默认视为 `~/code/myapp` |

### 2.3 项目上下文

执行 `/on <path>` 时：

1. **解析路径**：根据上表规则展开为绝对路径
2. **验证目录**：必须是存在的目录
3. **读取 git remote**：从 `.git/config` 读取 origin remote，解析出 `owner/repo`
4. **设置上下文**：保存 `workdir` 和 `repo` 信息

### 2.4 参数格式

| Command | 参数格式 | 示例 |
|---------|---------|------|
| `/fix` | Issue URL、完整 `owner/repo#number`、或仅 `number` | `/fix https://github.com/owner/repo/issues/123` 或 `/fix owner/repo#123` 或 `/fix 123` |
| `/review` | PR URL、完整 `owner/repo#number`、或仅 `number` | `/review https://github.com/owner/repo/pull/456` 或 `/review owner/repo#456` 或 `/review 456` |
| `/revise` | PR URL、完整 `owner/repo#number`、或仅 `number` | `/revise https://github.com/owner/repo/pull/456` 或 `/revise owner/repo#456` 或 `/revise 456` |

**说明：**
- 提供仅 `number` 时，`owner/repo` 从项目上下文（`/on` 设置的 repo）中获取

---

## 3. Task: Discuss（需求讨论）

**起点：所有新功能和 Bug 都从这里开始**

### 3.1 Input / Output

| | 内容 |
|---|------|
| **Input** | Human 的消息（新功能需求或 Bug 描述） |
| **Output** | 确认文档（`/new` 时生成），等待 `/new` |

### 3.2 流程

```
Human: /on ~/code/myapp
       │
       ▼
解析路径 → 验证目录 → 读取 git remote → 设置项目上下文
       │
       ▼
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
AI: 确认理解 + 继续讨论
       │
       ▼
← 循环直到 Human 认为讨论充分 →
       │
       ▼
Human: 执行 /new
```

### 3.3 无项目上下文时的行为

```
Human: [任意消息，但没有先执行 /on]
       │
       ▼
AI: 请先执行 /on <path> 设置项目上下文
       │
       ▼
等待 Human 执行 /on
```

设置项目上下文后，继续 Discuss 模式。

### 3.4 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/on <path>` | |
| 讨论迭代 | 提出需求 / 补充 / 修改 | 确认细节 / 更新文档 / 输出当前方案 |
| 创建 Issue | 执行 `/new` | 读取确认文档，创建 GitHub Issue |

---

## 4. Task: Create Issue

### 4.1 Input / Output

| | 内容 |
|---|------|
| **Input** | 当前 session 的讨论内容 |
| **Output** | GitHub Issue |

### 4.2 前置条件

- 必须先执行 `/on <path>` 设置项目上下文
- 必须有 Discuss 阶段的讨论内容
- `repo` 信息从项目上下文中获取

### 4.3 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/new` | |
| 整理文档 | | 从 session 提取讨论内容，整理成 Issue 格式 |
| 创建 | | 调用 GitHub API 创建 Issue |

### 4.4 Issue 格式

创建 Issue 时，AI 从讨论内容中整理出以下格式：

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

---

## 5. Task: Develop（编码开发）

### 5.1 Input / Output

| | 内容 |
|---|------|
| **Input** | GitHub Issue URL |
| **Output** | GitHub PR |

### 5.2 前置条件

- 必须先执行 `/on <path>` 设置项目上下文
- `repo` 信息从项目上下文中获取
- Issue 必须在该 repo 中存在

### 5.3 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/fix <issue>` | |
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

### 6.2 前置条件

- 必须先执行 `/on <path>` 设置项目上下文
- `repo` 信息从项目上下文中获取
- PR 必须在该 repo 中存在

### 6.3 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/review <pr>` | |
| 读取代码 | | 读取 PR 代码 |
| 代码审查 | | 专业方式 review |
| 写入 Comment | | 写入 GitHub PR Comment |

### 6.4 Review 维度

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

### 7.2 前置条件

- 必须先执行 `/on <path>` 设置项目上下文
- `repo` 信息从项目上下文中获取
- PR 必须有 Review Comments

### 7.3 角色

| | Human | AI |
|---|-------|-----|
| 发起 | 执行 `/revise <pr>` | |
| 读取反馈 | | 读取 PR 的 Review Comments |
| 分析反馈 | | 分析所有 review 反馈 |
| 修改代码 | | 修改代码 |
| push 代码 | | push 代码到远端 |

### 7.4 迭代说明

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
│      │ AI 更新 .humera/discuss.md                              │
│      │                                                         │
│      │ Human 认为讨论充分                                       │
│      │                                                         │
│      ▼                                                         │
│   /new ─────────────────────────→ GitHub Issue                  │
│      │                                                         │
│      ▼                                                         │
│   /fix <issue> ──────────────────→ GitHub PR                   │
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


