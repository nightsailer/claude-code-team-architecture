# Claude Code 多 Agent 团队协作机制 — 深度分析报告

> 生成日期：2026-02-08
> 分析环境：Claude Code CLI (claude-opus-4-6)
> 分析团队：analysis-team（1 lead + 3 researchers）

---

## 目录

1. [架构总览](#1-架构总览)
2. [团队系统 (Team System)](#2-团队系统)
3. [任务系统 (Task System)](#3-任务系统)
4. [Agent 通信机制 (Communication)](#4-agent-通信机制)
5. [Agent 生命周期](#5-agent-生命周期)
6. [tmux 底层机制深度剖析](#6-tmux-底层机制深度剖析)
7. [底层存储架构](#7-底层存储架构)
8. [实际运行示例](#8-实际运行示例)
9. [Prompt 工程细节](#9-prompt-工程细节)

---

## 1. 架构总览

Claude Code 的多 Agent 团队协作系统是一个基于文件系统的分布式 Agent 协调框架。核心设计理念：

```
┌─────────────────────────────────────────────────┐
│                  Team Lead (主会话)                │
│  - 创建团队 (TeamCreate)                          │
│  - 创建/分配任务 (TaskCreate/TaskUpdate)           │
│  - 生成队员 (Task tool with team_name)            │
│  - 通信协调 (SendMessage)                         │
│  - 关闭队员 (shutdown_request)                    │
│  - 清理团队 (TeamDelete)                          │
├─────────────────────────────────────────────────┤
│  Agent-A (tmux %14)  │  Agent-B (tmux %15)  │ ...│
│  独立 Claude 进程     │  独立 Claude 进程     │    │
│  共享任务列表          │  共享任务列表          │    │
│  通过 SendMessage     │  通过 SendMessage     │    │
│  与 lead 通信         │  与 lead 通信         │    │
└─────────────────────────────────────────────────┘
           │                    │
     ┌─────┴────────────────────┴─────┐
     │     共享文件系统 (~/.claude/)     │
     │  ├── teams/{name}/config.json  │
     │  ├── teams/{name}/inboxes/*.json│
     │  ├── tasks/{name}/*.json       │
     │  ├── session-env/{id}/         │
     │  ├── debug/{id}.txt            │
     │  └── shell-snapshots/          │
     └────────────────────────────────┘
```

**关键特性：**
- **进程隔离**：每个 Agent 运行在独立的 tmux pane 中
- **文件共享**：团队和任务数据通过 `~/.claude/` 文件系统共享
- **消息驱动**：Agent 间通过 SendMessage 工具进行异步通信
- **中心化协调**：Team Lead 负责任务分配和生命周期管理

---

## 2. 团队系统

### 2.1 团队创建

使用 `TeamCreate` 工具创建团队，系统自动生成：

| 资源 | 路径 | 用途 |
|------|------|------|
| 团队配置 | `~/.claude/teams/{name}/config.json` | 团队元数据和成员列表 |
| 消息收件箱 | `~/.claude/teams/{name}/inboxes/{agentName}.json` | Agent 消息落盘（含已读状态） |
| 任务目录 | `~/.claude/tasks/{name}/` | 团队共享任务列表 |
| 任务锁文件 | `~/.claude/tasks/{name}/.lock` | 并发访问控制 |

### 2.2 config.json Schema

```jsonc
{
  // === 顶层字段 ===
  "name": "analysis-team",          // string: 团队名称（唯一标识）
  "description": "团队描述",         // string: 团队用途说明
  "createdAt": 1770535107409,       // number: Unix 时间戳（毫秒）
  "leadAgentId": "team-lead@analysis-team",  // string: Team Lead 的 agentId
  "leadSessionId": "c93b690c-...",  // string: Team Lead 的会话 UUID

  // === 成员列表 ===
  "members": [
    // --- Team Lead (自动创建) ---
    {
      "agentId": "team-lead@analysis-team",  // string: {name}@{team}
      "name": "team-lead",                    // string: 用于通信的标识名
      "agentType": "team-lead",               // string: Agent 类型
      "model": "claude-opus-4-6",             // string: 使用的模型
      "joinedAt": 1770535107409,              // number: 加入时间戳
      "tmuxPaneId": "",                        // string: tmux pane（lead 为空）
      "cwd": "/path/to/project",              // string: 工作目录
      "subscriptions": []                      // array: 消息订阅
    },

    // --- 普通成员 (Task spawn) ---
    {
      "agentId": "researcher-config@analysis-team",
      "name": "researcher-config",             // 用于 SendMessage 的 recipient
      "agentType": "general-purpose",          // 由 Task 的 subagent_type 决定
      "model": "claude-opus-4-6",
      "prompt": "你是 analysis-team 的成员...", // string: Agent 的初始指令
      "color": "blue",                         // string: UI 显示颜色（blue/green/yellow/...）
      "planModeRequired": false,               // boolean: 是否需要计划审批
      "joinedAt": 1770535144590,
      "tmuxPaneId": "%14",                     // string: tmux pane 标识
      "cwd": "/path/to/project",
      "subscriptions": [],
      "backendType": "tmux",                   // string: 运行后端类型
      "isActive": true                         // boolean: 是否活跃
    }
  ]
}
```

### 2.3 字段详解

| 字段 | 类型 | 仅成员 | 说明 |
|------|------|--------|------|
| `agentId` | string | — | 格式: `{name}@{team_name}`，全局唯一标识 |
| `name` | string | — | 人类可读名称，通信和任务分配时使用 |
| `agentType` | string | — | Agent 类型：`team-lead`, `general-purpose`, `Explore`, `Plan` 等 |
| `model` | string | — | 底层 LLM 模型 ID |
| `joinedAt` | number | — | Unix 毫秒时间戳 |
| `tmuxPaneId` | string | — | tmux pane ID（如 `%14`），lead 为空字符串，in-process 后端为 `"in-process"` |
| `cwd` | string | — | Agent 的工作目录 |
| `subscriptions` | array | — | 消息订阅列表（当前均为空） |
| `prompt` | string | ✓ | Agent 的初始任务指令（由 Task 工具传入） |
| `color` | string | ✓ | UI 颜色标识，按加入顺序分配（见 2.5 颜色分配规则） |
| `planModeRequired` | boolean | ✓ | 是否要求先提交计划再执行 |
| `backendType` | string | ✓ | 运行后端：`tmux`（独立 tmux pane 进程）或 `in-process`（与 Lead 同进程） |
| `isActive` | boolean | ✓ | 当前是否处于活跃状态 |

### 2.4 团队命名

- **命名团队**：`TeamCreate` 指定 `team_name`，如 `analysis-team`
- **匿名团队**：历史上也存在以 UUID 命名的团队（如 `e132a923-...`），通常是自动生成的

### 2.5 颜色分配规则（已验证）

成员颜色按 spawn 注册顺序自动分配，Team Lead 无颜色标记。完整的 **8 色循环**：

| 索引（mod 8） | 颜色 | 验证实例（tmux-verify） |
|------------|--------|----------------------|
| 0 | `blue` | prober-01（第 1 个注册） |
| 1 | `green` | color-02（第 2 个注册） |
| 2 | `yellow` | color-03（第 3 个注册） |
| 3 | `purple` | color-04（第 8 个注册，延迟启动） |
| 4 | `orange` | color-05（第 4 个注册） |
| 5 | `pink` | color-06（第 5 个注册） |
| 6 | `cyan` | color-07（第 6 个注册） |
| 7 | `red` | color-08（第 7 个注册） |

第 9 个成员 color-09 分配到 `blue`（索引 8 mod 8 = 0），确认循环回到起点。

**颜色分配公式**：`color = COLORS[memberIndex % 8]`

**注意：config.json 并发写入竞争**：多个 Agent 同时注册时可能出现 lost update（如 color-04 的条目被覆盖丢失），但 Agent 本身仍持有正确的颜色分配（通过 CLI 参数 `--agent-color` 传入）。

### 2.6 tmux pane ID 分配（已验证）

tmux pane ID 是**全局递增**的，不按团队隔离：

| 团队 | 成员 | paneId |
|------|------|--------|
| analysis-team | researcher-config | `%14` |
| analysis-team | researcher-tasks | `%15` |
| analysis-team | researcher-comms | `%16` |
| verify-team | tester-01 | `%18` |
| verify-team | tester-02 | `%19` |

可见 `%17` 被跳过（可能分配给了其他系统 pane），ID 由 tmux 全局管理，不保证连续。

---

## 3. 任务系统

### 3.1 存储结构

```
~/.claude/tasks/{team_name}/
├── .lock          # 并发锁文件（0 字节）
├── 1.json         # 任务 #1
├── 2.json         # 任务 #2
├── 3.json         # 任务 #3
└── ...
```

### 3.2 任务文件 Schema

```jsonc
{
  "id": "1",                           // string: 自增 ID
  "subject": "分析团队配置文件结构",      // string: 任务标题（祈使句式）
  "description": "详细描述...",          // string: 任务详情和验收标准
  "activeForm": "分析团队配置文件结构",   // string: 进行中状态的显示文本（现在进行时）
  "status": "completed",                // string: 状态 (pending|in_progress|completed|deleted)
  "owner": "researcher-config",         // string|undefined: 负责人名称
  "blocks": ["4"],                      // string[]: 本任务阻塞的下游任务 ID
  "blockedBy": [],                      // string[]: 本任务被阻塞的上游任务 ID
  "metadata": {}                        // object|undefined: 附加元数据
}
```

### 3.3 状态机

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

- **pending**: 新创建的任务默认状态
- **in_progress**: Agent 开始工作时标记
- **completed**: 任务完成
- **deleted**: 任务被移除（永久操作）

### 3.4 依赖关系（DAG）

任务之间通过 `blocks` / `blockedBy` 形成有向无环图（DAG）：

```
本次实例:
  Task #1 ──blocks──→ Task #4
  Task #2 ──blocks──→ Task #4
  Task #3 ──blocks──→ Task #4

  Task #4 ──blockedBy──→ [#1, #2, #3]
```

**双向链接机制**：
- `TaskUpdate` 设置 `addBlockedBy: ["1"]` 时，系统自动：
  - 在任务 #4 的 `blockedBy` 中添加 `"1"`
  - 在任务 #1 的 `blocks` 中添加 `"4"`
- 这是**系统自动维护的双向引用**，确保一致性

**阻塞规则**：
- `blockedBy` 非空的任务不能被认领（claim）
- Agent 应优先处理 ID 较小的可用任务
- 当所有 `blockedBy` 任务完成后，下游任务自动解除阻塞

### 3.5 .lock 文件

- 位于任务目录根部
- 0 字节文件，用于文件系统级的并发控制
- 防止多个 Agent 同时修改同一任务文件导致数据竞争

### 3.6 任务分配

| 方式 | 说明 |
|------|------|
| `TaskUpdate owner=X` | Team Lead 指定 owner |
| Agent 自行认领 | Agent 通过 `TaskUpdate` 设置自己为 owner |
| 无 owner | 空闲任务，等待分配 |

---

## 4. Agent 通信机制

### 4.1 SendMessage 工具

Agent 间通信的核心工具，支持 5 种消息类型：

#### 4.1.1 message — 点对点消息

```jsonc
{
  "type": "message",
  "recipient": "researcher-config",    // 收件人名称（非 agentId）
  "content": "请开始分析...",           // 消息内容
  "summary": "分配配置分析任务"         // 5-10 字摘要（UI 预览）
}
```

- **单播**：一对一通信
- **用途**：任务指令、进度汇报、结果交付

#### 4.1.2 broadcast — 广播消息

```jsonc
{
  "type": "broadcast",
  "content": "停止所有工作，发现阻断问题",
  "summary": "紧急阻断通知"
}
```

- **多播**：发送给团队所有成员
- **成本**：N 个队友 = N 次独立消息投递
- **应慎用**：仅用于紧急团队级通知

#### 4.1.3 shutdown_request — 关闭请求

```jsonc
{
  "type": "shutdown_request",
  "recipient": "researcher-config",
  "content": "任务完成，请关闭"
}
```

- **协议性**：需要对方响应（approve/reject）
- **优雅关闭**：给 Agent 机会保存状态

#### 4.1.4 shutdown_response — 关闭响应

```jsonc
// 同意关闭
{
  "type": "shutdown_response",
  "request_id": "abc-123",     // 从 shutdown_request 消息中提取
  "approve": true
}

// 拒绝关闭
{
  "type": "shutdown_response",
  "request_id": "abc-123",
  "approve": false,
  "content": "还在处理任务 #3，需要 5 分钟"
}
```

#### 4.1.5 plan_approval_response — 计划审批

```jsonc
{
  "type": "plan_approval_response",
  "request_id": "abc-123",
  "recipient": "researcher-config",
  "approve": true              // 或 false + content 说明原因
}
```

- **前提**：队员的 `planModeRequired: true`
- **流程**：队员调用 `ExitPlanMode` → Lead 收到审批请求 → 批准/拒绝

### 4.2 SendMessage 响应格式

调用 `SendMessage` 后，系统返回包含路由信息的响应（已验证）：

```jsonc
// message 类型的响应
{
  "success": true,
  "message": "Message sent to tester-01's inbox",
  "routing": {
    "sender": "team-lead",
    "target": "@tester-01",
    "targetColor": "blue",               // 接收方的颜色标识
    "summary": "唤醒并分配任务系统验证工作",
    "content": "你被唤醒了。请执行以下操作..."
  }
}

// shutdown_request 类型的响应
{
  "success": true,
  "message": "Shutdown request sent to tester-01. Request ID: shutdown-1770536808909@tester-01",
  "request_id": "shutdown-1770536808909@tester-01",  // 格式: shutdown-{timestamp}@{agent_name}
  "target": "tester-01"
}
```

### 4.3 消息投递机制

```
发送方 Agent                    系统                      接收方 Agent
    │                           │                           │
    ├── SendMessage ──────────→│                           │
    │                           │                           │
    │                           ├── 1. 写入接收方 inbox 文件 │
    │                           │   (inboxes/{name}.json,   │
    │                           │    read=false)             │
    │                           │                           │
    │                           ├── 2. 检查接收方状态 ────→│
    │                           │                           │
    │                           │ [接收方空闲]              │
    │                           ├── 通知唤醒 ────────────→│ (读取 inbox)
    │                           │                           │
    │                           │ [接收方忙碌]              │
    │                           ├── (等待 turn 结束)        │
    │                           ├── turn 结束 ───────────→│ (读取 inbox)
    │                           │                           │
    │                           │                           ├── 消费消息
    │                           │                           │   (read→true)
    │                           │                           │
```

**关键特性：**

1. **Inbox 文件落盘**：每条消息写入接收方的 inbox 文件 `~/.claude/teams/{team}/inboxes/{name}.json`（JSON 数组），新消息 `read: false`，消费后更新为 `read: true`
2. **自动投递**：接收方无需主动检查收件箱，消息自动作为新 conversation turn 注入
3. **队列缓冲**：如果接收方正在执行（mid-turn），消息在 inbox 文件中等待，当前 turn 结束后投递
4. **Idle 通知**：Agent 每个 turn 结束后自动进入 idle，系统向 Lead 发送 idle 通知（也写入 Lead 的 inbox 文件）
5. **Peer DM 可见性**：队员之间的 DM 摘要会包含在 idle 通知中，Lead 可见但无需回应
6. **消息持久化**：已消费的消息不从 inbox 文件中删除，保留完整历史直到 TeamDelete 清理

### 4.4 Idle 通知格式

系统自动发送的 idle 通知实际为 JSON 格式（已验证）：

```jsonc
// 基础 idle 通知
{
  "type": "idle_notification",
  "from": "tester-01",
  "timestamp": "2026-02-08T07:38:20.153Z",  // ISO 8601 格式
  "idleReason": "available"                   // 当前仅观察到 "available"
}

// 带 Peer DM 摘要的 idle 通知
{
  "type": "idle_notification",
  "from": "tester-02",
  "timestamp": "2026-02-08T07:46:31.579Z",
  "idleReason": "available",
  "summary": "[to tester-01] P2P通信测试消息发送给tester-01"  // 可选，Peer DM 时携带
}
```

**关键观察：**
- `summary` 字段仅在队员间 P2P 通信时出现，格式为 `[to {recipient}] {摘要内容}`
- 同一个 Agent 可能连续发送多次 idle 通知（例如 turn 结束后快速连续发出 2 次），这是正常行为
- `idleReason` 目前仅观察到 `"available"` 值，可能存在其他未触发的值

### 4.5 Inbox 文件 Schema（已验证）

**路径**：`~/.claude/teams/{team_name}/inboxes/{agentName}.json`

**创建时机**：
- Agent 的 inbox 文件在首次收到消息时创建（如 TaskUpdate 分配 owner 触发 `task_assignment`）
- Team Lead 的 inbox 文件在首次收到 Agent 消息时创建

**文件格式**：JSON 数组，每条消息为一个对象

```jsonc
[
  {
    "from": "team-lead",                  // string: 发送者名称
    "text": "消息正文或 JSON 字符串",       // string: 消息内容（见下方类型说明）
    "timestamp": "2026-02-08T11:36:55Z",  // string: ISO 8601 时间戳
    "read": true,                          // boolean: 是否已被接收方消费
    "summary": "摘要文本",                  // string (可选): SendMessage 的 summary 参数
    "color": "blue"                        // string (可选): 发送方 Agent 的颜色标识
  }
]
```

**`text` 字段的消息类型**：

| 消息类型 | `text` 格式 | 触发来源 |
|---------|-------------|---------|
| 普通消息 | 纯文本 | `SendMessage(type="message")` |
| task_assignment | JSON: `{"type":"task_assignment","taskId":"1",...}` | `TaskUpdate(owner=X)` |
| idle_notification | JSON: `{"type":"idle_notification","idleReason":"available",...}` | Agent turn 结束（系统自动） |
| shutdown_request | JSON: `{"type":"shutdown_request","requestId":"...",...}` | `SendMessage(type="shutdown_request")` |
| shutdown_approved | JSON: `{"type":"shutdown_approved","paneId":"...",...}` | Agent 批准关闭 |

**`read` 字段行为**：
- 写入时默认 `false`
- 接收方进程消费消息后，系统更新为 `true`（**写回同一文件**）
- 已消费的消息**不会被删除**，inbox 文件保留完整消息历史

**`color` 字段**：
- 仅出现在**非 Lead 发送**的消息中
- 值为发送方 Agent 在 config.json 中的颜色标识

**生命周期**：
- TeamDelete 时，`inboxes/` 目录随 `teams/{name}/` 一起被删除

### 4.6 通信拓扑

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

- **星型拓扑为主**：Lead 与每个成员通信
- **支持 P2P**：队员之间可以直接 DM
- **基于文件的消息存储**：所有消息持久化到 `inboxes/{agentName}.json`，由系统负责路由和唤醒

---

## 5. Agent 生命周期

```
┌──────────┐     Task(team_name=X)     ┌──────────┐
│  不存在   │ ────────────────────────→ │  创建中   │
└──────────┘                           └────┬─────┘
                                            │ 注册到 config.json
                                            │ 分配 tmux pane
                                            ▼
                                       ┌──────────┐
                                ┌──────│  活跃中   │◄─────┐
                                │      └────┬─────┘      │
                                │           │ turn 结束   │ 收到消息/任务
                                │           ▼             │
                                │      ┌──────────┐      │
                                │      │  Idle    │──────┘
                                │      └────┬─────┘
                                │           │ shutdown_request
                                │           ▼
                                │      ┌──────────┐
                                │      │ 关闭确认  │
                                │      └────┬─────┘
                                │           │ approve=true
                                │           ▼
                                │      ┌──────────┐
                                └──────│  已终止   │
                                       └──────────┘
```

### 5.1 创建阶段

1. Lead 调用 `Task` 工具，指定 `team_name` 和 `name`
2. 系统在 tmux 中创建新 pane（分配 paneId 如 `%14`）
3. 在新 pane 中启动独立的 Claude 进程
4. 将成员信息写入 `config.json` 的 `members` 数组
5. Agent 的 `prompt` 字段保存了初始指令

### 5.2 活跃阶段

- Agent 独立执行任务，拥有完整的工具集（取决于 `agentType`）
- 可以读写文件、执行命令、调用 API
- 通过 `TaskUpdate` 更新任务状态
- 通过 `SendMessage` 与 Lead 和其他队友通信

### 5.3 Idle 阶段

- 每个 API 回合（turn）结束后自动进入 idle
- 系统自动向 Lead 发送 idle 通知
- **Idle ≠ 终止**：idle 的 Agent 可以接收消息并被唤醒
- 正常工作流：执行 → 发消息 → idle → 被唤醒 → 继续执行

### 5.4 关闭阶段

1. Lead 发送 `shutdown_request`
2. Agent 收到请求，可以：
   - `approve: true` → 进程终止
   - `approve: false` → 继续工作（附带拒绝原因）
3. 全部 Agent 关闭后，Lead 可调用 `TeamDelete` 清理资源

**关闭协议的完整消息链（已验证）：**

```
Lead                        系统                        Agent
  │                          │                           │
  ├─ shutdown_request ─────→│──→ 投递到 Agent ─────────→│
  │                          │                           │
  │                          │       Agent 调用           │
  │                          │    shutdown_response       │
  │                          │      (approve=true)        │
  │                          │                           │
  │◄── shutdown_approved ───│◄── 进程开始终止 ──────────│
  │    (含 paneId,           │                           │
  │     backendType)         │                           │
  │                          │                           │
  │◄── teammate_terminated ─│◄── 进程已终止 ───────────│
  │    (系统消息)             │                           │
```

Lead 收到的实际通知（已验证）：

```jsonc
// 1. Agent 批准关闭
{
  "type": "shutdown_approved",
  "requestId": "shutdown-1770536808909@tester-01",
  "from": "tester-01",
  "timestamp": "2026-02-08T07:46:52.929Z",
  "paneId": "%18",           // tmux pane ID
  "backendType": "tmux"      // 运行后端
}

// 2. 系统确认终止（来自 teammate_id="system"）
{
  "type": "teammate_terminated",
  "message": "tester-01 has shut down."
}
```

**注意**：`teammate_terminated` 和 `shutdown_approved` 几乎同时到达，顺序可能交替（实测中 `teammate_terminated` 略早于 `shutdown_approved`）。

---

## 6. tmux 底层机制深度剖析

> 本章通过创建 `tmux-probe` 团队，实时抓取 tmux 状态、进程树、启动命令等数据，还原了 Agent 从 spawn 到销毁的完整 tmux 层面行为。

### 6.1 tmux 会话布局

Claude Code 团队系统运行在用户已有的 tmux session 中（本例为 `dev`）：

```
tmux session: "dev"
├── window 0
│   ├── pane %0  (ttys003) ← Team Lead: claude --dangerously-skip-permissions
│   │                          PID 58657, 主会话进程
│   ├── pane %17 (ttys030) ← 系统辅助 pane (zsh)
│   │
│   └── pane %20 (ttys036) ← [spawn 时动态创建] Agent: probe-01
│                              PID 9507, TeamDelete 时自动销毁
└── window 1
    └── pane %3  (ttys022) ← 用户终端 (zsh)
```

**关键点**：Agent pane 是**动态创建、动态销毁**的，不会在团队关闭后残留。

### 6.2 Agent Spawn 的 tmux 操作序列

spawn 前后的 pane 状态对比：

```
spawn 前 (3 panes):
  %0  (80x55)  │ %17 (186x55)
               │
────────────────────────────────

spawn 后 (4 panes):
  %0  (80x55)  │ %17 (186x27)    ← 上半部分（被压缩）
               ├──────────────
               │ %20 (186x27)    ← 新 agent pane（下半部分）
────────────────────────────────
```

详细执行步骤：

```
Lead 调用 Task(team_name=X, name=Y)
    │
    ├── 1. tmux split-window — 在当前 window 中垂直分裂出新 pane
    │      → %17 从 186x55 缩为 186x27（上半）
    │      → 新 pane %20 = 186x27（下半）
    │
    ├── 2. 在新 pane 中启动 zsh
    │      PID 9424, PPID=41159 (tmux server)
    │
    ├── 3. 保存 shell 环境快照
    │      ~/.claude/shell-snapshots/snapshot-zsh-{timestamp}-{id}.sh
    │
    ├── 4. 在 zsh 中执行 claude 命令（带完整 agent 参数）
    │      → claude 进程 PID 9507, PPID=9424 (zsh)
    │
    └── 5. 注册到 config.json: tmuxPaneId="%20"
```

### 6.3 Agent 启动命令（实际抓取）

```bash
# Team Lead（主会话，无 agent 参数）
claude --dangerously-skip-permissions

# Agent（完整参数链，每个都有明确用途）
/Users/night/.local/share/claude/versions/2.1.34 \
  --agent-id probe-01@tmux-probe \              # 全局唯一 ID，用于消息路由
  --agent-name probe-01 \                        # 人类可读名称，用于 SendMessage recipient
  --team-name tmux-probe \                       # 所属团队，用于定位 config.json 和 tasks/
  --agent-color blue \                           # UI 颜色标识
  --parent-session-id c93b690c-...-d9451b749ac2 \# Lead 的会话 ID，用于消息回路
  --agent-type general-purpose \                 # Agent 类型，决定可用工具集
  --dangerously-skip-permissions \               # 权限模式（继承自 Lead）
  --model claude-opus-4-6                        # LLM 模型
```

**核心发现**：Agent 本质上就是另一个 `claude` CLI 进程，通过 `--agent-*` 参数区分身份。Lead 和 Agent 运行的是**同一个二进制文件**（`/Users/night/.local/share/claude/versions/2.1.34`）。

**原生 Agent CLI 参数完整列表（二次验证）**：

| 参数 | 用途 | 是否始终存在 |
|------|------|------------|
| `--agent-id` | 全局唯一 ID，用于消息路由 | 是 |
| `--agent-name` | 人类可读名称 | 是 |
| `--team-name` | 所属团队 | 是 |
| `--agent-color` | UI 颜色标识 | 是 |
| `--parent-session-id` | Lead 的会话 ID | 是 |
| `--agent-type` | Agent 类型 | 是 |
| `--model` | LLM 模型 | 是 |
| `--dangerously-skip-permissions` | 绕过权限检查 | **条件性**：仅当 Lead 使用该模式时传递 |
| `--permission-mode` | 权限模式（6 种） | **条件性**：由 Lead 或调用方指定时传递 |
| `--allowedTools` | 工具白名单 | **条件性**：配置了工具限制时传递 |
| `--disallowedTools` | 工具黑名单 | **条件性**：配置了工具限制时传递 |

> 注：上述后四个条件性参数均为 Claude CLI 原生支持（通过 `claude --help` 确认），在 Team spawn 中是否传递取决于 Lead 的配置。`--agent-*` 系列参数为 Team 系统内部参数，不在 `claude --help` 中显示。

### 6.4 进程树（实际抓取）

```
tmux server (PID 41159)
│
├── zsh (PID 41160, pane %0, /dev/ttys003)
│   └── claude --dangerously-skip-permissions (PID 58657)  ← Team Lead
│       ├── caffeinate -i -t 300                           ← 防系统休眠
│       ├── alpaca-mcp-server serve                        ← MCP 服务进程
│       └── pyright-langserver --stdio                     ← 语言服务进程
│
├── zsh (PID 9424, pane %20, /dev/ttys036)                 ← [动态创建]
│   └── claude --agent-id probe-01@... (PID 9507)          ← Agent 进程
│       └── /bin/zsh -c source snapshot-*.sh && eval '...' ← Bash 工具子进程
│
├── zsh (PID 70762, pane %17, /dev/ttys030)                ← 辅助 pane
└── zsh (PID 62613, pane %3, /dev/ttys022)                 ← 用户终端
```

PPID 链路：`Bash 子进程 → claude agent → zsh → tmux server`

### 6.5 TTY 与 stdin 处理

| 进程 | TTY | stdin | 说明 |
|------|-----|-------|------|
| Team Lead (58657) | `/dev/ttys003` | 真实终端 | 接收用户键盘输入 |
| Agent (9507) | `/dev/ttys036` | tmux 伪终端 | 有 TTY 但无人类交互 |
| Agent 的 Bash 子进程 | `??` | `/dev/null` | **无控制终端** |

**重要影响**：
- Agent 的 Bash 工具每次调用都 fork 新的 zsh 子进程
- 子进程先 `source` shell 快照恢复环境，再执行命令
- stdin 来自 `/dev/null`，因此**无法运行交互式命令**（如 `git rebase -i`、`vim`、`less`）
- 环境变量 `TMUX_PANE=%20` 可在 Agent 内部获取

### 6.6 Pane 生命周期与 tmux 操作映射

| 生命周期事件 | tmux 操作 | 进程影响 |
|-------------|-----------|----------|
| Team Lead 启动 | 用户在已有 pane 中运行 `claude` | 无 tmux 操作 |
| Agent spawn | `tmux split-window` → 新 pane | 新 zsh → 新 claude 进程 |
| Agent 活跃 | pane 中运行 claude，可通过 `tmux capture-pane` 观察 | 进程正常运行 |
| Agent idle | 无 tmux 操作 | 进程仍在，只是不做 API 调用 |
| Agent shutdown (approve) | claude 进程退出 → zsh 退出 | pane 可能保留但空 |
| TeamDelete | `tmux kill-pane` 销毁所有 agent pane | 进程树全部清理 |

### 6.7 为什么选择 tmux 作为进程管理器

| 需求 | tmux 的解决方案 | 替代方案对比 |
|------|----------------|-------------|
| 进程隔离 | 每个 pane 独立终端 + 独立进程树 | Docker 过重，subprocess 无终端 |
| 环境继承 | shell-snapshots → source 恢复 | 需要额外的环境传递机制 |
| 免网络通信 | pane 间不需要 socket/端口 | HTTP/gRPC 需要端口管理 |
| 可观测性 | `tmux capture-pane -t %20 -p` 可直接看输出 | subprocess 需要额外日志管道 |
| 资源回收 | `tmux kill-pane` 一行清理 | 需要手动杀进程树 |
| 跨平台 | macOS / Linux 均可用 | Windows 不支持（需 WSL） |
| 用户可见 | 用户可以切换到 agent pane 实时观察 | subprocess 对用户不透明 |

### 6.8 tmux 的局限性

1. **pane 数量限制**：tmux window 中 pane 过多时，每个 pane 尺寸会非常小，影响输出显示
2. **Windows 不支持**：tmux 仅支持 Unix 系统，Windows 用户需要 WSL
3. **pane 分裂方向**：当前使用垂直分裂（水平堆叠），多个 agent 会逐渐压缩 pane 高度
4. **pane ID 不复用**：已销毁 pane 的 ID（如 `%20`）不会被复用，长期运行后 ID 会持续增长

---

## 7. 底层存储架构

### 7.1 `~/.claude/` 目录结构全景

```
~/.claude/
├── CLAUDE.md                     # 用户全局指令
├── settings.json                 # Claude Code 设置
├── history.jsonl                 # 会话历史（JSONL 格式）
├── stats-cache.json              # 统计缓存
├── statusline-command.sh         # 状态栏脚本
│
├── teams/                        # 团队配置
│   └── {team-name}/
│       ├── config.json           # 团队元数据 + 成员列表
│       └── inboxes/              # 消息收件箱（已验证）
│           ├── team-lead.json    # Lead 的消息记录
│           └── {agentName}.json  # 各 Agent 的消息记录
│
├── tasks/                        # 任务数据
│   └── {team-name}/              # 按团队隔离
│       ├── .lock                 # 并发锁
│       └── {id}.json             # 单任务文件
│
├── session-env/                  # 会话环境
│   └── {session-uuid}/           # 每个会话独立目录
│
├── debug/                        # 调试日志
│   ├── {session-uuid}.txt        # 每会话一个日志文件
│   └── latest -> ...             # 指向最新日志的符号链接
│
├── shell-snapshots/              # Shell 环境快照
│   └── snapshot-zsh-{ts}-{id}.sh # zsh 环境快照脚本
│
├── agents/                       # 自定义 Agent 定义
├── commands/                     # 自定义命令
├── hooks/                        # Hook 配置
├── skills/                       # 自定义 Skills
├── plugins/                      # 插件
├── plans/                        # 计划文件
├── todos/                        # Todo 列表
├── projects/                     # 项目配置
├── output-styles/                # 输出样式
├── paste-cache/                  # 粘贴板缓存
├── cache/                        # 通用缓存
├── file-history/                 # 文件修改历史
├── statsig/                      # 特性开关
└── telemetry/                    # 遥测数据
```

### 7.2 数据流

```
TeamCreate
    │
    ├── 写入 ~/.claude/teams/{name}/config.json (初始 lead 成员)
    └── 创建 ~/.claude/tasks/{name}/ 目录 + .lock 文件

TaskCreate
    │
    └── 写入 ~/.claude/tasks/{name}/{id}.json

Task(team_name=X, name=Y)  [spawn agent]
    │
    ├── 更新 ~/.claude/teams/{name}/config.json (添加新成员)
    ├── 创建 tmux pane
    ├── 创建 ~/.claude/session-env/{session-uuid}/
    ├── 写入 ~/.claude/shell-snapshots/snapshot-*.sh
    └── 写入 ~/.claude/debug/{session-uuid}.txt

SendMessage
    │
    ├── 写入接收方 inbox 文件 (inboxes/{recipient}.json, read=false)
    ├── 写入发送方 JSONL（tool_use 记录）
    ├── 唤醒接收方进程（tmux 后端: 匿名 UDS socketpair 通知; in-process 后端: 进程内直接传递）
    └── 接收方消费后更新 inbox 文件 (read=true)
    注："inbox" 是实际的文件系统路径，非内存队列

TeamDelete
    │
    ├── 删除 ~/.claude/teams/{name}/ （含 config.json 和 inboxes/ 目录）
    └── 删除 ~/.claude/tasks/{name}/
```

### 7.3 关键发现

1. **消息传递机制（二次修正，通过 inbox-verify + delivery-probe 团队验证）**：

   前版报告结论"inbox 为进程内存队列"**错误**。实际 inbox 是文件系统上的 JSON 文件：

   ```
   Lead 调用 SendMessage(recipient="watcher", content="...")
       │
       ├── 1. 写入接收方 inbox 文件 ← 主要落盘路径
       │      ~/.claude/teams/{team}/inboxes/watcher.json
       │      (追加消息对象到 JSON 数组, read=false)
       │
       ├── 2. 写入发送方 JSONL（tool_use 记录） ← 对话历史落盘
       │      ~/.claude/projects/{project}/{session-id}.jsonl
       │
       ├── 3. 唤醒接收方进程 ← 通知机制（因后端而异）
       │      tmux 后端: 匿名 UDS socketpair 通知
       │      in-process 后端: 进程内直接传递
       │
       └── 4. 接收方消费消息后更新 inbox 文件 ← read 状态回写
              (read: false → true，消息不删除)
   ```

   **Inbox 文件是真实存在的**：`"Message sent to watcher's inbox"` 中的 inbox 指 `~/.claude/teams/{team}/inboxes/{name}.json` 文件。每个 Agent（包括 Team Lead）都有独立的 inbox 文件。

   **前版报告为何没找到 inbox 文件**：
   - 前版使用 `find ~/.claude/ -name '*inbox*'` 搜索——虽然 `*inbox*` 应能匹配 `inboxes`
   - 但前版报告在 **TeamDelete 之后**执行搜索，此时 `inboxes/` 目录已随团队删除
   - 这导致了错误的"无 inbox 文件"结论

   **二次验证证据（inbox-verify 团队）**：

   | 检查项 | 结果 |
   |--------|------|
   | Agent spawn 后 | `inboxes/listener.json` 立即出现（含 task_assignment 消息） |
   | SendMessage 后 | 消息追加到 `inboxes/listener.json`，`read: false` |
   | Agent 消费后 | 同一文件中 `read` 字段更新为 `true` |
   | Agent 发消息给 Lead 后 | `inboxes/team-lead.json` 被创建 |
   | idle_notification | 以 JSON 字符串形式写入 Lead 的 inbox 文件 |
   | shutdown_request/approved | 均写入对应 inbox 文件 |
   | TeamDelete 后 | `inboxes/` 目录随 `teams/{name}/` 被完整删除 |

   **投递机制因后端而异（delivery-probe + mechanism-verify + tmux-verify 团队验证）**：

   | 后端类型 | 进程模型 | 消息持久化 | 唤醒通知 |
   |---------|---------|-----------|---------|
   | `tmux` | 独立进程（不同 tmux pane） | 写入 inbox 文件 | 匿名 UDS socketpair |
   | `in-process` | 同一进程 | 写入 inbox 文件 | 进程内直接传递 |

   两种后端**都使用 inbox 文件**作为消息持久化层。区别仅在于进程间唤醒通知的方式。

   **唤醒机制深度验证（lsof 实测）**：

   tmux 后端的 Agent 进程中观察到：
   - **4 个 KQUEUE 文件描述符**（macOS 内核事件通知机制）
   - **DIR handles** 指向 `~/.claude/`、`~/.claude/commands`、`~/.claude/skills`、项目 `.claude/` — 用于通用文件系统变更监控（命令/技能热加载等）
   - **匿名 UDS socketpair** — Lead 与 Agent 之间的进程间通知通道（无文件系统路径，非命名 socket）
   - **无** `.sock` / `.socket` 命名 socket 文件

   Lead 进程额外持有 `~/.claude/tasks/{team}/` 目录的 DIR handles（Agent 不持有），用于监控 Task 文件变更。

   ```
   消息投递完整流程（tmux 后端）：
   1. 发送方调用 SendMessage
   2. 系统写入接收方 inbox 文件（JSON 追加，read=false）
   3. 系统通过匿名 UDS socketpair 通知接收方进程
   4. 接收方读取 inbox 文件，消费消息
   5. 接收方更新 inbox 文件（read=true）
   ```

2. **任务即文件**：每个任务是一个独立的 JSON 文件，通过 `.lock` 文件实现简单的并发控制。这是一种轻量级的"数据库"设计。

3. **tmux 作为进程管理器**：Agent 进程运行在 tmux pane 中，利用 tmux 的进程隔离和终端复用能力。

4. **Shell 快照**：每次 spawn agent 时，系统会保存当前 shell 环境的快照（`shell-snapshots/`），确保新 Agent 继承正确的环境变量。

5. **对话持久化（JSONL）**：每个 Agent（包括 Lead）都有独立的 JSONL 对话记录文件，位于 `~/.claude/projects/{project-path}/{session-id}.jsonl`。该文件记录了完整的对话历史，包括工具调用、工具结果、消息收发等。

6. **消息持久化（Inbox 文件）**：每个 Agent 都有独立的 inbox JSON 文件，位于 `~/.claude/teams/{team}/inboxes/{name}.json`。该文件记录了该 Agent 接收的所有消息（含已读状态），是消息投递的主要载体。Inbox 文件在 TeamDelete 时被删除。

7. **Agent 后端类型**：系统支持两种 Agent 后端——`tmux`（独立进程在 tmux pane 中）和 `in-process`（与 Lead 同进程）。后端类型在 config.json 的 `backendType` 字段中标识。当 Claude CLI 进程不在 tmux 环境中运行时（`$TMUX` 未设置），使用 `in-process` 后端。

8. **环境变量**（mechanism-verify + tmux-verify 团队验证）：

   | 变量 | 值 | Lead | tmux Agent | in-process Agent |
   |------|---|------|-----------|-----------------|
   | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | `1` | 有 | 有 | 有 |
   | `CLAUDECODE` | `1` | 有 | 有 | 有 |
   | `CLAUDE_CODE_ENTRYPOINT` | `cli` | 有 | 有 | 有 |
   | `TMUX_PANE` | `%XX` | 有 | 有 | 无 |

   - `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 标识团队模式已启用（实验性功能标志）
   - `CLAUDECODE=1` 标识运行在 Claude Code 环境中
   - Agent 的团队信息（team name、agent name 等）不通过环境变量传递，而是通过 CLI 参数

9. **Task 子任务输出路径**（tmux-verify 团队验证）：

   Task 工具（subagent）的中间输出存储在：
   ```
   /tmp/claude-{uid}/{project-path-escaped}/tasks/{hash}.output
   ```
   例如：`/tmp/claude-501/-Users-night-working-o3cloud-.../tasks/b6b99ac.output`

10. **config.json 并发写入竞争**（tmux-verify 团队发现）：

    多个 Agent 同时注册时，config.json 存在 lost update 问题：多个进程读取同一版本的 config.json，各自追加成员后写回，后写入的覆盖先写入的。在 tmux-verify 团队中，color-04 的注册条目因此丢失（但 Agent 本身通过 `--agent-color` 参数仍持有正确的颜色分配，通信不受影响）。

---

## 8. 实际运行示例

### 8.1 本次分析团队的完整时间线

| 时间 (ms) | 事件 | 涉及文件 |
|-----------|------|----------|
| 1770535107409 | TeamCreate "analysis-team" | `teams/analysis-team/config.json` 创建 |
| 1770535107409 | Team Lead 注册 | config.json members[0] |
| — | TaskCreate #1~#4 | `tasks/analysis-team/1~4.json` |
| — | TaskUpdate #4 blockedBy=[1,2,3] | 4.json + 1/2/3.json 双向更新 |
| 1770535144590 | Spawn researcher-config | config.json members[1], tmux %14 |
| 1770535149817 | Spawn researcher-tasks | config.json members[2], tmux %15 |
| 1770535156897 | Spawn researcher-comms | config.json members[3], tmux %16 |
| — | TaskUpdate #1~#3 owner + in_progress | 各任务 json 更新 |
| — | researcher-config 完成 → SendMessage | 1.json status=completed |
| — | researcher-tasks 完成 → SendMessage | 2.json status=completed |
| — | researcher-comms 完成 → SendMessage | 3.json status=completed |
| — | Task #4 解除阻塞 | 4.json blockedBy 清空 |
| — | Team Lead 编写报告 | 本文件 |

### 8.2 团队配置变化对比

**创建时（仅 lead）：**
```json
"members": [
  { "name": "team-lead", "agentType": "team-lead", "tmuxPaneId": "" }
]
```

**spawn 3 个 agent 后：**
```json
"members": [
  { "name": "team-lead",       "agentType": "team-lead",       "tmuxPaneId": "" },
  { "name": "researcher-config","agentType": "general-purpose", "tmuxPaneId": "%14", "color": "blue" },
  { "name": "researcher-tasks", "agentType": "general-purpose", "tmuxPaneId": "%15", "color": "green" },
  { "name": "researcher-comms", "agentType": "general-purpose", "tmuxPaneId": "%16", "color": "yellow" }
]
```

### 8.3 验证团队（verify-team）完整测试记录

为验证报告准确性，额外创建了 `verify-team` 并执行了系统性测试：

**团队配置**：1 Lead + 2 Testers（tester-01 蓝色, tester-02 绿色）

| # | 验证项 | 操作 | 结果 |
|---|--------|------|------|
| 1 | TeamCreate | 创建 verify-team | config.json 正确生成，含 lead 成员 |
| 2 | Agent spawn | spawn tester-01 | 注册到 config.json，分配 tmux %18，颜色 blue |
| 3 | Agent 自检 | tester-01 读取 config.json | 确认自身已注册，字段完整 |
| 4 | idle 通知 | tester-01 turn 结束 | Lead 收到 `idle_notification(idleReason=available)` |
| 5 | 唤醒 idle Agent | Lead 向 idle 的 tester-01 发消息 | tester-01 被唤醒，执行任务 |
| 6 | 任务 CRUD | tester-01 执行 Create→Update→List→Complete | 四步操作全部成功 |
| 7 | 连续 idle | tester-01 多次 idle | 连续 2 次 idle_notification，正常行为 |
| 8 | P2P 通信 | tester-02 → tester-01 DM | 直接通信成功，无需 Lead 中转 |
| 9 | P2P 唤醒 | tester-02 的消息唤醒 idle 的 tester-01 | tester-01 自动唤醒并回复 |
| 10 | Peer DM 可见性 | Lead 观察 idle 通知 | 摘要格式 `[to tester-01] P2P通信测试消息...` |
| 11 | shutdown 协议 | Lead 同时发送 2 个 shutdown_request | 两个 Agent 均 approve 并终止 |
| 12 | 终止通知 | 系统消息 | 收到 `shutdown_approved` + `teammate_terminated` |
| 13 | TeamDelete | 清理 verify-team | teams/ 和 tasks/ 目录均被删除 |

**P2P 通信验证链路**：
```
tester-02 ──DM──→ tester-01 (idle → 被唤醒)
     idle 通知 → Lead 摘要: [to tester-01] P2P通信测试消息发送给tester-01

tester-01 ──DM──→ tester-02 (回复确认)
     idle 通知 → Lead 摘要: [to tester-02] 确认收到P2P通信测试消息

tester-02 ──SendMessage──→ team-lead (汇总报告)
```

**结论**：分析报告中描述的所有 11 项核心机制均与实际行为一致。

---

## 9. Prompt 工程细节

### 9.1 Agent 创建时的 Prompt 结构

当 Team Lead 通过 `Task` 工具 spawn 一个 Agent 时，`prompt` 参数成为该 Agent 的**初始系统指令**，被完整保存在 `config.json` 的 `prompt` 字段中。

**设计要点：**

1. **角色定义**：明确告知 Agent 所属团队和名称
   ```
   "你是 analysis-team 的成员 researcher-config。"
   ```

2. **任务描述**：详细列出需要执行的步骤
   ```
   "请执行以下操作：
    1. 读取 ~/.claude/teams/...
    2. 查找 ~/.claude/teams/ 下所有..."
   ```

3. **输出指令**：指定结果交付方式
   ```
   "完成后，将分析结果通过 SendMessage 发送给 team-lead。"
   ```

### 9.2 系统注入的上下文

除了用户提供的 `prompt`，系统还会自动注入：

| 上下文 | 来源 | 说明 |
|--------|------|------|
| CLAUDE.md | 项目根目录 | 项目级指令 |
| ~/.claude/CLAUDE.md | 用户主目录 | 用户级全局指令 |
| 工具定义 | 系统 | 可用工具列表和参数 |
| 团队上下文 | config.json | 团队名、成员列表 |
| 环境变量 | shell-snapshots | Shell 环境 |

### 9.3 Agent 类型与工具权限

| agentType | 可用工具 | 适用场景 |
|-----------|----------|----------|
| `general-purpose` | 全部工具（Read, Write, Edit, Bash, Task...） | 编码、分析、综合任务 |
| `Explore` | 只读工具（Read, Grep, Glob, WebFetch...） | 代码搜索、信息收集 |
| `Plan` | 只读工具 | 架构设计、方案规划 |
| `Bash` | 仅 Bash 命令 | 系统操作、脚本执行 |

### 9.4 权限模式（`--permission-mode`）

Claude CLI 原生支持 6 种权限模式，通过 `--permission-mode` 参数传递（`claude --help` 确认）：

| 模式 | CLI 参数 | 行为 |
|------|----------|------|
| `default` | `--permission-mode default` | 默认模式，需要用户确认敏感操作 |
| `acceptEdits` | `--permission-mode acceptEdits` | 自动接受文件编辑操作 |
| `bypassPermissions` | `--dangerously-skip-permissions` | 绕过所有权限检查 |
| `plan` | `--permission-mode plan` | 只读分析模式 |
| `dontAsk` | `--permission-mode dontAsk` | 自动拒绝不确定的操作 |
| `delegate` | `--permission-mode delegate` | 通过 inbox 协议将权限请求转发给 Lead |

> 注：`delegate` 模式下，Agent 执行受限操作时会通过 inbox 发送 `permission_request` 消息，由 Lead/Controller 审批后通过 `permission_response` 回传。该模式在本次验证中未实测触发。

### 9.5 工具白名单/黑名单

Claude CLI 支持通过以下参数控制 Agent 可用的工具集（`claude --help` 确认）：

| 参数 | 用途 | 示例 |
|------|------|------|
| `--allowedTools` / `--allowed-tools` | 工具白名单 | `--allowedTools "Bash(git:*) Edit Read"` |
| `--disallowedTools` / `--disallowed-tools` | 工具黑名单 | `--disallowedTools "Write Bash"` |
| `--tools` | 指定内建工具列表 | `--tools "Bash,Edit,Read"` |

这些参数提供了比 `agentType` 更细粒度的工具访问控制。

---

## 附录 A：与传统微服务架构的类比

| Claude Code 团队系统 | 微服务架构 |
|----------------------|------------|
| Team Lead | API Gateway / Orchestrator |
| Agent (tmux pane) | 微服务实例 (Container) |
| config.json | Service Registry |
| tasks/*.json | Task Queue (简化版) |
| SendMessage + inboxes/ | Message Broker (文件持久化) |
| .lock 文件 | Distributed Lock |
| shell-snapshots | Container Image |
| TeamDelete | Service Mesh Teardown |

## 附录 B：最佳实践

1. **团队规模**：3-5 个 Agent 为宜，过多会增加通信开销
2. **任务粒度**：每个任务应可在单个 Agent 的上下文窗口内完成
3. **依赖管理**：善用 `blocks/blockedBy` 构建 DAG，避免循环依赖
4. **消息节制**：优先使用 `message` 而非 `broadcast`，减少 token 消耗
5. **优雅关闭**：始终通过 `shutdown_request/response` 协议关闭 Agent
6. **资源清理**：工作完成后调用 `TeamDelete` 释放资源
