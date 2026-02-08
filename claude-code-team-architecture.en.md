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
6. [tmux Internals Deep Dive](#6-tmux-internals-deep-dive)
7. [Storage Architecture](#7-storage-architecture)
8. [Live Examples](#8-live-examples)
9. [Prompt Engineering](#9-prompt-engineering)

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
     │  ├── teams/{name}/inboxes/*.json│
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
| Message inboxes | `~/.claude/teams/{name}/inboxes/{agentName}.json` | Agent message persistence (with read status) |
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
| `tmuxPaneId` | string | — | tmux pane ID (e.g. `%14`), empty for lead, `"in-process"` for in-process backend |
| `cwd` | string | — | Agent's working directory |
| `subscriptions` | array | — | Message subscription list (currently always empty) |
| `prompt` | string | ✓ | Initial task instructions (passed via Task tool) |
| `color` | string | ✓ | UI color identifier, assigned in join order (see 2.5) |
| `planModeRequired` | boolean | ✓ | Whether plan approval is required before execution |
| `backendType` | string | ✓ | Runtime backend: `tmux` (separate tmux pane process) or `in-process` (same process as Lead) |
| `isActive` | boolean | ✓ | Whether the agent is currently active |

### 2.4 Team Naming

- **Named teams**: `TeamCreate` with explicit `team_name`, e.g. `analysis-team`
- **Anonymous teams**: UUID-named teams (e.g. `e132a923-...`) are auto-generated

### 2.5 Color Assignment Rules (Verified)

Member colors are assigned automatically by registration order. Team Lead has no color. Complete **8-color cycle**:

| Index (mod 8) | Color | Verified (tmux-verify) |
|---------------|-------|----------------------|
| 0 | `blue` | prober-01 (1st registered) |
| 1 | `green` | color-02 (2nd registered) |
| 2 | `yellow` | color-03 (3rd registered) |
| 3 | `purple` | color-04 (8th registered, delayed start) |
| 4 | `orange` | color-05 (4th registered) |
| 5 | `pink` | color-06 (5th registered) |
| 6 | `cyan` | color-07 (6th registered) |
| 7 | `red` | color-08 (7th registered) |

The 9th member color-09 was assigned `blue` (index 8 mod 8 = 0), confirming the cycle wraps around.

**Color assignment formula**: `color = COLORS[memberIndex % 8]`

**Note: config.json concurrent write race**: When multiple agents register simultaneously, lost updates may occur (e.g., color-04's entry was overwritten). However, the agent itself still holds the correct color assignment (passed via CLI parameter `--agent-color`).

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
    │                            │                              │
    │                            ├── 1. Write to receiver's     │
    │                            │   inbox file                 │
    │                            │   (inboxes/{name}.json,      │
    │                            │    read=false)                │
    │                            │                              │
    │                            ├── 2. Check receiver state ──→│
    │                            │                              │
    │                            │ [Receiver idle]              │
    │                            ├── Notify/wake up ──────────→│ (reads inbox)
    │                            │                              │
    │                            │ [Receiver busy]              │
    │                            ├── (wait for turn end)        │
    │                            ├── Turn ends ───────────────→│ (reads inbox)
    │                            │                              │
    │                            │                              ├── Consume message
    │                            │                              │   (read→true)
    │                            │                              │
```

**Key characteristics:**

1. **Inbox file persistence**: Each message is written to the receiver's inbox file `~/.claude/teams/{team}/inboxes/{name}.json` (JSON array), new messages have `read: false`, updated to `read: true` after consumption
2. **Auto-delivery**: Receivers don't need to check their inbox; messages are injected as new conversation turns
3. **Queue buffering**: If receiver is mid-turn, messages wait in the inbox file and deliver when the turn ends
4. **Idle notifications**: Agents auto-enter idle after each turn; system sends idle notification to Lead (also written to Lead's inbox file)
5. **Peer DM visibility**: DM summaries between teammates are included in idle notifications, visible to Lead
6. **Message retention**: Consumed messages are NOT removed from the inbox file; full history is preserved until TeamDelete

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

### 4.5 Inbox File Schema (Verified)

**Path**: `~/.claude/teams/{team_name}/inboxes/{agentName}.json`

**Creation timing**:
- An agent's inbox file is created on first message receipt (e.g. `task_assignment` triggered by TaskUpdate assigning owner)
- Team Lead's inbox file is created on first message from an agent

**File format**: JSON array, each message is an object

```jsonc
[
  {
    "from": "team-lead",                  // string: sender name
    "text": "Message body or JSON string", // string: message content (see type table below)
    "timestamp": "2026-02-08T11:36:55Z",  // string: ISO 8601 timestamp
    "read": true,                          // boolean: whether consumed by receiver
    "summary": "Summary text",             // string (optional): SendMessage summary parameter
    "color": "blue"                        // string (optional): sender agent's color identifier
  }
]
```

**Message types in the `text` field**:

| Message Type | `text` Format | Trigger Source |
|-------------|---------------|----------------|
| Regular message | Plain text | `SendMessage(type="message")` |
| task_assignment | JSON: `{"type":"task_assignment","taskId":"1",...}` | `TaskUpdate(owner=X)` |
| idle_notification | JSON: `{"type":"idle_notification","idleReason":"available",...}` | Agent turn ends (automatic) |
| shutdown_request | JSON: `{"type":"shutdown_request","requestId":"...",...}` | `SendMessage(type="shutdown_request")` |
| shutdown_approved | JSON: `{"type":"shutdown_approved","paneId":"...",...}` | Agent approves shutdown |

**`read` field behavior**:
- Defaults to `false` on write
- Updated to `true` after receiver process consumes the message (**written back to the same file**)
- Consumed messages are **never deleted**; inbox file retains full message history

**`color` field**:
- Only present on messages sent by **non-Lead** agents
- Value is the sender agent's color identifier from config.json

**Lifecycle**:
- On TeamDelete, the `inboxes/` directory is deleted along with `teams/{name}/`

### 4.6 Communication Topology

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
- **File-based message storage**: All messages persist to `inboxes/{agentName}.json`; the system handles routing and wake-up

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

## 6. tmux Internals Deep Dive

> This chapter was produced by creating a `tmux-probe` team and capturing live tmux state, process trees, and startup commands to fully trace agent behavior at the tmux layer.

### 6.1 tmux Session Layout

The team system runs inside the user's existing tmux session (in this case `dev`):

```
tmux session: "dev"
├── window 0
│   ├── pane %0  (ttys003) ← Team Lead: claude --dangerously-skip-permissions
│   │                          PID 58657, main session process
│   ├── pane %17 (ttys030) ← System helper pane (zsh)
│   │
│   └── pane %20 (ttys036) ← [dynamically created on spawn] Agent: probe-01
│                              PID 9507, destroyed on TeamDelete
└── window 1
    └── pane %3  (ttys022) ← User terminal (zsh)
```

**Key**: Agent panes are **dynamically created and destroyed** — they do not persist after team cleanup.

### 6.2 tmux Operations During Agent Spawn

Pane state comparison before and after spawn:

```
Before spawn (3 panes):
  %0  (80x55)  │ %17 (186x55)
               │
────────────────────────────────

After spawn (4 panes):
  %0  (80x55)  │ %17 (186x27)    ← top half (compressed)
               ├──────────────
               │ %20 (186x27)    ← new agent pane (bottom half)
────────────────────────────────
```

Detailed execution sequence:

```
Lead calls Task(team_name=X, name=Y)
    │
    ├── 1. tmux split-window — vertically splits an existing pane
    │      → %17 shrinks from 186x55 to 186x27 (top)
    │      → new pane %20 = 186x27 (bottom)
    │
    ├── 2. Launch zsh in the new pane
    │      PID 9424, PPID=41159 (tmux server)
    │
    ├── 3. Save shell environment snapshot
    │      ~/.claude/shell-snapshots/snapshot-zsh-{timestamp}-{id}.sh
    │
    ├── 4. Execute claude command with full agent parameters
    │      → claude process PID 9507, PPID=9424 (zsh)
    │
    └── 5. Register in config.json: tmuxPaneId="%20"
```

### 6.3 Agent Startup Command (Actual Capture)

```bash
# Team Lead (main session, no agent parameters)
claude --dangerously-skip-permissions

# Agent (full parameter chain, each with a specific purpose)
/Users/night/.local/share/claude/versions/2.1.34 \
  --agent-id probe-01@tmux-probe \              # globally unique ID for message routing
  --agent-name probe-01 \                        # human-readable name for SendMessage recipient
  --team-name tmux-probe \                       # team name for locating config.json and tasks/
  --agent-color blue \                           # UI color identifier
  --parent-session-id c93b690c-...-d9451b749ac2 \# Lead's session ID for message return path
  --agent-type general-purpose \                 # agent type, determines available toolset
  --dangerously-skip-permissions \               # permission mode (inherited from Lead)
  --model claude-opus-4-6                        # LLM model
```

**Core finding**: An agent is just another `claude` CLI process distinguished by `--agent-*` parameters. Lead and agents run the **same binary** (`/Users/night/.local/share/claude/versions/2.1.34`).

**Complete native Agent CLI parameters (second verification)**:

| Parameter | Purpose | Always Present |
|-----------|---------|---------------|
| `--agent-id` | Globally unique ID for message routing | Yes |
| `--agent-name` | Human-readable name | Yes |
| `--team-name` | Team affiliation | Yes |
| `--agent-color` | UI color identifier | Yes |
| `--parent-session-id` | Lead's session ID | Yes |
| `--agent-type` | Agent type | Yes |
| `--model` | LLM model | Yes |
| `--dangerously-skip-permissions` | Bypass permission checks | **Conditional**: only when Lead uses this mode |
| `--permission-mode` | Permission mode (6 choices) | **Conditional**: passed when Lead/caller specifies |
| `--allowedTools` | Tool whitelist | **Conditional**: passed when tool restrictions are configured |
| `--disallowedTools` | Tool blacklist | **Conditional**: passed when tool restrictions are configured |

> Note: The last four conditional parameters are native Claude CLI features (confirmed via `claude --help`). Whether they are passed during Team spawn depends on the Lead's configuration. The `--agent-*` parameters are Team system internals not shown in `claude --help`.

### 6.4 Process Tree (Actual Capture)

```
tmux server (PID 41159)
│
├── zsh (PID 41160, pane %0, /dev/ttys003)
│   └── claude --dangerously-skip-permissions (PID 58657)  ← Team Lead
│       ├── caffeinate -i -t 300                           ← prevent system sleep
│       ├── alpaca-mcp-server serve                        ← MCP service process
│       └── pyright-langserver --stdio                     ← language service
│
├── zsh (PID 9424, pane %20, /dev/ttys036)                 ← [dynamically created]
│   └── claude --agent-id probe-01@... (PID 9507)          ← Agent process
│       └── /bin/zsh -c source snapshot-*.sh && eval '...' ← Bash tool subprocess
│
├── zsh (PID 70762, pane %17, /dev/ttys030)                ← helper pane
└── zsh (PID 62613, pane %3, /dev/ttys022)                 ← user terminal
```

PPID chain: `Bash subprocess → claude agent → zsh → tmux server`

### 6.5 TTY and stdin Handling

| Process | TTY | stdin | Notes |
|---------|-----|-------|-------|
| Team Lead (58657) | `/dev/ttys003` | Real terminal | Receives user keyboard input |
| Agent (9507) | `/dev/ttys036` | tmux pseudo-terminal | Has TTY but no human interaction |
| Agent's Bash subprocess | `??` | `/dev/null` | **No controlling terminal** |

**Important implications:**
- Agent's Bash tool forks a new zsh subprocess per invocation
- Subprocess first `source`s the shell snapshot to restore environment, then executes the command
- stdin comes from `/dev/null`, so **interactive commands are impossible** (`git rebase -i`, `vim`, `less`)
- Environment variable `TMUX_PANE=%20` is accessible inside the agent

### 6.6 Pane Lifecycle to tmux Operation Mapping

| Lifecycle Event | tmux Operation | Process Impact |
|----------------|----------------|----------------|
| Team Lead starts | User runs `claude` in existing pane | No tmux operation |
| Agent spawn | `tmux split-window` → new pane | New zsh → new claude process |
| Agent active | claude runs in pane; observable via `tmux capture-pane` | Process running normally |
| Agent idle | No tmux operation | Process alive, just not making API calls |
| Agent shutdown (approve) | claude exits → zsh exits | Pane may remain but empty |
| TeamDelete | `tmux kill-pane` destroys all agent panes | Entire process tree cleaned up |

### 6.7 Why tmux as Process Manager

| Requirement | tmux Solution | Alternative Comparison |
|-------------|---------------|----------------------|
| Process isolation | Independent terminal + process tree per pane | Docker too heavy; subprocess lacks terminal |
| Environment inheritance | shell-snapshots → source to restore | Requires extra env passing mechanism |
| No network needed | No socket/port between panes | HTTP/gRPC requires port management |
| Observability | `tmux capture-pane -t %20 -p` shows live output | subprocess opaque to user |
| Resource cleanup | `tmux kill-pane` single-command cleanup | Must manually kill process tree |
| Cross-platform | Works on macOS / Linux | Windows unsupported (needs WSL) |
| User visibility | User can switch to agent pane and watch live | subprocess invisible to user |

### 6.8 tmux Limitations

1. **Pane count**: Too many panes in one window makes each very small, affecting output display
2. **No Windows support**: tmux is Unix-only; Windows users need WSL
3. **Split direction**: Current vertical splits (horizontal stacking) progressively compress pane height with more agents
4. **Pane IDs never recycled**: Destroyed pane IDs (e.g. `%20`) are never reused; IDs grow indefinitely over long sessions

---

## 7. Storage Architecture

### 7.1 `~/.claude/` Directory Overview

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
│       ├── config.json           # Team metadata + member list
│       └── inboxes/              # Message inboxes (verified)
│           ├── team-lead.json    # Lead's message log
│           └── {agentName}.json  # Each agent's message log
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

### 7.2 Data Flow

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
    ├── Write to receiver's inbox file (inboxes/{recipient}.json, read=false)
    ├── Write to sender's JSONL (tool_use record)
    ├── Notify receiver process (tmux backend: anonymous UDS socketpair; in-process backend: in-memory)
    └── Receiver updates inbox file after consumption (read=true)
    Note: "inbox" is an actual filesystem path, not an in-memory queue

TeamDelete
    │
    ├── Delete ~/.claude/teams/{name}/ (including config.json and inboxes/ directory)
    └── Delete ~/.claude/tasks/{name}/
```

### 7.3 Key Findings

1. **Message delivery mechanism (second correction, verified via inbox-verify + delivery-probe teams)**:

   The previous report's claim "inbox is an in-process memory queue" was **incorrect**. The inbox is an actual JSON file on the filesystem:

   ```
   Lead calls SendMessage(recipient="watcher", content="...")
       │
       ├── 1. Write to receiver's inbox file ← primary persistence path
       │      ~/.claude/teams/{team}/inboxes/watcher.json
       │      (append message object to JSON array, read=false)
       │
       ├── 2. Write to sender's JSONL (tool_use record) ← conversation history
       │      ~/.claude/projects/{project}/{session-id}.jsonl
       │
       ├── 3. Notify receiver process ← wake-up mechanism (varies by backend)
       │      tmux backend: anonymous UDS socketpair notification
       │      in-process backend: direct in-memory delivery
       │
       └── 4. Receiver updates inbox file after consumption ← read status writeback
              (read: false → true, message not deleted)
   ```

   **Inbox files are real**: `"Message sent to watcher's inbox"` refers to `~/.claude/teams/{team}/inboxes/{name}.json`. Every agent (including Team Lead) has an independent inbox file.

   **Why the previous report missed inbox files**:
   - Previous report searched with `find ~/.claude/ -name '*inbox*'` — while `*inbox*` should match `inboxes`
   - The search was executed **after TeamDelete**, which deletes the `inboxes/` directory along with the team
   - This led to the incorrect "no inbox files exist" conclusion

   **Second verification evidence (inbox-verify team)**:

   | Check | Result |
   |-------|--------|
   | After agent spawn | `inboxes/listener.json` appeared immediately (with task_assignment message) |
   | After SendMessage | Message appended to `inboxes/listener.json`, `read: false` |
   | After agent consumption | `read` field updated to `true` in the same file |
   | After agent messages Lead | `inboxes/team-lead.json` created |
   | idle_notification | Written as JSON string to Lead's inbox file |
   | shutdown_request/approved | Both written to corresponding inbox files |
   | After TeamDelete | `inboxes/` directory deleted along with `teams/{name}/` |

   **Delivery mechanism varies by backend (delivery-probe team verification)**:

   | Backend Type | Process Model | Message Persistence | Wake-up Notification |
   |-------------|--------------|--------------------|--------------------|
   | `tmux` | Separate processes (different tmux panes) | Write to inbox file | Anonymous UDS socketpair |
   | `in-process` | Same process | Write to inbox file | Direct in-memory delivery |

   Both backends **use inbox files** as the message persistence layer. The only difference is the inter-process wake-up notification mechanism.

   **Wake-up mechanism deep verification (lsof inspection)**:

   The tmux-backend Agent process was observed to have:
   - **4 KQUEUE file descriptors** (macOS kernel event notification)
   - **DIR handles** on `~/.claude/`, `~/.claude/commands`, `~/.claude/skills`, project `.claude/` — for general filesystem change monitoring (command/skill hot-reloading, etc.)
   - **Anonymous UDS socketpairs** — inter-process notification channel between Lead and Agent (no filesystem path, not named sockets)
   - **No** `.sock` / `.socket` named socket files

   The Lead process additionally holds DIR handles on `~/.claude/tasks/{team}/` directories (Agents do not), used for monitoring Task file changes.

   ```
   Complete message delivery flow (tmux backend):
   1. Sender calls SendMessage
   2. System writes to receiver's inbox file (JSON append, read=false)
   3. System notifies receiver process via anonymous UDS socketpair
   4. Receiver reads inbox file, consumes message
   5. Receiver updates inbox file (read=true)
   ```

2. **Task = File**: Each task is an independent JSON file with simple concurrency control via `.lock`. A lightweight "database" design.

3. **tmux as process manager**: Agent processes run in tmux panes, leveraging tmux's process isolation and terminal multiplexing.

4. **Shell snapshots**: Each agent spawn saves a shell environment snapshot (`shell-snapshots/`), ensuring new agents inherit correct environment variables.

5. **Conversation persistence (JSONL)**: Each agent (including Lead) has an independent JSONL conversation log at `~/.claude/projects/{project-path}/{session-id}.jsonl`. This file records the complete conversation history including tool calls, tool results, and message exchanges.

6. **Message persistence (Inbox files)**: Each agent has an independent inbox JSON file at `~/.claude/teams/{team}/inboxes/{name}.json`. This file records all messages received by that agent (with read status) and serves as the primary vehicle for message delivery. Inbox files are deleted on TeamDelete.

7. **Agent backend types**: The system supports two agent backends — `tmux` (separate process in a tmux pane) and `in-process` (same process as Lead). The backend type is recorded in config.json's `backendType` field. When the Claude CLI process is not running inside tmux (`$TMUX` not set), the `in-process` backend is used.

8. **Environment variables** (mechanism-verify + tmux-verify team verification):

   | Variable | Value | Lead | tmux Agent | in-process Agent |
   |----------|-------|------|-----------|-----------------|
   | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | `1` | Yes | Yes | Yes |
   | `CLAUDECODE` | `1` | Yes | Yes | Yes |
   | `CLAUDE_CODE_ENTRYPOINT` | `cli` | Yes | Yes | Yes |
   | `TMUX_PANE` | `%XX` | Yes | Yes | No |

   - `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` indicates team mode is enabled (experimental feature flag)
   - `CLAUDECODE=1` identifies the Claude Code runtime environment
   - Agent team information (team name, agent name, etc.) is passed via CLI parameters, not environment variables

9. **Task subagent output path** (tmux-verify team verification):

   Task tool (subagent) intermediate outputs are stored at:
   ```
   /tmp/claude-{uid}/{project-path-escaped}/tasks/{hash}.output
   ```
   Example: `/tmp/claude-501/-Users-night-working-o3cloud-.../tasks/b6b99ac.output`

10. **config.json concurrent write race** (tmux-verify team discovery):

    When multiple agents register simultaneously, config.json is subject to lost updates: multiple processes read the same version of config.json, each appends its member entry, and the last write overwrites earlier ones. In the tmux-verify team, color-04's registration entry was lost this way (though the agent itself still held the correct color assignment via `--agent-color` parameter, and communication was unaffected).

---

## 8. Live Examples

### 8.1 Analysis Team Timeline

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

### 8.2 Config Evolution

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

### 8.3 Verification Team (verify-team) Test Record

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

## 9. Prompt Engineering

### 9.1 Agent Prompt Structure

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

### 9.2 System-Injected Context

Beyond the user-provided `prompt`, the system automatically injects:

| Context | Source | Description |
|---------|--------|-------------|
| CLAUDE.md | Project root | Project-level instructions |
| ~/.claude/CLAUDE.md | User home | User-level global instructions |
| Tool definitions | System | Available tools and parameters |
| Team context | config.json | Team name, member list |
| Environment | shell-snapshots | Shell environment variables |

### 9.3 Agent Types and Tool Permissions

| agentType | Available Tools | Use Case |
|-----------|----------------|----------|
| `general-purpose` | All tools (Read, Write, Edit, Bash, Task...) | Coding, analysis, general tasks |
| `Explore` | Read-only (Read, Grep, Glob, WebFetch...) | Code search, information gathering |
| `Plan` | Read-only | Architecture design, planning |
| `Bash` | Bash commands only | System operations, script execution |

---

### 9.4 Permission Modes (`--permission-mode`)

Claude CLI natively supports 6 permission modes, passed via `--permission-mode` (confirmed via `claude --help`):

| Mode | CLI Parameter | Behavior |
|------|---------------|----------|
| `default` | `--permission-mode default` | Default mode, requires user confirmation for sensitive operations |
| `acceptEdits` | `--permission-mode acceptEdits` | Auto-accept file edit operations |
| `bypassPermissions` | `--dangerously-skip-permissions` | Bypass all permission checks |
| `plan` | `--permission-mode plan` | Read-only analysis mode |
| `dontAsk` | `--permission-mode dontAsk` | Auto-reject uncertain operations |
| `delegate` | `--permission-mode delegate` | Route permission requests to Lead via inbox protocol |

> Note: In `delegate` mode, when an Agent attempts a restricted operation, it sends a `permission_request` message via the inbox. The Lead/Controller approves or rejects via `permission_response`. This mode was not tested in live verification.

### 9.5 Tool Whitelist/Blacklist

Claude CLI supports fine-grained tool access control via the following parameters (confirmed via `claude --help`):

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `--allowedTools` / `--allowed-tools` | Tool whitelist | `--allowedTools "Bash(git:*) Edit Read"` |
| `--disallowedTools` / `--disallowed-tools` | Tool blacklist | `--disallowedTools "Write Bash"` |
| `--tools` | Specify built-in tool list | `--tools "Bash,Edit,Read"` |

These parameters provide more granular tool access control than `agentType` alone.

---

## Appendix A: Comparison with Microservice Architecture

| Claude Code Team System | Microservice Architecture |
|------------------------|--------------------------|
| Team Lead | API Gateway / Orchestrator |
| Agent (tmux pane) | Service Instance (Container) |
| config.json | Service Registry |
| tasks/*.json | Task Queue (simplified) |
| SendMessage + inboxes/ | Message Broker (file-persisted) |
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
