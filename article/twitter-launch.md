# 做 Agent 研发的人，是怎么用 AI 写代码的

我的工作是做 AI Agent 研发。日常就是跟 agent 的能力边界打交道 — 它能做什么、不能做什么、怎么让它做得更好。

2024 年 5 月，Claude 3.5 还没出来的时候，我就开始用 GPT-4 + Dify 自己搭 workflow 写代码了。后来 Cursor、Claude Code 这些工具成熟之后，自然也第一时间用到了自己的项目里。不是写 demo、不是跑 benchmark，是真的用它来写生产代码、改 bug、做重构、提 PR。

从 Dify workflow 到现在的 AI coding agent，用了一年多。我发现**最大的坑不是模型不够聪明，而是它记不住事**。

## 上下文是 agent 的命脉

做 agent 研发的人都知道，上下文管理是 agent 系统里最核心的问题之一。没有上下文，再强的模型也是无根之木。

AI coding agent 也一样。一个复杂功能做到一半，对话已经很长了。第二天开新会话 — 它什么都不记得。你得重新描述背景、重新解释决策、重新踩一遍昨天已经趟过的坑。

更糟糕的是 compress。大部分工具在上下文快满时会自动"压缩"对话历史。作为做 agent 的人，我太清楚这意味着什么 — 这是不可控的有损压缩。关键的设计决策、踩过的坑、中间推理链条，压缩之后大概率丢失。然后 agent 开始重复犯已经解决过的错误。

这个问题我做 agent 的时候就在解决。现在发现，用 agent 写代码时，同样的问题又出现了。

## 所以我用了做 agent 的思路来解决它

做 agent 系统时，我们怎么处理上下文？**结构化存储 + 按需加载**。不会把所有信息都塞进 prompt，而是在关键节点保存状态，下次需要时精确恢复。

同样的思路用在 AI coding agent 上：

**用 HANDOFF 替代 compress。**

在上下文还充足的时候，主动让 agent 写一份结构化的交接文档 — 已经完成了什么、正在做什么、下一步做什么、有什么坑要注意。然后开新会话，agent 读取 HANDOFF，直接恢复上下文继续工作。

这不是模糊的"摘要"，而是精确的状态快照。信息密度远高于 compress 后的残留。

如果 compress 已经触发了也不慌 — 回退到压缩之前的状态，执行一次 HANDOFF，再开新会话恢复。几乎不会出现上下文丢失。

一个复杂功能做三天，每天换会话，每次 agent 都能从交接文档里精确恢复到昨天的状态。和人类团队的交接逻辑一模一样。

## 另一个做 agent 的人才会注意到的问题

我的项目有多个仓库 — 后端服务、CLI 工具、前端。传统的 AGENTS.md 放在单个 repo 里，agent 只能看到这一个仓库。

但做 agent 的人知道，**agent 的能力取决于它能获取到多少信息**。如果它不知道其他仓库的存在，不了解前后端的依赖关系，改了一个 API 不会想到调用方也要改。

所以我的工作区结构是这样的：

```
my-project/
├── AGENTS.md              # Agent 入口
├── orchestrator/           # 全局配置（跨仓库）
│   └── ALWAYS/
│       └── RESOURCE-MAP.yml  # 所有仓库 + 基础设施的全局索引
└── repos/
    ├── backend/
    ├── frontend/
    └── mobile/
```

Agent 的配置在所有代码仓库之上，启动后自动加载全局视图。它知道每个仓库的位置、技术栈、依赖关系和基础设施。这不是 monorepo — 每个 repo 仍然独立，工作区只是给 agent 一个全局的信息入口。

## 一年多趟出来的工程化方案

### Boot Sequence — Agent 自举

每次打开项目，agent 按固定顺序加载配置：工作协议 → 开发流程 → 资源索引 → 任务状态。不需要你手动喂背景信息，agent 自己读文件、自己恢复上下文、自己知道接下来该做什么。

第一次使用时 RESOURCE-MAP.yml 是空的，agent 会主动问你项目情况（几个仓库、什么技术栈、怎么部署），然后自动生成配置。之后每次启动都是全自动的。

### Worktree 隔离 — 每个任务独立开发

每个任务在独立的 git worktree 中进行，不在主仓库上切分支：

```bash
git worktree add repos/backend-auth -b feature/42-auth main
```

好处是多个任务可以并行，互不干扰。主仓库始终保持干净。做完之后清理 worktree 就行，和主仓库完全隔离。

### Sub-Agent — 不膨胀上下文的前提下做更多事

单个 agent 的上下文窗口是固定的。如果所有工作都塞在一个对话里 — 分析代码、写方案、改文件、跑测试 — 上下文很快就满了。

Sub-Agent 的思路是：**主 agent 只做规划和决策，具体的重活委托给 sub-agent 做，结果通过 workspace 文件传回来。**

比如你说"分析一下所有模块的依赖关系，顺便检查测试覆盖率"，主 agent 把它拆成两个任务，分别委托给两个 sub-agent。每个 sub-agent 独立执行，把分析报告写入 `workspace/` 目录，然后只给主 agent 返回 4 行摘要：

```
状态：已完成
报告：workspace/1.1-dependency-report.md
产出：1 个文件
决策点：无
```

主 agent 的上下文里只多了 4 行，但实际上完成了一个需要大量代码阅读和分析的任务。如果把这些工作全部塞进主 agent 的对话，上下文早就爆了。

本质上，Sub-Agent 是上下文窗口的**扩容器** — 主 agent 的上下文负责全局规划，sub-agent 的上下文负责具体执行，通过文件系统做信息中转。每个 agent 的上下文都不会被撑爆，但整体完成的工作量远超单个 agent 的上限。

### SCOPE 控制 — 限制 Agent 的写入范围

本地文件模式下，每个任务有一个 SCOPE.yml，白名单指定 agent 可以写哪些文件：

```yaml
write:
  - repos/backend/src/auth/
  - repos/backend/src/middleware/

forbidden:
  - .env
```

超出范围的文件，agent 不能擅自修改，必须先问你。这是我做 agent 研发养成的习惯 — **给 agent 的权限应该刚好够用，不多不少**。

## 开源了

这套工作流从最早的 Dify workflow 时代就开始摸索，到现在迭代了很多版。现在整理成了一个开源协议叫 **Code Relay**。

就是一组 markdown 和 yaml 文件，不是 CLI，不是 SDK。复制到你的项目里就能用。兼容 Cursor、Claude Code、OpenCode、Windsurf 等所有主流工具。

两种模式：
- **GitHub 模式** — 任务管理在 Issue + Project Board 上，适合已经用 GitHub 的团队
- **本地文件模式** — 不依赖任何平台，适合离线工作或不想绑定特定平台

两种模式都支持 HANDOFF、SCOPE、Sub-Agent、Worktree 等全部核心能力，区别只是任务状态存在哪里。

如果你也在认真用 AI coding agent 做项目，欢迎试试。有问题直接提 Issue。

GitHub: https://github.com/yan5xu/code-relay
