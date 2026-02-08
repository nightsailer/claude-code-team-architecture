# Claude Code Multi-Agent Team Collaboration — Deep Architecture Analysis

> Generated: 2026-02-08
> Environment: Claude Code CLI (claude-opus-4-6)
> Analysis team: analysis-team (1 lead + 3 researchers)
> Verification team: verify-team (1 lead + 2 testers)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Team System](#2-team-system)
3. [Task System](#3-task-system)
4. [Agent Communication](#4-agent-communication)
5. [Agent Lifecycle](#5-agent-lifecycle)
6. [Storage Architecture](#6-storage-architecture)
7. [Live Examples](#7-live-examples)
8. [Prompt Engineering](#8-prompt-engineering)

---

## 1. Architecture Overview

Claude Code's multi-agent team system is a filesystem-based distributed agent coordination framework.

```
┌─────────────────────────────────────────────────┐
│                Team Lead (main session)           │
│  - Create team      (TeamCreate)                 │
│  - Create/assign tasks (TaskCreate/TaskUpdate)   │
│  - Spawn agents     (Task tool with team_name)   │
│  - Coordinate       (SendMessage)                │
│  - Shutdown agents  (shutdown_request)            │
│  - Cleanup          (TeamDelete)                 │
├─────────────────────────────────────────────────┤
│  Agent-A (tmux %14) │ Agent-B (tmux %15) │  ... │
│  Independent Claude  │ Independent Claude  │     │
│  Shared task list    │ Shared task list    │     │
│  SendMessage to lead │ SendMessage to lead │     │
└─────────────────────────────────────────────────┘
           │                    │
     ┌─────┴────────────────────┴─────┐
     │   Shared filesystem (~/.claude/) │
     │  ├── teams/{name}/config.json   │
     │  ├── tasks/{name}/*.json        │
     │  ├── session-env/{id}/          │
     │  ├── debug/{id}.txt             │
     │  └── shell-snapshots/           │
     └─────────────────────────────────┘
```

**Key characteristics:**
- **Process isolation**: Each agent runs in an independent tmux pane
- **File sharing**: Team and task data shared via `~/.claude/` filesystem
- **Message-driven**: Agents communicate asynchronously via SendMessage
- **Centralized coordination**: Team Lead manages task assignment and lifecycle

---

## 2. Team System

### 2.1 Team Creation

`TeamCreate` automatically generates:

| Resource | Path | Purpose |
|----------|------|---------|
| Team config | `~/.claude/teams/{name}/config.json` | Team metadata + member list |
| Task directory | `~/.claude/tasks/{name}/` | Shared task list |
| Task lock file | `~/.claude/tasks/{name}/.lock` | Concurrency control |

### 2.2 config.json Schema

```jsonc
{
  // === Top-level fields ===
  "name": "analysis-team",          // string: team name (unique identifier)
  "description": "Team description", // string: purpose
  "createdAt": 1770535107409,       // number: Unix timestamp (ms)
  "leadAgentId": "team-lead@analysis-team",  // string: lead's agentId
  "leadSessionId": "c93b690c-...",  // string: lead's session UUID

  // === Member list ===
  "members": [
    // --- Team Lead (auto-created) ---
    {
      "agentId": "team-lead@analysis-team",  // string: {name}@{team}
      "name": "team-lead",                    // string: identifier for messaging
      "agentType": "team-lead",               // string: agent type
      "model": "claude-opus-4-6",             // string: LLM model ID
      "joinedAt": 1770535107409,              // number: join timestamp
      "tmuxPaneId": "",                        // string: tmux pane (empty for lead)
      "cwd": "/path/to/project",              // string: working directory
      "subscriptions": []                      // array: message subscriptions
    },

    // --- Regular member (Task spawn) ---
    {
      "agentId": "researcher-config@analysis-team",
      "name": "researcher-config",             // used as SendMessage recipient
      "agentType": "general-purpose",          // determined by Task's subagent_type
      "model": "claude-opus-4-6",
      "prompt": "You are a member of ...",     // string: initial instructions
      "color": "blue",                         // string: UI color (blue/green/yellow/...)
      "planModeRequired": false,               // boolean: require plan approval
      "joinedAt": 1770535144590,
      "tmuxPaneId": "%14",                     // string: tmux pane identifier
      "cwd": "/path/to/project",
      "subscriptions": [],
      "backendType": "tmux",                   // string: runtime backend
      "isActive": true                         // boolean: currently active
    }
  ]
}
```

### 2.3 Field Reference

| Field | Type | Member-only | Description |
|-------|------|-------------|-------------|
| `agentId` | string | — | Format: `{name}@{team_name}`, globally unique |
| `name` | string | — | Human-readable name, used for messaging and task assignment |
| `agentType` | string | — | Agent type: `team-lead`, `general-purpose`, `Explore`, `Plan`, etc. |
| `model` | string | — | Underlying LLM model ID |
| `joinedAt` | number | — | Unix millisecond timestamp |
| `tmuxPaneId` | string | — | tmux pane ID (e.g. `%14`), empty for lead |
| `cwd` | string | — | Agent's working directory |
| `subscriptions` | array | — | Message subscription list (currently always empty) |
| `prompt` | string | ✓ | Initial task instructions (passed via Task tool) |
| `color` | string | ✓ | UI color identifier, assigned in join order (see 2.5) |
| `planModeRequired` | boolean | ✓ | Whether plan approval is required before execution |
| `backendType` | string | ✓ | Runtime backend (currently `tmux`) |
| `isActive` | boolean | ✓ | Whether the agent is currently active |

### 2.4 Team Naming

- **Named teams**: `TeamCreate` with explicit `team_name`, e.g. `analysis-team`
- **Anonymous teams**: UUID-named teams (e.g. `e132a923-...`) are auto-generated

### 2.5 Color Assignment Rules (Verified)

Member colors are assigned automatically by spawn order. Team Lead has no color:

| Join Order | Color | Verified (analysis-team) | Verified (verify-team) |
|------------|-------|--------------------------|------------------------|
| 1st member | `blue` | researcher-config | tester-01 |
| 2nd member | `green` | researcher-tasks | tester-02 |
| 3rd member | `yellow` | researcher-comms | — |

Inferred sequence: `blue → green → yellow → ...` (more colors require further verification).

### 2.6 tmux Pane ID Assignment (Verified)

tmux pane IDs are **globally incrementing**, not team-scoped:

| Team | Member | paneId |
|------|--------|--------|
| analysis-team | researcher-config | `%14` |
| analysis-team | researcher-tasks | `%15` |
| analysis-team | researcher-comms | `%16` |
| verify-team | tester-01 | `%18` |
| verify-team | tester-02 | `%19` |

Note: `%17` was skipped (likely allocated to a system pane). IDs are managed globally by tmux and are not guaranteed to be contiguous.

---

## 3. Task System

### 3.1 Storage Structure

```
~/.claude/tasks/{team_name}/
├── .lock          # Concurrency lock file (0 bytes)
├── 1.json         # Task #1
├── 2.json         # Task #2
├── 3.json         # Task #3
└── ...
```

### 3.2 Task File Schema

```jsonc
{
  "id": "1",                           // string: auto-incrementing ID
  "subject": "Analyze team config",     // string: task title (imperative mood)
  "description": "Detailed desc...",    // string: task details and acceptance criteria
  "activeForm": "Analyzing team config",// string: display text when in_progress (present continuous)
  "status": "completed",                // string: (pending|in_progress|completed|deleted)
  "owner": "researcher-config",         // string|undefined: assignee name
  "blocks": ["4"],                      // string[]: downstream task IDs blocked by this task
  "blockedBy": [],                      // string[]: upstream task IDs blocking this task
  "metadata": {}                        // object|undefined: additional metadata
}
```

### 3.3 State Machine

```
                    ┌──────────┐
                    │ (created)│
                    └────┬─────┘
                         │ TaskCreate
                         ▼
                    ┌──────────┐
             ┌──────│  pending │──────┐
             │      └────┬─────┘      │
             │           │ TaskUpdate  │ TaskUpdate
             │           │ status=     │ status=
             │           │ in_progress │ deleted
             │           ▼             ▼
             │      ┌──────────┐  ┌─────────┐
             │      │in_progress│  │ deleted │
             │      └────┬─────┘  └─────────┘
             │           │ TaskUpdate
             │           │ status=completed
             │           ▼
             │      ┌──────────┐
             └──────│completed │
                    └──────────┘
```

- **pending**: Default status for newly created tasks
- **in_progress**: Marked when an agent starts working
- **completed**: Task finished
- **deleted**: Task removed (permanent)

### 3.4 Dependency Graph (DAG)

Tasks form a directed acyclic graph via `blocks` / `blockedBy`:

```
Example:
  Task #1 ──blocks──→ Task #4
  Task #2 ──blocks──→ Task #4
  Task #3 ──blocks──→ Task #4

  Task #4 ──blockedBy──→ [#1, #2, #3]
```

**Bidirectional linking mechanism:**
- When `TaskUpdate` sets `addBlockedBy: ["1"]`, the system automatically:
  - Adds `"1"` to task #4's `blockedBy`
  - Adds `"4"` to task #1's `blocks`
- This is a **system-maintained bidirectional reference** ensuring consistency

**Blocking rules:**
- Tasks with non-empty `blockedBy` cannot be claimed
- Agents should prefer lower-ID available tasks
- When all `blockedBy` tasks complete, downstream tasks are automatically unblocked

### 3.5 .lock File

- Located at the task directory root
- 0-byte file for filesystem-level concurrency control
- Prevents data races when multiple agents modify task files simultaneously

### 3.6 Task Assignment

| Method | Description |
|--------|-------------|
| `TaskUpdate owner=X` | Team Lead assigns an owner |
| Agent self-claim | Agent sets itself as owner via `TaskUpdate` |
| No owner | Unassigned task, awaiting assignment |

---

## 4. Agent Communication

### 4.1 SendMessage Tool

The core communication tool, supporting 5 message types:

#### 4.1.1 message — Point-to-Point

```jsonc
{
  "type": "message",
  "recipient": "researcher-config",    // recipient name (not agentId)
  "content": "Please start analysis...",
  "summary": "Assign config analysis"  // 5-10 word summary (UI preview)
}
```

- **Unicast**: One-to-one communication
- **Use cases**: Task instructions, progress reports, result delivery

#### 4.1.2 broadcast — Broadcast

```jsonc
{
  "type": "broadcast",
  "content": "Stop all work, blocking issue found",
  "summary": "Critical blocking issue"
}
```

- **Multicast**: Sent to all team members
- **Cost**: N teammates = N independent message deliveries
- **Use sparingly**: Only for urgent team-wide notifications

#### 4.1.3 shutdown_request — Shutdown Request

```jsonc
{
  "type": "shutdown_request",
  "recipient": "researcher-config",
  "content": "Tasks complete, please shut down"
}
```

- **Protocol-based**: Requires response (approve/reject)
- **Graceful shutdown**: Gives agent a chance to save state

#### 4.1.4 shutdown_response — Shutdown Response

```jsonc
// Approve
{
  "type": "shutdown_response",
  "request_id": "shutdown-1770536808909@tester-01",
  "approve": true
}

// Reject
{
  "type": "shutdown_response",
  "request_id": "shutdown-1770536808909@tester-01",
  "approve": false,
  "content": "Still working on task #3, need 5 more minutes"
}
```

#### 4.1.5 plan_approval_response — Plan Approval

```jsonc
{
  "type": "plan_approval_response",
  "request_id": "abc-123",
  "recipient": "researcher-config",
  "approve": true              // or false + content with reason
}
```

- **Prerequisite**: Member's `planModeRequired: true`
- **Flow**: Member calls `ExitPlanMode` → Lead receives approval request → Approve/Reject

### 4.2 SendMessage Response Format (Verified)

```jsonc
// message type response
{
  "success": true,
  "message": "Message sent to tester-01's inbox",
  "routing": {
    "sender": "team-lead",
    "target": "@tester-01",
    "targetColor": "blue",               // recipient's color identifier
    "summary": "Wake up and assign task",
    "content": "You've been woken up. Please execute..."
  }
}

// shutdown_request type response
{
  "success": true,
  "message": "Shutdown request sent to tester-01. Request ID: shutdown-1770536808909@tester-01",
  "request_id": "shutdown-1770536808909@tester-01",  // format: shutdown-{timestamp}@{agent_name}
  "target": "tester-01"
}
```

### 4.3 Message Delivery Mechanism

```
Sender Agent                    System                      Receiver Agent
    │                            │                              │
    ├── SendMessage ───────────→│                              │
    │                            ├── Check receiver state ───→│
    │                            │                              │
    │                            │ [Receiver idle]              │
    │                            ├── Deliver immediately ────→│ (wake up)
    │                            │                              │
    │                            │ [Receiver busy]              │
    │                            ├── Queue message              │
    │                            │   (wait for turn end)        │
    │                            ├── Turn ends ────────────→│ (deliver queued)
    │                            │                              │
```

**Key characteristics:**

1. **Auto-delivery**: Receivers don't need to check their inbox; messages are injected as new conversation turns
2. **Queue buffering**: If receiver is mid-turn, messages queue and deliver when the turn ends
3. **Idle notifications**: Agents auto-enter idle after each turn; system sends idle notification to Lead
4. **Peer DM visibility**: DM summaries between teammates are included in idle notifications, visible to Lead
5. **Inbox mechanism**: Messages are delivered to the receiver's inbox (`"Message sent to tester-01's inbox"`), system handles wake-up

### 4.4 Idle Notification Format (Verified)

```jsonc
// Basic idle notification
{
  "type": "idle_notification",
  "from": "tester-01",
  "timestamp": "2026-02-08T07:38:20.153Z",  // ISO 8601 format
  "idleReason": "available"                   // only "available" observed so far
}

// Idle notification with Peer DM summary
{
  "type": "idle_notification",
  "from": "tester-02",
  "timestamp": "2026-02-08T07:46:31.579Z",
  "idleReason": "available",
  "summary": "[to tester-01] P2P test message sent to tester-01"  // optional, present during peer DMs
}
```

**Key observations:**
- `summary` field only appears during peer-to-peer communication, format: `[to {recipient}] {summary content}`
- An agent may send multiple consecutive idle notifications (e.g. 2 in quick succession after a turn ends) — this is normal behavior
- `idleReason` has only been observed as `"available"`, other values may exist but were not triggered

### 4.5 Communication Topology

```
                    ┌───────────┐
                    │ Team Lead │
                    └─┬───┬───┬─┘
                      │   │   │
           message    │   │   │   message
              ┌───────┘   │   └───────┐
              ▼           ▼           ▼
        ┌─────────┐ ┌─────────┐ ┌─────────┐
        │Agent-A  │ │Agent-B  │ │Agent-C  │
        └─────────┘ └─────────┘ └─────────┘
              ▲                       │
              └───── message ─────────┘
                  (peer-to-peer DM)
```

- **Star topology primary**: Lead communicates with each member
- **P2P supported**: Teammates can DM each other directly
- **No central message bus**: Messages are routed directly by the system, not stored intermediately

---

## 5. Agent Lifecycle

```
┌──────────┐     Task(team_name=X)     ┌──────────┐
│  Not exist│ ────────────────────────→ │ Creating  │
└──────────┘                           └────┬─────┘
                                            │ Register to config.json
                                            │ Allocate tmux pane
                                            ▼
                                       ┌──────────┐
                                ┌──────│  Active   │◄─────┐
                                │      └────┬─────┘      │
                                │           │ Turn ends   │ Receives message
                                │           ▼             │
                                │      ┌──────────┐      │
                                │      │   Idle   │──────┘
                                │      └────┬─────┘
                                │           │ shutdown_request
                                │           ▼
                                │      ┌──────────┐
                                │      │ Confirm  │
                                │      │ Shutdown │
                                │      └────┬─────┘
                                │           │ approve=true
                                │           ▼
                                │      ┌──────────┐
                                └──────│Terminated│
                                       └──────────┘
```

### 5.1 Creation Phase

1. Lead calls `Task` tool with `team_name` and `name`
2. System creates a new tmux pane (assigns paneId e.g. `%14`)
3. Launches an independent Claude process in the new pane
4. Writes member info to `config.json`'s `members` array
5. Agent's `prompt` field stores the initial instructions

### 5.2 Active Phase

- Agent executes tasks independently with full toolset (depending on `agentType`)
- Can read/write files, execute commands, call APIs
- Updates task status via `TaskUpdate`
- Communicates with Lead and teammates via `SendMessage`

### 5.3 Idle Phase

- Automatically enters idle after each API round-trip (turn)
- System sends idle notification to Lead
- **Idle ≠ Terminated**: Idle agents can receive messages and be woken up
- Normal workflow: Execute → Send message → Idle → Woken up → Continue

### 5.4 Shutdown Phase

1. Lead sends `shutdown_request`
2. Agent receives request and can:
   - `approve: true` → Process terminates
   - `approve: false` → Continue working (with rejection reason)
3. After all agents shut down, Lead can call `TeamDelete` to clean up

**Complete shutdown message chain (verified):**

```
Lead                        System                        Agent
  │                          │                              │
  ├─ shutdown_request ─────→│──→ Deliver to Agent ────────→│
  │                          │                              │
  │                          │       Agent calls             │
  │                          │    shutdown_response          │
  │                          │      (approve=true)           │
  │                          │                              │
  │◄── shutdown_approved ───│◄── Process terminating ─────│
  │    (includes paneId,     │                              │
  │     backendType)         │                              │
  │                          │                              │
  │◄── teammate_terminated ─│◄── Process terminated ──────│
  │    (system message)      │                              │
```

Actual notifications received by Lead (verified):

```jsonc
// 1. Agent approves shutdown
{
  "type": "shutdown_approved",
  "requestId": "shutdown-1770536808909@tester-01",
  "from": "tester-01",
  "timestamp": "2026-02-08T07:46:52.929Z",
  "paneId": "%18",           // tmux pane ID
  "backendType": "tmux"      // runtime backend
}

// 2. System confirms termination (from teammate_id="system")
{
  "type": "teammate_terminated",
  "message": "tester-01 has shut down."
}
```

**Note**: `teammate_terminated` and `shutdown_approved` arrive nearly simultaneously; order may vary (in testing, `teammate_terminated` arrived slightly before `shutdown_approved`).

---

## 6. Storage Architecture

### 6.1 `~/.claude/` Directory Overview

```
~/.claude/
├── CLAUDE.md                     # User global instructions
├── settings.json                 # Claude Code settings
├── history.jsonl                 # Session history (JSONL format)
├── stats-cache.json              # Statistics cache
├── statusline-command.sh         # Status bar script
│
├── teams/                        # Team configurations
│   └── {team-name}/
│       └── config.json           # Team metadata + member list
│
├── tasks/                        # Task data
│   └── {team-name}/              # Isolated per team
│       ├── .lock                 # Concurrency lock
│       └── {id}.json             # Individual task file
│
├── session-env/                  # Session environments
│   └── {session-uuid}/           # One directory per session
│
├── debug/                        # Debug logs
│   ├── {session-uuid}.txt        # One log file per session
│   └── latest -> ...             # Symlink to latest log
│
├── shell-snapshots/              # Shell environment snapshots
│   └── snapshot-zsh-{ts}-{id}.sh # zsh environment snapshot script
│
├── agents/                       # Custom agent definitions
├── commands/                     # Custom commands
├── hooks/                        # Hook configurations
├── skills/                       # Custom skills
├── plugins/                      # Plugins
├── plans/                        # Plan files
├── todos/                        # Todo lists
├── projects/                     # Project configurations
├── output-styles/                # Output styles
├── paste-cache/                  # Clipboard cache
├── cache/                        # General cache
├── file-history/                 # File modification history
├── statsig/                      # Feature flags
└── telemetry/                    # Telemetry data
```

### 6.2 Data Flow

```
TeamCreate
    │
    ├── Write ~/.claude/teams/{name}/config.json (initial lead member)
    └── Create ~/.claude/tasks/{name}/ directory + .lock file

TaskCreate
    │
    └── Write ~/.claude/tasks/{name}/{id}.json

Task(team_name=X, name=Y)  [spawn agent]
    │
    ├── Update ~/.claude/teams/{name}/config.json (add new member)
    ├── Create tmux pane
    ├── Create ~/.claude/session-env/{session-uuid}/
    ├── Write ~/.claude/shell-snapshots/snapshot-*.sh
    └── Write ~/.claude/debug/{session-uuid}.txt

SendMessage
    │
    └── In-memory routing (message content NOT persisted to filesystem)

TeamDelete
    │
    ├── Delete ~/.claude/teams/{name}/
    └── Delete ~/.claude/tasks/{name}/
```

### 6.3 Key Findings

1. **Messages don't persist to disk**: `SendMessage` routes messages through in-memory channels, not the filesystem. Message history is unrecoverable after session ends (except indirect traces in `debug/*.txt` logs).

2. **Task = File**: Each task is an independent JSON file with simple concurrency control via `.lock`. A lightweight "database" design.

3. **tmux as process manager**: Agent processes run in tmux panes, leveraging tmux's process isolation and terminal multiplexing.

4. **Shell snapshots**: Each agent spawn saves a shell environment snapshot (`shell-snapshots/`), ensuring new agents inherit correct environment variables.

---

## 7. Live Examples

### 7.1 Analysis Team Timeline

| Timestamp (ms) | Event | Files Affected |
|-----------------|-------|----------------|
| 1770535107409 | TeamCreate "analysis-team" | `teams/analysis-team/config.json` created |
| 1770535107409 | Team Lead registered | config.json members[0] |
| — | TaskCreate #1~#4 | `tasks/analysis-team/1~4.json` |
| — | TaskUpdate #4 blockedBy=[1,2,3] | 4.json + 1/2/3.json bidirectional update |
| 1770535144590 | Spawn researcher-config | config.json members[1], tmux %14 |
| 1770535149817 | Spawn researcher-tasks | config.json members[2], tmux %15 |
| 1770535156897 | Spawn researcher-comms | config.json members[3], tmux %16 |
| — | TaskUpdate #1~#3 owner + in_progress | Task json files updated |
| — | researcher-config done → SendMessage | 1.json status=completed |
| — | researcher-tasks done → SendMessage | 2.json status=completed |
| — | researcher-comms done → SendMessage | 3.json status=completed |
| — | Task #4 unblocked | 4.json blockedBy cleared |
| — | Team Lead writes report | This document |

### 7.2 Config Evolution

**At creation (lead only):**
```json
"members": [
  { "name": "team-lead", "agentType": "team-lead", "tmuxPaneId": "" }
]
```

**After spawning 3 agents:**
```json
"members": [
  { "name": "team-lead",       "agentType": "team-lead",       "tmuxPaneId": "" },
  { "name": "researcher-config","agentType": "general-purpose", "tmuxPaneId": "%14", "color": "blue" },
  { "name": "researcher-tasks", "agentType": "general-purpose", "tmuxPaneId": "%15", "color": "green" },
  { "name": "researcher-comms", "agentType": "general-purpose", "tmuxPaneId": "%16", "color": "yellow" }
]
```

### 7.3 Verification Team (verify-team) Test Record

A separate `verify-team` was created to validate report accuracy:

**Configuration**: 1 Lead + 2 Testers (tester-01 blue, tester-02 green)

| # | Test Item | Operation | Result |
|---|-----------|-----------|--------|
| 1 | TeamCreate | Create verify-team | config.json generated correctly with lead member |
| 2 | Agent spawn | Spawn tester-01 | Registered in config.json, assigned tmux %18, color blue |
| 3 | Agent self-check | tester-01 reads config.json | Confirmed self-registration, all fields present |
| 4 | Idle notification | tester-01 turn ends | Lead received `idle_notification(idleReason=available)` |
| 5 | Wake idle agent | Lead messages idle tester-01 | tester-01 woken up, executed tasks |
| 6 | Task CRUD | tester-01 runs Create→Update→List→Complete | All four operations successful |
| 7 | Consecutive idle | tester-01 multiple idle | 2 consecutive idle_notifications, normal behavior |
| 8 | P2P communication | tester-02 → tester-01 DM | Direct communication successful, no Lead relay |
| 9 | P2P wake-up | tester-02's message wakes idle tester-01 | tester-01 auto-woken and replied |
| 10 | Peer DM visibility | Lead observes idle notifications | Summary format `[to tester-01] P2P test message...` |
| 11 | Shutdown protocol | Lead sends 2 shutdown_requests | Both agents approved and terminated |
| 12 | Termination notices | System messages | Received `shutdown_approved` + `teammate_terminated` |
| 13 | TeamDelete | Cleanup verify-team | teams/ and tasks/ directories deleted |

**P2P communication chain:**
```
tester-02 ──DM──→ tester-01 (idle → woken up)
     idle notification → Lead summary: [to tester-01] P2P test message sent to tester-01

tester-01 ──DM──→ tester-02 (reply confirmation)
     idle notification → Lead summary: [to tester-02] Confirmed P2P test message received

tester-02 ──SendMessage──→ team-lead (summary report)
```

**Conclusion**: All 11 core mechanisms described in the analysis report are consistent with actual behavior.

---

## 8. Prompt Engineering

### 8.1 Agent Prompt Structure

When Team Lead spawns an agent via the `Task` tool, the `prompt` parameter becomes the agent's **initial system instruction**, stored in full in `config.json`'s `prompt` field.

**Design principles:**

1. **Role definition**: Explicitly state team and name
   ```
   "You are a member of analysis-team named researcher-config."
   ```

2. **Task description**: Detailed step-by-step instructions
   ```
   "Please execute the following:
    1. Read ~/.claude/teams/...
    2. Search ~/.claude/teams/ for all..."
   ```

3. **Output directive**: Specify result delivery method
   ```
   "When done, send results to team-lead via SendMessage."
   ```

### 8.2 System-Injected Context

Beyond the user-provided `prompt`, the system automatically injects:

| Context | Source | Description |
|---------|--------|-------------|
| CLAUDE.md | Project root | Project-level instructions |
| ~/.claude/CLAUDE.md | User home | User-level global instructions |
| Tool definitions | System | Available tools and parameters |
| Team context | config.json | Team name, member list |
| Environment | shell-snapshots | Shell environment variables |

### 8.3 Agent Types and Tool Permissions

| agentType | Available Tools | Use Case |
|-----------|----------------|----------|
| `general-purpose` | All tools (Read, Write, Edit, Bash, Task...) | Coding, analysis, general tasks |
| `Explore` | Read-only (Read, Grep, Glob, WebFetch...) | Code search, information gathering |
| `Plan` | Read-only | Architecture design, planning |
| `Bash` | Bash commands only | System operations, script execution |

---

## Appendix A: Comparison with Microservice Architecture

| Claude Code Team System | Microservice Architecture |
|------------------------|--------------------------|
| Team Lead | API Gateway / Orchestrator |
| Agent (tmux pane) | Service Instance (Container) |
| config.json | Service Registry |
| tasks/*.json | Task Queue (simplified) |
| SendMessage | Message Broker (in-memory) |
| .lock file | Distributed Lock |
| shell-snapshots | Container Image |
| TeamDelete | Service Mesh Teardown |

## Appendix B: Best Practices

1. **Team size**: 3-5 agents is optimal; more increases communication overhead
2. **Task granularity**: Each task should be completable within a single agent's context window
3. **Dependency management**: Use `blocks/blockedBy` to build DAGs; avoid circular dependencies
4. **Message discipline**: Prefer `message` over `broadcast` to reduce token consumption
5. **Graceful shutdown**: Always use `shutdown_request/response` protocol to close agents
6. **Resource cleanup**: Call `TeamDelete` after work is complete to release resources
