# Code Relay 项目分析 (Part 2/3): 核心流程详解

## 1. 启动流程 (Boot Sequence)
Agent 的启动不是随意的，而是遵循严格的“加载协议”，确保 Agent 在接管任务前具备所有必要信息。

**加载顺序：**
1.  **`AGENTS.md`**: 入口文件，引导 Agent 进入 Orchestrator。
2.  **`CORE.md`**: 加载核心工作协议（Scope 控制、状态持久化规则）。
3.  **`DEV-FLOW.md`**: 加载开发规范（Git Flow、Commit 格式）。
4.  **`RESOURCE-MAP.yml`**: 加载全局资源地图（了解有哪些仓库、技术栈、基础设施）。
5.  **`PROGRAM.md` & `STATUS.yml`**: 加载具体任务定义和当前进度。
6.  **`HANDOFF.md` / `CHECKPOINT.md`**: 恢复上一轮会话的上下文。

**设计意图**：通过分层加载，Agent 先“懂规矩”（Core/Dev-Flow），再“看地图”（Resource），最后“干活”（Program），实现 Agent 的**自举（Bootstrap）**。

## 2. 上下文管理机制 (Context Management)
这是 Code Relay 最核心的创新点，旨在解决 LLM 的 Context Window 限制。

### 2.1 结构化对象
*   **HANDOFF (交接棒)**：用于会话间的传递。
    *   包含：`当前状态`、`分支信息`、`下一步计划`、`注意事项`。
    *   作用：让 Agent 在新会话开始时，能像接力赛跑一样无缝继续。
*   **CHECKPOINT (快照)**：用于会话内的“存档”。
    *   包含：`目标`、`已完成/进行中`、`关键决策`、`文件变更列表`。
    *   作用：当上下文即将填满时，把内存中的状态“Dump”到硬盘，防止信息被 Compress 算法有损压缩。

### 2.2 策略：主动 vs 被动
*   **主动保存 (推荐)**：在上下文充裕时，Agent 主动将状态写入文件。这是**无损**的。
*   **回退恢复**：如果触发了系统的自动压缩（Context Compression），Code Relay 建议回退（Undo）到压缩前，执行主动保存，然后开启新会话读取文件。这避免了依赖模糊的压缩摘要。

## 3. 开发工作流 (Dev Flow)
项目强制使用 **Git Worktree** 模式，而非传统的 Branch 切换。

**标准作业程序 (SOP)：**
1.  **确认任务**：读取 `PROGRAM.md`。
2.  **创建环境**：
    ```bash
    cd repos/<target-repo>
    git worktree add ../repos/<repo>-<feature> -b feature/<name> main
    ```
    *   *优势*：保持主仓库（`main`）干净；多个任务（Feature, Bugfix）物理隔离，互不干扰；Agent 不会混淆不同任务的文件。
3.  **开发与验证**：在新建的文件夹中修改代码、运行测试。
4.  **提交与合并**：
    *   遵循 Conventional Commits (`feat: ...`).
    *   创建 PR -> 合并 -> 删除 Worktree。
5.  **状态更新**：修改 `STATUS.yml`，记录进度。

## 4. 范围控制 (Scope Control - Local模式)
在 `orchestrator/PROGRAMS/<id>/SCOPE.yml` 中定义白名单。
*   **`write`**: 允许 Agent 修改的文件/目录。
*   **`forbidden`**: 严禁触碰的区域。
*   **机制**：Agent 在生成代码修改指令前，必须自检是否在 SCOPE 允许范围内。这为 Agent 在本地文件系统上“裸奔”穿上了防护服。
