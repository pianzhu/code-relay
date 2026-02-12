# Code Relay 项目分析 (Part 1/3): 核心概览

## 1. 项目定位
**Code Relay** 是一套专为 **AI Coding Agent** 设计的工作协议（Protocol）。
它的核心目标是让 AI Agent 能够：
- **跨会话工作**：克服 LLM 的“失忆”问题。
- **自我管理**：拥有标准化的启动、交接和状态管理流程。
- **安全可控**：在限定的文件范围内工作（Scope）。
- **全局视野**：天然支持多仓库（Multi-repo）协作，而非受限于单个仓库。

## 2. 解决的核心痛点
该项目针对 AI 辅助编程中的四个主要问题提出了解决方案：
1.  **跨会话失忆**：通过标准化的 `HANDOFF` 和 `CHECKPOINT` 机制，实现上下文的持久化和无损传递。
2.  **上下文窗口限制**：通过主动的结构化文档（而非被动的自动压缩），保留高价值信息。
3.  **缺乏工作流**：定义了从任务获取、分支管理到代码提交的标准流程 (`DEV-FLOW`)。
4.  **多仓库割裂**：通过 `orchestrator` 目录和 `RESOURCE-MAP.yml`，让 Agent 能同时感知和操作多个代码仓库（Frontend, Backend, Infra 等）。

## 3. 核心概念与机制

### 3.1 两种模式
项目提供了两种运作模式，核心差异在于**状态管理**的存储位置：
*   **GitHub 协作模式**：
    *   状态存储于 GitHub Issues 和 Project Boards。
    *   适合已有 GitHub 工作流的团队。
*   **Local 本地文件模式**：
    *   状态存储于本地文件 (`STATUS.yml`, `PROGRAM.md`)。
    *   适合离线工作或需要更细粒度文件控制 (`SCOPE`) 的场景。

### 3.2 关键机制
*   **Boot Sequence (启动序列)**：Agent 启动时按固定顺序加载配置（Core -> Dev Flow -> Resource -> Program），实现“自举”。
*   **HANDOFF / CHECKPOINT (交接与快照)**：
    *   **HANDOFF**: 会话结束时的结构化交接文档（已完成、进行中、下一步）。
    *   **CHECKPOINT**: 上下文即将填满时的中间状态快照，用于应对 Context Window 限制。
*   **SCOPE (范围控制)**：(Local 模式) 通过白名单机制限制 Agent 可写入的文件路径，防止误操作。
*   **Worktree (工作树)**：强制使用 `git worktree` 进行开发，保证主仓库整洁，支持多任务并行。

## 4. 目录结构逻辑
项目采用了“元仓库”的设计思路：
```text
root/
├── AGENTS.md                  # Agent 的入口指引
├── orchestrator/              # "大脑"：存放全局配置、协议和任务状态
│   ├── ALWAYS/                # 总是加载的核心配置 (Boot, Core, Resources)
│   └── PROGRAMS/              # (Local模式) 具体的任务定义和状态
└── repos/                     # "躯干"：实际的业务代码仓库 (frontend, backend...)
```
这种结构让 Agent 站在“上帝视角”俯瞰所有子仓库。
