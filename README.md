# Code Relay

[中文版](README.zh-CN.md)

A structured protocol for AI coding agents. Enables self-bootstrapping, cross-session memory, multi-repo global awareness, and safe scoped execution.

## The Problem

AI coding agents in real-world projects face four core issues:

1. **Cross-session amnesia** — Every conversation starts from zero. What was done, what was decided, what pitfalls were hit — all lost
2. **Limited context window** — Large tasks exceed the context limit and get auto-compressed, losing critical information uncontrollably
3. **No standard workflow** — The agent doesn't know how to pick up tasks, report progress, or hand off work
4. **Multi-repo blindness** — Traditional AGENTS.md lives inside a single repo. The agent can only see one repo and can't understand relationships between frontend, backend, and services

## Core Concepts

| Concept | Purpose |
|---------|---------|
| **Boot Sequence** | Agent loads configuration in a fixed order on startup, automatically restoring context |
| **HANDOFF / CHECKPOINT** | Structured context save & restore — solves the problem of large tasks exceeding the context window |
| **SCOPE** | Whitelist controls which files the agent can write to, preventing unintended changes (Local mode) |
| **Worktree Isolation** | Each task develops in an isolated git worktree, no interference between tasks |

### Native Multi-Repo Support

The traditional approach puts AGENTS.md in a single repo's root — the agent can only see that one repo. But real projects usually have multiple repositories: frontend, backend, CLI, infrastructure scripts, etc.

Code Relay's workspace structure sits naturally above all repositories:

```
your-project/                          # Workspace root
├── AGENTS.md                          # Agent entry point
├── orchestrator/                      # Global config (cross-repo)
│   └── ALWAYS/
│       └── RESOURCE-MAP.yml           # Global index of all repos + infrastructure
│
└── repos/                             # All code repositories
    ├── backend/                       # Backend (Node.js)
    ├── frontend/                      # Frontend (React)
    ├── mobile/                        # Mobile (Flutter)
    └── infra/                         # Infrastructure (Terraform)
```

The agent has a **global view**:

- Knows every repo's location, tech stack, and dependencies
- When changing a backend API, can check if frontend calls need updating
- Understands shared databases, caches, and other infrastructure
- Cross-repo search and refactoring
- A single task can span changes across multiple repos

This is not a monorepo — each repo is still an independent git repository with its own branches and CI. The workspace simply organizes them together, giving the agent a global information entry point.

### Why Not Just Compress

Most AI coding agents auto-compress (summarize) conversation history when approaching the context limit. The problems:

- Compress is lossy — key decision details, pitfalls encountered, intermediate reasoning all get lost
- Once triggered, you can't control what gets discarded
- After compression, the agent often repeats mistakes it already solved

**HANDOFF / CHECKPOINT is the alternative to compress:**

- **Proactively triggered** — You have the agent write a handoff document while context is still rich, instead of waiting for auto-compression
- **Structured save** — Not a vague "summary", but a complete state snapshot: what's done, what's in progress, what's next, what to watch out for
- **Recoverable** — If compress already triggered, you can roll back to the previous checkpoint and restore full context via the HANDOFF document
- **Zero information loss** — When the agent reads HANDOFF on next startup, the context quality far exceeds what remains after compression

## Two Modes

### GitHub Collaboration Mode

State management on GitHub Issues + Project Board.

Best for: Teams already using GitHub, zero additional infrastructure needed.

```bash
cp -r github/* your-project/
```

### Local File Collaboration Mode

State management on the local filesystem.

Best for: Platform-independent, offline work, SCOPE controls, and Sub-Agent orchestration.

```bash
cp -r local/* your-project/
```

### Comparison

| Capability | GitHub | Local |
|-----------|--------|-------|
| State management | Issue + Project Board | STATUS.yml |
| Task definition | Issue body | PROGRAM.md |
| Write scope control | — | SCOPE.yml |
| Context recovery | Issue comment `## HANDOFF` | workspace/HANDOFF.md |
| Global view | `gh project item-list` | Scan PROGRAMS/ directory |
| Sub-Agent orchestration | — | SUB-AGENT.md |
| Platform dependency | GitHub / GitLab | None |

## Quick Start

### 1. Copy

```bash
# Pick a mode
cp -r github/* your-project/
# or
cp -r local/* your-project/
```

### 2. Launch

Open the project in your AI coding agent. On first launch, the agent will:

1. Read `AGENTS.md`, enter the Boot Sequence
2. Find `RESOURCE-MAP.yml` is empty, ask about your project
3. You describe it (repos, tech stack, infrastructure, etc.), the agent generates the config automatically
4. Initialization complete, start working

On subsequent launches, the agent reads existing config and auto-restores context.

## Usage Examples

### Basic Workflow

#### GitHub Mode

```
User: Load context

Agent:
  1. Read core config from orchestrator/ALWAYS/
  2. gh project item-list → Show Board status
  3. Wait for user to pick a task

User: Continue #42

Agent:
  1. gh issue view 42 → Get task definition
  2. Read ## HANDOFF from Issue comments → Restore progress
  3. Create worktree, pull feature/42-xxx branch
  4. Develop
  5. Verify → commit + push + PR
  6. Issue auto-closes
```

#### Local Mode

```
User: Continue P-2026-001

Agent:
  1. Read core config from orchestrator/ALWAYS/
  2. Read PROGRAMS/P-2026-001/PROGRAM.md → Task definition
  3. Read SCOPE.yml → Confirm writable file scope
  4. Read workspace/HANDOFF.md → Restore progress
  5. Develop in worktree
  6. Update STATUS.yml
```

### Context Management (Core Scenario)

Large tasks can't be completed in a single session. The agent's context window is finite. The core value of this protocol is enabling **zero-loss** context transfer across multiple sessions.

#### Scenario 1: Proactive Save (Recommended)

When you notice the conversation is getting long, proactively have the agent save state:

```
User: Save progress, prepare for handoff

Agent (GitHub mode):
  1. Push all commits on current branch
  2. Leave a ## HANDOFF comment on the Issue:
     - Work completed
     - Tasks in progress
     - Next steps
     - Gotchas and pitfalls encountered

Agent (Local mode):
  1. Push current branch
  2. Update STATUS.yml (task progress)
  3. Write workspace/HANDOFF.md
  4. If context is particularly tight, also write workspace/CHECKPOINT.md (fuller snapshot)
```

Then open a new session, say "continue #42" or "continue P-2026-001", and the agent restores from the HANDOFF document.

#### Scenario 2: Recovery After Compress

If you didn't save in time and the agent's context was auto-compressed:

```
User: Context got compressed, roll back

Agent:
  1. Roll back to the state before compression (most tools support undo/rollback)
  2. Execute the HANDOFF flow with the restored full context (as in Scenario 1)
  3. Open new session, restore from HANDOFF

Result: Full context preserved via HANDOFF document, unaffected by compression
```

#### Scenario 3: Multi-Session Relay Development

A task spanning three days across multiple session switches:

```
Day 1 (Session A):
  - Completed database schema design and basic CRUD
  - HANDOFF before session ends: "Schema done, CRUD implemented, next: API layer"

Day 2 (Session B):
  - Read HANDOFF, skip completed work
  - Complete API layer + middleware
  - New HANDOFF: "API done, found auth logic needs refactoring, next..."

Day 3 (Session C):
  - Read latest HANDOFF, start directly from auth refactoring
  - Complete all work, submit PR

Each session switch, the agent gets complete, structured context from HANDOFF.
Not a vague "summary", but precise "done / in progress / next / watch out for".
```

## Directory Structure

```
code-relay/
├── README.md
│
├── github/                            # GitHub collaboration mode
│   ├── AGENTS.md                      # Agent entry point
│   └── orchestrator/
│       └── ALWAYS/
│           ├── BOOT.md                # Boot sequence
│           ├── CORE.md                # Work protocol
│           ├── DEV-FLOW.md            # Development flow
│           └── RESOURCE-MAP.yml       # Resource index
│
└── local/                             # Local file collaboration mode
    ├── AGENTS.md                      # Agent entry point
    └── orchestrator/
        ├── ALWAYS/
        │   ├── BOOT.md                # Boot sequence
        │   ├── CORE.md                # Work protocol
        │   ├── DEV-FLOW.md            # Development flow
        │   ├── SUB-AGENT.md           # Sub-Agent spec
        │   └── RESOURCE-MAP.yml       # Resource index
        └── PROGRAMS/
            └── _TEMPLATE/             # Copy when creating new tasks
                ├── PROGRAM.md         # Task definition template
                ├── STATUS.yml         # Status tracking template
                ├── SCOPE.yml          # Write scope template
                └── workspace/         # Working documents
```

## Design Principles

1. **Single source of truth** — State lives in one place only, no copies
2. **Agent self-bootstrap** — Agent autonomously loads all context on startup, no manual feeding required
3. **File-based communication** — Large content goes into files, not conversation, protecting the context window
4. **Least privilege** — SCOPE controls write access (Local mode)
5. **Progressive** — Start simple, enable advanced features as needed

## Compatibility

Code Relay is a set of Markdown and YAML files, compatible with any AI coding agent that can read project files:

- [OpenCode](https://github.com/anomalyco/opencode)
- Cursor
- Claude Code
- Windsurf
- GitHub Copilot
- Any tool supporting AGENTS.md / CLAUDE.md

## License

MIT
