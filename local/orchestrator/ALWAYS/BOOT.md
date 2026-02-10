# Boot Sequence

## 首次启动

如果 `RESOURCE-MAP.yml` 内容为空（全部注释）：

1. 询问用户项目基本情况：仓库地址、技术栈、基础设施、部署方式
2. 根据用户描述生成 `RESOURCE-MAP.yml`
3. 根据项目特点调整 `AGENTS.md` 中的工作语言和风格
4. 继续正常启动流程

## 正常启动

用户指定 Program 后，按以下顺序加载：

## 加载顺序

1. `orchestrator/ALWAYS/CORE.md` — 核心工作协议
2. `orchestrator/ALWAYS/DEV-FLOW.md` — 开发流程规范
3. `orchestrator/ALWAYS/RESOURCE-MAP.yml` — 资源索引
4. `orchestrator/PROGRAMS/{program_id}/PROGRAM.md` — 任务定义
5. `orchestrator/PROGRAMS/{program_id}/STATUS.yml` — 当前状态
6. `orchestrator/PROGRAMS/{program_id}/SCOPE.yml` — 写入范围

## 加载完成后输出

```
Program: {名称}
目标: {一句话}
当前阶段: {阶段名}
下一步: {具体行动}
```

## 特殊情况

- 如果 Program 目录不存在，询问用户是否创建（从 `_TEMPLATE` 复制）
- 如果存在 `workspace/CHECKPOINT.md`，优先读取恢复上下文
- 如果存在 `workspace/HANDOFF.md`，读取上次交接内容

## 新建 Program

从模板创建：

```bash
cp -r orchestrator/PROGRAMS/_TEMPLATE orchestrator/PROGRAMS/P-YYYY-NNN-<name>
```

然后编辑 PROGRAM.md、SCOPE.yml，填入任务定义和写入范围。
