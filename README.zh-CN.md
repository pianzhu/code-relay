# Code Relay

[English](README.md)

一套给 AI Coding Agent 的工作协议。让 Agent 能自举启动、跨会话保持记忆、在限定范围内安全工作。

## 解决什么问题

AI Coding Agent 在实际项目开发中有四个核心问题：

1. **跨会话失忆** — 每次对话从零开始，之前做了什么、决定了什么、踩过什么坑，全部丢失
2. **上下文窗口有限** — 大任务超出上下文上限后被自动压缩（compress），关键信息不可控地丢失
3. **没有标准工作流** — Agent 不知道如何获取任务、报告进度、交接工作
4. **多仓库割裂** — 传统的 AGENTS.md 放在单个仓库内，Agent 只能看到一个 repo，无法理解前后端、微服务之间的关系

## 核心概念

| 概念 | 作用 |
|------|------|
| **Boot Sequence** | Agent 启动后按固定顺序加载配置，自动恢复上下文 |
| **HANDOFF / CHECKPOINT** | 结构化的上下文保存与恢复，解决大任务超出上下文窗口的问题 |
| **SCOPE** | 白名单控制 Agent 可写入的文件范围，防止误操作（Local 模式） |
| **Worktree 隔离** | 每个任务在独立的 git worktree 中开发，互不干扰 |

### 天然支持多仓库

传统做法是把 AGENTS.md 放在单个仓库的根目录，Agent 只能看到这一个 repo。但实际项目通常有多个仓库 — 前端、后端、CLI、基础设施脚本等。

Code Relay 的工作区结构天然在所有仓库之上：

```
your-project/                          # 工作区根目录
├── AGENTS.md                          # Agent 入口
├── orchestrator/                      # 全局配置（跨仓库）
│   └── ALWAYS/
│       └── RESOURCE-MAP.yml           # 所有仓库 + 基础设施的全局索引
│
└── repos/                             # 所有代码仓库
    ├── backend/                       # 后端 (Node.js)
    ├── frontend/                      # 前端 (React)
    ├── mobile/                        # 移动端 (Flutter)
    └── infra/                         # 基础设施 (Terraform)
```

Agent 拥有**全局视野**：

- 知道所有仓库的位置、技术栈、依赖关系
- 改后端 API 时能同步检查前端调用是否需要更新
- 了解共享的数据库、缓存等基础设施
- 跨仓库搜索和重构
- 一个任务可以同时涉及多个仓库的改动

这不是 monorepo — 每个 repo 仍然是独立的 git 仓库，有自己的分支和 CI。工作区只是把它们组织在一起，给 Agent 一个全局的信息入口。

### 为什么不用 Compress

大多数 AI Coding Agent 在上下文接近上限时会自动 compress（压缩对话历史）。问题是：

- Compress 是有损的 — 关键决策细节、踩过的坑、中间推理过程会丢失
- 一旦触发，你无法控制丢弃了什么
- 压缩后 Agent 经常重复犯已经解决过的错误

**HANDOFF / CHECKPOINT 是 compress 的替代方案：**

- **主动触发** — 你在上下文还充足时主动让 Agent 写交接文档，而不是等系统自动压缩
- **结构化保存** — 不是简单的"摘要"，而是完整的状态快照：已完成什么、进行中什么、下一步做什么、有什么注意事项
- **可回退** — 如果已经触发了 compress，可以回退到上一个 checkpoint，用 HANDOFF 文档恢复完整上下文
- **零信息损失** — Agent 下次启动时读取 HANDOFF，获得的上下文质量远高于 compress 后的残留

## 两种模式

### GitHub 协作模式

状态管理在 GitHub Issue + Project Board 上。

适合：已经使用 GitHub 的团队，希望零额外基础设施。

```bash
cp -r github/* your-project/
```

### 本地文件协作模式

状态管理在本地文件系统。

适合：不依赖特定平台，需要离线工作，需要 SCOPE 控制和 Sub-Agent 编排。

```bash
cp -r local/* your-project/
```

### 对比

| 能力 | GitHub | Local |
|------|--------|-------|
| 状态管理 | Issue + Project Board | STATUS.yml |
| 任务定义 | Issue body | PROGRAM.md |
| 写入范围控制 | — | SCOPE.yml |
| 上下文恢复 | Issue comment `## HANDOFF` | workspace/HANDOFF.md |
| 全局视图 | `gh project item-list` | 扫描 PROGRAMS/ 目录 |
| Sub-Agent 编排 | — | SUB-AGENT.md |
| 平台依赖 | GitHub / GitLab | 无 |

## 快速开始

### 1. 复制

```bash
# 选一种模式
cp -r github/* your-project/
# 或
cp -r local/* your-project/
```

### 2. 启动

在 AI Coding Agent 中打开项目。Agent 首次启动时会：

1. 读取 `AGENTS.md`，进入 Boot Sequence
2. 发现 `RESOURCE-MAP.yml` 为空，询问你的项目情况
3. 你描述一下（仓库、技术栈、基础设施等），Agent 自动生成配置
4. 完成初始化，开始工作

之后每次启动，Agent 直接读取已有配置，自动恢复上下文。

## 使用示例

### 基本工作流

#### GitHub 模式

```
用户: 载入上下文

Agent:
  1. 读取 orchestrator/ALWAYS/ 下的核心配置
  2. gh project item-list → 展示 Board 状态
  3. 等待用户选择任务

用户: 继续 #42

Agent:
  1. gh issue view 42 → 获取任务定义
  2. 读取 Issue comments 中的 ## HANDOFF → 恢复进度
  3. 创建 worktree，拉取 feature/42-xxx 分支
  4. 开发
  5. 验证通过 → commit + push + PR
  6. Issue 自动关闭
```

#### Local 模式

```
用户: 继续 P-2026-001

Agent:
  1. 读取 orchestrator/ALWAYS/ 下的核心配置
  2. 读取 PROGRAMS/P-2026-001/PROGRAM.md → 任务定义
  3. 读取 SCOPE.yml → 确认可写文件范围
  4. 读取 workspace/HANDOFF.md → 恢复进度
  5. 在 worktree 中开发
  6. 更新 STATUS.yml
```

### 上下文管理（核心场景）

大任务不可能在一次会话中完成。Agent 的上下文窗口是有限的，这套协议的核心价值就是让你在多次会话之间**零损失**地传递上下文。

#### 场景 1：主动保存（推荐）

当你发现对话已经很长，主动让 Agent 保存状态：

```
用户: 先保存一下进度，准备交接

Agent (GitHub 模式):
  1. push 当前分支的所有 commit
  2. 在 Issue 留 ## HANDOFF comment：
     - 已完成的工作
     - 进行中的任务
     - 下一步计划
     - 注意事项和踩过的坑

Agent (Local 模式):
  1. push 当前分支
  2. 更新 STATUS.yml（任务进度）
  3. 写入 workspace/HANDOFF.md
  4. 如果上下文特别紧张，额外写 workspace/CHECKPOINT.md（更完整的快照）
```

然后开一个新会话，说"继续 #42"或"继续 P-2026-001"，Agent 从 HANDOFF 文档恢复，继续工作。

#### 场景 2：Compress 已触发，回退恢复

如果你没来得及主动保存，Agent 的上下文已经被自动压缩了：

```
用户: 上下文被压缩了，回退一下

Agent:
  1. 回退到 compress 之前的状态（大多数工具支持 undo/rollback）
  2. 在回退后的完整上下文中执行 HANDOFF 流程（如场景 1）
  3. 开新会话，从 HANDOFF 恢复

结果: 完整上下文通过 HANDOFF 文档保留，不受 compress 影响
```

#### 场景 3：跨会话接力开发

任务做了三天，经历了多次会话切换：

```
第 1 天 (会话 A):
  - 完成了数据库 schema 设计和基础 CRUD
  - 会话结束前写 HANDOFF："schema 已建好，CRUD 已实现，下一步做 API 层"

第 2 天 (会话 B):
  - 读取 HANDOFF，跳过已完成的部分
  - 完成 API 层 + 中间件
  - 会话结束前写新的 HANDOFF："API 完成，发现认证逻辑需要重构，下一步..."

第 3 天 (会话 C):
  - 读取最新 HANDOFF，直接从认证重构开始
  - 完成所有工作，提 PR

每次会话切换，Agent 都从 HANDOFF 获得完整的、结构化的上下文。
不是模糊的"摘要"，而是精确的"已完成 / 进行中 / 下一步 / 注意事项"。
```

## 目录结构

```
code-relay/
├── README.md
│
├── github/                            # GitHub 协作模式
│   ├── AGENTS.md                      # Agent 入口
│   └── orchestrator/
│       └── ALWAYS/
│           ├── BOOT.md                # 启动顺序
│           ├── CORE.md                # 工作协议
│           ├── DEV-FLOW.md            # 开发流程
│           └── RESOURCE-MAP.yml           # 资源索引
│
└── local/                             # 本地文件协作模式
    ├── AGENTS.md                      # Agent 入口
    └── orchestrator/
        ├── ALWAYS/
        │   ├── BOOT.md                # 启动顺序
        │   ├── CORE.md                # 工作协议
        │   ├── DEV-FLOW.md            # 开发流程
        │   ├── SUB-AGENT.md           # Sub-Agent 规范
        │   └── RESOURCE-MAP.yml           # 资源索引
        └── PROGRAMS/
            └── _TEMPLATE/             # 新建任务时复制
                ├── PROGRAM.md         # 任务定义模板
                ├── STATUS.yml         # 状态跟踪模板
                ├── SCOPE.yml          # 写入范围模板
                └── workspace/         # 工作文档目录
```

## 设计原则

1. **单一真相源** — 状态只在一个地方，不维护副本
2. **Agent 自举** — Agent 启动后自主加载所有上下文，不依赖人类手动喂信息
3. **文件通信** — 大段内容写入文件，不塞进对话，保护上下文窗口
4. **最小权限** — 通过 SCOPE 控制写入范围（Local 模式）
5. **渐进式** — 从简单开始，按需启用高级功能

## 兼容性

Code Relay 是一组 Markdown 和 YAML 文件，兼容所有支持读取项目文件的 AI Coding Agent：

- [OpenCode](https://github.com/anomalyco/opencode)
- Cursor
- Claude Code
- Windsurf
- GitHub Copilot
- 其他支持 AGENTS.md / CLAUDE.md 的工具

## License

MIT
