# Code Relay 项目分析 (Part 3/3): 高级机制与总结

## 1. Sub-Agent 编排 (Sub-Agent Orchestration)
当任务过于复杂，单个 Agent 难以应对时，Code Relay 提供了基于**文件通信**的子代理协作协议。

### 1.1 核心原则
*   **文件作为总线**：主 Agent 和 Sub-Agent 之间**不通过对话**传递大量信息（如代码、长文档），而是通过文件系统。
*   **极简回报**：Sub-Agent 完成任务后，只向主 Agent 回报 4 行摘要：
    1.  `状态` (成功/失败)
    2.  `报告路径` (workspace/xxx.md)
    3.  `产出文件数`
    4.  `决策点` (需要主 Agent 决定的事项)

### 1.2 任务编号系统
采用 `X.Y` 格式管理并行与串行任务：
*   **X (阶段)**：不同 X 的任务必须串行（如 1.x 做完才能做 2.x）。
*   **Y (并行)**：同一 X 下的子任务（如 1.1, 1.2）可以并行执行。
*   *场景*：同时分析前端和后端的依赖（并行），分析完后统一重构 API（串行）。

## 2. 全局资源映射 (Resource Mapping)
`RESOURCE-MAP.yml` 是 Agent 的“世界地图”，解决了多仓库协作的痛点。

### 2.1 解决的问题
传统的 Coding Agent 通常被限制在当前打开的目录下。Code Relay 明确定义了：
*   **repos**: 所有相关代码仓库的物理路径和 Git 地址。
*   **infrastructure**: 数据库、缓存、消息队列的连接信息（非敏感）。
*   **deploy**: 部署方式和触发条件。

### 2.2 动态生成
该文件不需要用户手动编写。Agent 在首次 Boot 时，会询问用户项目情况，然后自动生成这份 YAML 配置。这体现了 **Agent-First** 的设计理念。

## 3. 总结与最佳实践

### 3.1 核心价值
Code Relay 不是一个软件，而是一套**协议 (Protocol)**。它不需要安装 Python 或 Node.js 环境，只需要 Agent 遵守一套 Markdown/YAML 定义的规则。这使得它兼容性极强（Cursor, GitHub Copilot, Claude Code 等均可用）。

### 3.2 最佳实践清单
1.  **勤写 HANDOFF**：不要等 Context 满了再保存，每一个小的 Milestone 完成后都主动要求 Agent 保存。
2.  **严守 SCOPE**：在 Local 模式下，务必配置 `SCOPE.yml`，防止 Agent 改错文件。
3.  **Worktree 隔离**：始终为每个 Program 创建独立的 git worktree，保持环境纯净。
4.  **结构化思考**：在 Program 开始前，强制 Agent 先写 `PROGRAM.md` 明确目标，避免无头苍蝇式开发。

### 3.3 适用场景推荐
*   **复杂重构**：涉及跨文件、跨仓库的改动。
*   **长期维护**：项目周期长，开发人员（或 Agent 会话）频繁切换。
*   **离线/安全敏感**：使用 Local 模式配合 Scope 控制，数据不出本地。
