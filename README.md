# claude-code-team-architecture

A practical deep dive into Claude Code’s multi-agent “team” mechanism — focusing on how **tasks, messaging, and lifecycle management** are implemented via a **file-backed state store** and **tmux-based orchestration**.

## Reports
- English: `claude-code-team-architecture.en.md`
- 中文：`claude-code-team-architecture.zh.md`

## Scope
- Lead / teammates orchestration (tmux panes)
- File-backed state under `~/.claude/`
- Tasks as JSON files + locking + status transitions
- Dependency DAG: `blocks` / `blockedBy`
- Messaging: direct vs broadcast, delivery behavior
- Graceful shutdown & cleanup