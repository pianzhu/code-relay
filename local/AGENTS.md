# AGENTS.md

此文件为 AI Coding Agent 在本工作区工作时提供指导。

---

## 工作语言与风格

- 简洁直接
- 代码注释清晰
- （Agent 首次启动时会根据用户偏好调整此段）

---

## Boot Sequence

Agent 启动后：

1. **用户指定 Program**（如 "继续 P-2026-001" 或 "新 Program: xxx"）
2. **读取** `orchestrator/ALWAYS/BOOT.md`
3. **按指示加载**相关文件
4. **输出**当前状态和下一步

如果用户未指定 Program，扫描 `orchestrator/PROGRAMS/` 展示任务列表，询问要做什么。

---

## 目录结构

```
your-project/
├── AGENTS.md                      # 本文件
├── orchestrator/
│   ├── ALWAYS/                    # 核心配置（每次必读）
│   │   ├── BOOT.md                # 启动加载顺序
│   │   ├── CORE.md                # 工作协议
│   │   ├── DEV-FLOW.md            # 开发流程规范
│   │   ├── SUB-AGENT.md           # Sub-Agent 规范（按需使用）
│   │   └── RESOURCE-MAP.yml           # 资源索引
│   │
│   └── PROGRAMS/                  # 开发任务
│       └── P-YYYY-NNN-name/       # 每个 Program 一个目录
│           ├── PROGRAM.md         # 任务定义
│           ├── STATUS.yml         # 状态跟踪
│           ├── SCOPE.yml          # 写入范围
│           └── workspace/         # 工作文档
│
└── repos/                         # 代码仓库（可选，多仓库时使用）
```

---

## 快速命令

- "继续 P-2026-001" — 加载并继续该 Program
- "新 Program: xxx" — 创建新的开发任务
- "委托: xxx" — 使用 Sub-Agent 执行任务

---

## 状态来源

- **Programs 列表**: 扫描 `orchestrator/PROGRAMS/` 目录
- **Program 状态**: 读取各 Program 下的 `STATUS.yml`
- **仓库信息**: 读取 `orchestrator/ALWAYS/RESOURCE-MAP.yml`

不要在此文件维护状态副本，直接从源文件读取。
