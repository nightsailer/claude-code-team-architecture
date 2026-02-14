# Claude Code 多 Agent 团队协作机制 — 深度分析报告

> 生成日期：2026-02-08 | 更新日期：2026-02-13
> 分析环境：Claude Code CLI (claude-opus-4-6) v2.1.34 ~ v2.1.41
> 分析团队：analysis-team（1 lead + 3 researchers）
> 验证团队：verify-team, schema-verify, advanced-verify, delegate-verify, plan-reject-verify, subs-verify 等
> 验证覆盖率：V1（84 项，98.8% PASS）+ V2（38 项，100% PASS），共 28 个新发现

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
| `subscriptions` | array | — | 消息订阅列表（预留/未实现字段，运行时无效果，均为空数组） |
| `prompt` | string | ✓ | Agent 的初始任务指令（由 Task 工具传入） |
| `color` | string | ✓ | UI 颜色标识，按加入顺序分配（见 2.5 颜色分配规则） |
| `planModeRequired` | boolean | ✓ | 是否要求先提交计划再执行（所有 teammate 的标准字段，默认 `false`；`true` 时 Agent 启动即进入 plan 模式，需通过 `plan_approval_request/response` 协议退出） |
| `backendType` | string | ✓ | 运行后端：`tmux`（独立 tmux pane 进程）或 `in-process`（与 Lead 同进程） |
| `isActive` | boolean | ✓ | 当前是否处于活跃状态（活跃时 `true`，idle 时 `false`；Agent 终止后成员条目从 `members` 数组中**完全移除**，而非标记为 inactive） |

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
- **详细协议见** [§4.7 Plan Mode 机制](#47-plan-mode-机制)

#### 4.1.6 permission_request / permission_response — 权限请求

当 teammate 以 `--permission-mode acceptEdits`（Task tool `mode="delegate"` 映射）运行时，执行需要权限的操作会向 Lead 发送权限请求：

```jsonc
// Lead 通过 UI 审批后，系统自动生成 permission_response
// Lead 无需手动调用 SendMessage 来处理权限请求
```

- **详细协议见** [§4.8 Permission 协议](#48-permission-协议)

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

// broadcast 类型的响应（已验证）
{
  "success": true,
  "message": "Message broadcast to 2 teammate(s): verifier-01, verifier-02",
  "recipients": ["verifier-01", "verifier-02"],  // 实际接收者列表（message 响应无此字段）
  "routing": {
    "sender": "team-lead",
    "target": "@team",            // broadcast 使用 "@team"，非 "@{name}"
    "summary": "广播测试",
    "content": "全体注意：广播测试消息"
    // 注意：无 targetColor（广播无单一目标色）
  }
}
```

**三种响应格式对比：**

| 字段 | message | broadcast | shutdown_request |
|------|---------|-----------|-----------------|
| `routing.target` | `"@{name}"` | `"@team"` | — |
| `routing.targetColor` | 有 | **无** | — |
| `recipients` | **无** | 有（数组） | — |
| `request_id` | **无** | **无** | 有 |

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
7. **字段映射**：SendMessage 参数与 inbox 落盘字段的对应关系因消息类型而异（完整定义见附录 C.1）：
   - 普通消息 / broadcast：`content` → inbox 外层 `text`（纯文本）
   - shutdown_request：`content` → inbox 内层 `reason`（非外层 `text`）
   - shutdown_response：`approve` + `request_id` → 系统生成 `shutdown_approved` 写入 Lead inbox

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
- Agent 的 inbox 文件在 spawn 时创建——初始 prompt 作为第一条纯文本消息写入（`from: "team-lead"`，无 `summary` 和 `color` 字段）
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

| 消息类型 | `text` 格式 | 触发来源 | 完整字段 |
|---------|-------------|---------|---------|
| 普通消息 | 纯文本 | `SendMessage(type="message")` | — |
| task_assignment | JSON 字符串 | `TaskUpdate(owner=X)` | `type, taskId, subject, description, assignedBy, timestamp` |
| idle_notification | JSON 字符串 | Agent turn 结束（系统自动） | `type, from, timestamp, idleReason`；P2P 时额外含 `summary` |
| shutdown_request | JSON 字符串 | `SendMessage(type="shutdown_request")` | `type, requestId, from, reason, timestamp` |
| shutdown_approved | JSON 字符串 | Agent 批准关闭 | `type, requestId, from, timestamp, paneId, backendType` |
| plan_approval_request | JSON 字符串 | Agent 调用 `ExitPlanMode`（`planModeRequired: true`） | `type, from, timestamp, planFilePath, planContent, requestId` |
| plan_approval_response | JSON 字符串 | 系统自动审批 / Lead 手动发送 | `type, requestId, approved, timestamp`；approve 时含 `permissionMode`，reject 时含 `feedback` |
| permission_request | JSON 字符串 | Agent 执行受限操作（`--permission-mode acceptEdits`） | `type, request_id, agent_id, tool_name, tool_use_id, description, input, permission_suggestions` |
| permission_response | JSON 字符串 | 用户通过 UI 审批 | `type, request_id, subtype`；success 时含 `response`，error 时含 `error` |

> **注意**：`teammate_terminated` 不通过 inbox 文件投递，而是通过系统级会话注入（conversation turn injection）送达 Lead，格式为：`"Task {id} (type: in_process_teammate) (status: completed)"`。

> **待验证**：`shutdown_response(approve=false)` 场景下 Lead inbox 中是否出现 `shutdown_rejected` 类型消息，V1/V2 均未测试。所有已验证的 shutdown 流程均为 approve=true 路径。

> **命名约定差异**：shutdown / plan 系列消息使用 **camelCase** `requestId`，permission 系列使用 **snake_case** `request_id`，这是协议层面的命名不一致，消费方需注意兼容。

**`read` 字段行为**：
- 写入时默认 `false`
- 接收方进程消费消息后，系统更新为 `true`（**写回同一文件**）
- 已消费的消息**不会被删除**，inbox 文件保留完整消息历史

**`color` 字段**：
- 多数非 Lead 发送的消息外层含 `color`（如 DM、idle_notification、permission_request）
- 但并非所有非 Lead 消息都含该字段（如 `plan_approval_request` 外层无 `color`）
- 有 `color` 时，值为发送方 Agent 在 config.json 中的颜色标识

**Inbox 外层可选字段出现矩阵**（`from`, `text`, `timestamp`, `read` 为必选，此处仅列可选字段）：

| 消息来源 / 类型 | `summary` | `color` | 验证来源 |
|---|---|---|---|
| 初始 prompt（Lead→Agent） | ✗ | ✗ | V1-S3, V2-B.2 |
| SendMessage Lead→Agent | ✓ | ✗ | V1-M1 |
| SendMessage Agent→Lead | ✓ | ✓ | V1-M1r |
| P2P DM Agent→Agent | ✓ | ✓ | V1-P2P, V2-B.1 |
| broadcast Lead→All | 待验证 | ✗ | V1-M2（仅验证无 color） |
| task_assignment（系统） | ✗ | ✗ | V1-M3 |
| plan_approval_request | ✗ | ✗ | V2-A.3 |
| permission_request | ✗ | ✓ | V2-C.1.3 |
| idle_notification | 待验证 | 待验证 | 无外层样本 |
| shutdown_request / approved | 待验证 | 待验证 | 无外层样本 |
| plan_approval_response | 待验证 | 待验证 | 无外层样本 |
| permission_response | 待验证 | 待验证 | 无外层样本 |

> task_assignment 的完整外层示例（V1-M3 实测）：`{"from":"team-lead","text":"{\"type\":\"task_assignment\",...}","timestamp":"2026-02-13T10:10:53.467Z","read":true}`（无 `summary`，无 `color`）。

**内层 body 发送方字段冗余说明**：多种消息类型在内层 body 中也含发送方标识，但命名不统一——idle_notification / shutdown_request / shutdown_approved / plan_approval_request 使用 `from`（与外层同名冗余），task_assignment 使用 `assignedBy`，permission_request 使用 `agent_id`，permission_response 无发送方字段。

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

### 4.7 Plan Mode 机制（已验证）

#### 4.7.1 概述

Plan Mode 强制 Agent 先规划再执行。`planModeRequired: true` 的 Agent 启动即进入只读 plan 模式，必须先提交计划，获得审批后才能使用完整工具集。

```
planModeRequired=true 的 Agent 生命周期:

  启动 → [plan 模式，只读] → 编写计划 → ExitPlanMode
                                              │
                                              ├─ plan_approval_request → Lead inbox（信息通知）
                                              ├─ 系统读取 Lead 当前 permissionMode
                                              └─ plan_approval_response (approved=true) → Agent inbox
                                                    │
                                                    ▼
                                              [退出 plan 模式，获得 permissionMode 权限] → 执行计划
```

**关键发现**：plan_approval_request 由 Claude Code **运行时基础设施自动审批**（169ms~878ms），非人工门控。自动审批不可禁用。plan mode 的实际作用是**强制 agent 先规划再执行**，而非要求人类审批。

#### 4.7.2 plan_approval_request 消息格式

Agent 调用 `ExitPlanMode` 后，系统生成此消息写入 Lead 的 inbox：

```jsonc
// inbox 外层格式
{
  "from": "planner-01",
  "text": "<JSON-stringified plan_approval_request>",  // 内层 JSON 字符串
  "timestamp": "2026-02-13T10:43:07.887Z",
  "read": true
  // 注意：无 color 字段（与普通 DM 不同）
}
```

**内层消息体**：

```jsonc
{
  "type": "plan_approval_request",
  "from": "planner-01",                              // 发送方名称
  "timestamp": "2026-02-13T10:43:07.887Z",           // ISO 8601
  "planFilePath": "~/.claude/plans/{random-name}.md", // plan 文件绝对路径
  "planContent": "<完整 Markdown 计划文本>",           // 计划内容（内联）
  "requestId": "plan_approval-{epoch_ms}@{agentName}@{teamName}"
}
```

#### 4.7.3 plan_approval_response 消息格式

系统自动生成（或 Lead 手动通过 `SendMessage` 发送），写入 Agent 的 inbox：

```jsonc
// approve=true（自动审批，169~878ms 延迟）
{
  "type": "plan_approval_response",
  "requestId": "plan_approval-{epoch_ms}@{agentName}@{teamName}",
  "approved": true,
  "timestamp": "2026-02-13T10:43:07.980Z",
  "permissionMode": "acceptEdits"    // 审批后 Agent 获得的权限等级，继承 Lead 当前模式
}

// approve=false（Lead 手动发送的 reject）
{
  "type": "plan_approval_response",
  "requestId": "plan_approval-{epoch_ms}@{agentName}@{teamName}",
  "approved": false,
  "feedback": "计划需要补充：请增加错误处理逻辑",  // 拒绝原因
  "timestamp": "2026-02-13T12:19:03.344Z"
  // 注意：reject 时无 permissionMode 字段
}
```

**approve vs reject 字段对比**：

| 字段 | approve=true | approve=false |
|------|-------------|--------------|
| `type` | `"plan_approval_response"` | `"plan_approval_response"` |
| `requestId` | 有 | 有 |
| `approved` | `true` | `false` |
| `timestamp` | 有 | 有 |
| `permissionMode` | **有**（如 `"acceptEdits"`, `"default"`） | **无** |
| `feedback` | **无** | **有**（拒绝原因） |

#### 4.7.4 自动审批机制

| 测试 | Lead 权限模式 | 自动审批延迟 | 响应 `permissionMode` |
|------|-------------|------------|---------------------|
| #1 | accept edits on | 824ms | `"acceptEdits"` |
| #2 | default | 394ms | `"default"` |
| #3 | default | 878ms | `"default"` |
| #4 | default | 169ms | `"default"` |

- 自动审批由运行时基础设施执行，Lead 正在执行 `sleep` 时也会自动审批
- `permissionMode` 反映 Lead 当前权限模式设置
- 100ms 轮询也无法拦截自动审批
- 同一 `requestId` 可收到多条 response，approve 优先于 reject

#### 4.7.5 Reject 流程

尽管自动审批使正常 reject 无法实现，但 reject 协议本身是完整的：

1. Agent 收到 `approved: false` 后**保持 plan 模式**
2. Agent 读取 `feedback` 内容
3. Agent 根据 feedback 修改计划
4. Agent 重新调用 `ExitPlanMode`，生成**新的 `requestId`**
5. 新的 plan_approval_request 再次被自动审批

#### 4.7.6 planModeRequired vs EnterPlanMode

| 维度 | `planModeRequired: true` | 手动 `EnterPlanMode` |
|------|------------------------|---------------------|
| 触发方式 | Task tool `mode="plan"` | Agent 自主调用 |
| 启动行为 | 启动即进入 plan 模式 | 任意时刻进入 |
| 退出审批 | 需要 `plan_approval_request/response` 协议 | **无需审批**，直接退出 |
| CLI 参数 | `--plan-mode-required`（独立标志） | 无额外参数 |
| 适用范围 | 仅指定的 teammate | 所有 agent 可用 |

> **注意**：非 plan-mode agent 调用 `ExitPlanMode` 是幂等的（返回 `"User has approved exiting plan mode"`），不会产生错误。agent 可自主通过 `EnterPlanMode` 进入 plan 模式，再通过 `ExitPlanMode` 退出，全程无需审批。

### 4.8 Permission 协议（已验证）

当 teammate 以 `--permission-mode acceptEdits` 运行（Task tool `mode="delegate"` 映射）时，执行项目目录外的操作会触发权限请求协议。

#### 4.8.1 permission_request 消息格式

Agent inbox 外层：

```jsonc
{
  "from": "delegate-agent",
  "text": "<JSON-stringified permission_request>",
  "timestamp": "2026-02-13T11:01:02.304Z",
  "color": "purple",    // 包含 color（与 plan_approval_request 不同）
  "read": true
}
```

**内层消息体**：

```jsonc
{
  "type": "permission_request",
  "request_id": "perm-{epoch_ms}-{random_7char}",     // 如 "perm-1770980462304-4uqstf5"
  "agent_id": "delegate-agent",                        // 请求方名称（不含 @team 后缀）
  "tool_name": "Bash",                                 // 请求使用的工具
  "tool_use_id": "toolu_...",                          // Anthropic API tool_use ID
  "description": "Create directory /tmp/delegate-test", // 工具调用描述
  "input": { "command": "mkdir -p /tmp/delegate-test" },// 完整工具调用参数
  "permission_suggestions": [                           // 建议的权限授予方式
    {
      "type": "addDirectories",
      "directories": ["/tmp", "/tmp/delegate-test"],
      "destination": "session"
    }
  ]
}
```

**`permission_suggestions` 三种类型**：

| 类型 | 说明 | 示例 |
|------|------|------|
| `addDirectories` | 添加目录白名单 | `{"type":"addDirectories","directories":["/tmp"],"destination":"session"}` |
| `setMode` | 设置权限模式 | `{"type":"setMode","mode":"acceptEdits","destination":"session"}` |
| `addRules` | 添加工具规则 | `{"type":"addRules","rules":[{"toolName":"Read","ruleContent":"//tmp/**"}],"behavior":"allow","destination":"session"}` |

#### 4.8.2 permission_response 消息格式

用户通过 UI 审批后，系统写入 Agent 的 inbox：

```jsonc
// 成功响应（用户批准）
{
  "type": "permission_response",
  "request_id": "perm-1770980462304-4uqstf5",
  "subtype": "success",
  "response": {
    "updated_input": { /* 原始或修改后的工具参数 */ },
    "permission_updates": []
  }
}

// 拒绝响应（用户拒绝）
{
  "type": "permission_response",
  "request_id": "perm-1770980484351-5rzn82g",
  "subtype": "error",
  "error": "拒绝原因字符串"
}
```

#### 4.8.3 权限触发矩阵（`acceptEdits` 模式）

| 工具 | 操作 | 触发 permission_request |
|------|------|----------------------|
| Read | 读取项目目录内文件 | 否 |
| Bash | 项目目录外 mkdir | **是** |
| Write | 项目目录外创建文件 | **是** |
| Read | 读取项目目录外文件 | **是** |

**结论**：`acceptEdits` 模式下，项目目录内的 Read 不触发权限请求，但外部目录的所有操作（包括 Read）都触发。

### 4.9 Lead Delegate UI 模式（已验证）

> **注意**：Lead Delegate UI 模式与 Task tool 的 `mode="delegate"` 参数是两个独立概念。前者是 Lead 的 UI 模式切换，后者是 teammate 的权限模式。

#### 4.9.1 定义

Delegate 模式是 **Team Lead 的 UI 模式**，通过用户在 Claude Code UI 中按 Tab 键切换。它是纯 UI 层行为——config.json 和 CLI 参数中无任何 delegate 标记。

#### 4.9.2 实现机制

进入 delegate 模式时，系统通过 `system-reminder` 注入约束（每次工具调用后重复注入）：

```
## Delegate Mode

You are in delegate mode for team "{team_name}". In this mode, you can ONLY use the following tools:
- TeammateTool: For spawning teammates, sending messages, and team coordination
- TaskCreate: For creating new tasks
- TaskGet: For retrieving task details
- TaskUpdate: For updating task status and adding comments
- TaskList: For listing all tasks

You CANNOT use any other tools (Bash, Read, Write, Edit, etc.) until you exit delegate mode.
```

#### 4.9.3 可用工具

| 工具 | 可用 | 用途 |
|------|------|------|
| TeammateTool（Task/SendMessage） | ✅ | spawn teammate、发送消息 |
| TaskCreate / TaskGet / TaskUpdate / TaskList | ✅ | 任务 CRUD |
| Bash / Read / Write / Edit / Glob / Grep | ❌ | 被禁用 |

#### 4.9.4 对 Teammate 的影响

Lead 的 delegate UI 模式**不影响 teammate 的 CLI 参数和权限**。在 delegate 模式下 spawn 的 teammate（未指定 Task tool `mode`）与普通模式下 spawn 的 teammate 完全一致。

#### 4.9.5 已知限制

退出 delegate 模式后，部分工具**可能不立即恢复**。实测中 Bash 和 Read 仍报 `"No such tool available"`，而 Glob 可用。推测与上下文中工具列表的缓存有关。

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
   - `approve: true` → 进程终止，Lead inbox 收到 `shutdown_approved` 消息
   - `approve: false` → 继续工作（附带拒绝原因）；**reject 场景的 Lead inbox 消息格式待验证**（V1/V2 均未测试此路径）
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

// 2. 系统确认终止（非 inbox 投递，而是系统会话注入）
// Lead 实际收到的是 Task 工具完成通知，格式为：
// "Task {id} (type: in_process_teammate) (status: completed)"
```

**注意**：
- `shutdown_approved` 通过 inbox 文件投递，Lead 在消费 inbox 时读取
- `teammate_terminated` **不通过 inbox 文件投递**，而是通过系统级会话注入（conversation turn injection）送达 Lead
- 两者几乎同时到达，顺序可能交替
- Agent 终止后，其成员条目从 config.json 的 `members` 数组中**完全移除**

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
| `--plan-mode-required` | 强制 Agent 启动即进入 plan 模式 | **条件性**：Task tool `mode="plan"` 时传递（独立布尔标志，与 `--permission-mode` 正交） |
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
| `delegate` | `--permission-mode delegate` | 通过 inbox 协议将权限请求转发给 Lead（详见 §4.8） |

> 注：`delegate` 模式下（实际映射为 `acceptEdits`），Agent 执行受限操作时通过 inbox 发送 `permission_request` 消息，由用户通过 UI 审批后系统生成 `permission_response` 回传。（已验证）

#### Task tool `mode` 参数与 CLI 参数的映射关系（已验证）

| Task tool `mode` | CLI 参数 | config.json 字段 | 说明 |
|-------------------|---------|-----------------|------|
| `"plan"` | `--plan-mode-required` | `planModeRequired: true` | 独立布尔标志，与权限模式正交 |
| `"delegate"` | `--permission-mode acceptEdits` | **无**（权限模式不记录在 config.json 中） | 内部映射，仅通过 CLI 参数传递 |
| `"acceptEdits"` | `--permission-mode acceptEdits` | **无** | 直接映射 |
| `"bypassPermissions"` | `--dangerously-skip-permissions` | **无** | 直接映射 |
| 未指定 | 无 `--permission-mode` | **无** | 继承默认行为 |

**关键发现**：teammate 的权限模式**不记录在 config.json 中**，仅通过 CLI 参数传递。从 config.json 无法推断 teammate 的权限模式。

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

---

## 附录 C：完整 Schema 参考

> 本附录汇总所有消息体和文件格式的完整定义，作为自包含的协议参考手册。所有 Schema 均基于 V1（84 项）+ V2（38 项）黑盒验证数据。

### C.1 Inbox 消息完整定义

#### C.1.1 Inbox 文件格式

**路径**：`~/.claude/teams/{team_name}/inboxes/{agentName}.json`

**格式**：JSON 数组，每条消息为一个对象

```jsonc
[
  {
    // === 必选字段 ===
    "from": "team-lead",                  // string: 发送方名称
    "text": "消息正文或 JSON 字符串",       // string: 纯文本或 JSON-stringified 消息体
    "timestamp": "2026-02-13T10:11:35Z",  // string: ISO 8601
    "read": false,                         // boolean: 消费后由系统更新为 true

    // === 可选字段 ===
    "summary": "摘要文本",                  // string: SendMessage 的 summary 参数
    "color": "blue"                        // string: 发送方 Agent 的颜色标识
  }
]
```

**创建时机**：
- Agent inbox：spawn 时创建（初始 prompt 为第一条消息）
- Lead inbox：首次收到 Agent 消息时创建

**生命周期**：TeamDelete 时随 `teams/{name}/` 一起删除。已消费消息不删除，保留完整历史。

#### C.1.2 纯文本消息

**触发**：`SendMessage(type="message")` / `SendMessage(type="broadcast")` / Agent spawn 初始 prompt

```jsonc
// ① 初始 prompt（Lead → Agent，spawn 时自动写入）
{
  "from": "team-lead",
  "text": "你是 analysis-team 的成员 researcher-config...",
  "timestamp": "2026-02-13T10:05:44.590Z",
  "read": true
  // 无 summary，无 color
}

// ② SendMessage Lead → Agent
{
  "from": "team-lead",
  "text": "这是格式验证消息，请确认收到。",
  "timestamp": "2026-02-13T10:11:35.247Z",
  "read": true,
  "summary": "格式验证测试"
  // 有 summary，无 color（Lead 无颜色）
}

// ③ SendMessage Agent → Lead
{
  "from": "verifier-01",
  "text": "Markdown 格式的分析报告...",
  "timestamp": "2026-02-13T10:10:16.933Z",
  "read": true,
  "summary": "返回 config.json 成员条目和 inbox 内容",
  "color": "blue"
  // 有 summary，有 color
}

// ④ P2P DM（Agent → Agent）
{
  "from": "alice",
  "text": "Alice到Bob的P2P测试消息",
  "summary": "Alice-P2P-DM",
  "timestamp": "2026-02-13T10:45:20.631Z",
  "color": "green",
  "read": true
  // 有 summary，有 color
}

// ⑤ broadcast（每个接收方 inbox 中各一条，独立投递）
{
  "from": "team-lead",
  "text": "全体注意：广播测试消息",
  "timestamp": "2026-02-13T10:11:56.621Z",
  "read": true
  // 无 color（Lead 无颜色）；summary 待验证
}
```

**字段映射**：SendMessage `content` 参数 → inbox 外层 `text` 字段（纯文本直传）。

#### C.1.3 task_assignment

**触发**：`TaskUpdate(owner=X)` 分配任务 owner 时系统自动生成

**外层**（V1-M3 实测）：

```jsonc
{
  "from": "team-lead",
  "text": "{\"type\":\"task_assignment\",\"taskId\":\"1\",\"subject\":\"验证任务A\",\"description\":\"用于格式验证的测试任务\",\"assignedBy\":\"team-lead\",\"timestamp\":\"2026-02-13T10:10:53.467Z\"}",
  "timestamp": "2026-02-13T10:10:53.467Z",
  "read": true
  // 无 summary，无 color
}
```

**内层**（`text` 字段 JSON.parse 后）：

```jsonc
{
  "type": "task_assignment",           // string: 固定值
  "taskId": "1",                       // string: 任务 ID
  "subject": "验证任务A",              // string: 任务标题
  "description": "用于格式验证的测试任务", // string: 任务详情
  "assignedBy": "team-lead",           // string: 分配者名称（非 "from"）
  "timestamp": "2026-02-13T10:10:53.467Z"  // string: ISO 8601
}
```

#### C.1.4 idle_notification

**触发**：Agent 每个 turn 结束后系统自动生成，写入 Lead 的 inbox

**内层**：

```jsonc
// 基础变体（无 DM 发送时）
{
  "type": "idle_notification",           // string: 固定值
  "from": "verifier-01",                 // string: Agent 名称
  "timestamp": "2026-02-13T10:10:22.049Z", // string: ISO 8601
  "idleReason": "available"              // string: 目前仅观察到 "available"
}

// P2P 变体（Agent 刚向另一 Agent 发送了 DM）
{
  "type": "idle_notification",
  "from": "alice",
  "timestamp": "2026-02-13T10:45:27.138Z",
  "idleReason": "available",
  "summary": "[to bob] Alice-P2P-DM"     // string: 格式 "[to {recipient}] {SendMessage的summary}"
}
```

> 注：同一 Agent 可能连续发送多次 idle 通知（2~4 次），属正常行为。`summary` 仅在 P2P DM 后出现。

#### C.1.5 shutdown_request

**触发**：`SendMessage(type="shutdown_request")`

**内层**：

```jsonc
{
  "type": "shutdown_request",            // string: 固定值
  "requestId": "shutdown-1770977603516@verifier-01",  // string: shutdown-{epoch_ms}@{agentName}
  "from": "team-lead",                   // string: 发送方
  "reason": "验证完成，请关闭",           // string: 对应 SendMessage 的 content 参数
  "timestamp": "2026-02-13T10:13:23.516Z" // string: ISO 8601
}
```

> **字段映射**：SendMessage `content` → 内层 `reason`（非外层 `text`，与普通消息不同）。

#### C.1.6 shutdown_approved

**触发**：Agent 调用 `SendMessage(type="shutdown_response", approve=true)` 后系统生成

**内层**：

```jsonc
{
  "type": "shutdown_approved",           // string: 固定值
  "requestId": "shutdown-1770977604066@verifier-02",  // string: 与 request 对应
  "from": "verifier-02",                 // string: Agent 名称
  "timestamp": "2026-02-13T10:13:29.576Z", // string: ISO 8601
  "paneId": "%71",                       // string: tmux pane ID
  "backendType": "tmux"                  // string: "tmux" | "in-process"
}
```

> **待验证**：Agent 调用 `shutdown_response(approve=false)` 时 Lead inbox 中是否出现对应消息（如 `shutdown_rejected`），V1/V2 均未覆盖此路径。

#### C.1.7 plan_approval_request

**触发**：`planModeRequired: true` 的 Agent 调用 `ExitPlanMode`

**外层**（V2-A.3 实测）：

```jsonc
{
  "from": "planner-01",
  "text": "<下方 JSON 的字符串化>",
  "timestamp": "2026-02-13T10:43:07.887Z",
  "read": true
  // 无 summary，无 color（与其他 Agent 发送的消息不同）
}
```

**内层**：

```jsonc
{
  "type": "plan_approval_request",       // string: 固定值
  "from": "planner-01",                  // string: Agent 名称
  "timestamp": "2026-02-13T10:43:07.887Z", // string: ISO 8601
  "planFilePath": "/Users/night/.claude/plans/snug-inventing-popcorn.md",
                                          // string: plan 文件绝对路径（~/.claude/plans/{random-name}.md）
  "planContent": "# 计划标题\n\n## 步骤...", // string: 完整 Markdown 计划文本（内联）
  "requestId": "plan_approval-1770979387887@planner-01@advanced-verify"
                                          // string: plan_approval-{epoch_ms}@{agentName}@{teamName}
}
```

#### C.1.8 plan_approval_response

**触发**：系统自动审批（169~878ms 延迟）/ Lead 手动通过 `SendMessage(type="plan_approval_response")` 发送

**内层（approve=true，系统自动审批）**：

```jsonc
{
  "type": "plan_approval_response",      // string: 固定值
  "requestId": "plan_approval-1770979387887@planner-01@advanced-verify",
                                          // string: 与 request 对应
  "approved": true,                       // boolean
  "timestamp": "2026-02-13T10:43:07.980Z", // string: ISO 8601
  "permissionMode": "acceptEdits"         // string: 继承 Lead 当前权限模式（"default"|"acceptEdits" 等）
}
```

**内层（approve=false，reject）**：

```jsonc
{
  "type": "plan_approval_response",
  "requestId": "plan_approval-1770985126334@planner-reject@plan-reject-verify",
  "approved": false,
  "feedback": "计划需要补充：请增加对子目录中 .md 文件的扫描，不要仅限于根目录。",
                                          // string: 拒绝原因
  "timestamp": "2026-02-13T12:19:03.344Z"
  // 无 permissionMode
}
```

**条件字段对比**：

| 字段 | `approved: true` | `approved: false` |
|---|---|---|
| `permissionMode` | ✓（继承 Lead 模式） | ✗ |
| `feedback` | ✗ | ✓（拒绝原因字符串） |

#### C.1.9 permission_request

**触发**：`--permission-mode acceptEdits`（Task tool `mode="delegate"` 映射）的 Agent 执行项目目录外操作

**外层**（V2-C.1.3 实测）：

```jsonc
{
  "from": "delegate-agent",
  "text": "<下方 JSON 的字符串化>",
  "timestamp": "2026-02-13T11:01:02.304Z",
  "color": "purple",                     // 有 color（与 plan_approval_request 不同）
  "read": true
  // 无 summary
}
```

**内层**：

```jsonc
{
  "type": "permission_request",          // string: 固定值
  "request_id": "perm-1770980462304-4uqstf5",
                                          // string: perm-{epoch_ms}-{random_7char}（注意: snake_case）
  "agent_id": "delegate-agent",           // string: 不含 @team 后缀
  "tool_name": "Bash",                    // string: 请求使用的工具名
  "tool_use_id": "toolu_01ABC...",        // string: Anthropic API tool_use ID
  "description": "Create directory /tmp/delegate-test",
                                          // string: 工具调用描述
  "input": {                              // object: 完整工具调用参数
    "command": "mkdir -p /tmp/delegate-test"
  },
  "permission_suggestions": [             // array: 建议的权限授予方式（0~N 项）
    {
      "type": "addDirectories",           // "addDirectories" | "setMode" | "addRules"
      "directories": ["/tmp", "/tmp/delegate-test"],
      "destination": "session"
    }
  ]
}
```

**`permission_suggestions` 三种类型**：

| `type` | 专有字段 | 说明 |
|---|---|---|
| `addDirectories` | `directories: string[]`, `destination: string` | 添加目录白名单 |
| `setMode` | `mode: string`, `destination: string` | 设置权限模式 |
| `addRules` | `rules: [{toolName, ruleContent}]`, `behavior: string`, `destination: string` | 添加工具规则 |

#### C.1.10 permission_response

**触发**：用户通过 UI 审批或拒绝 permission_request

**内层（success，用户批准）**：

```jsonc
{
  "type": "permission_response",         // string: 固定值
  "request_id": "perm-1770980462304-4uqstf5",
                                          // string: 与 request 对应（snake_case）
  "subtype": "success",                   // string: "success" | "error"
  "response": {
    "updated_input": {                    // object: 原始或修改后的工具参数（Lead 可在 UI 中修改）
      "command": "mkdir -p /tmp/delegate-test"
    },
    "permission_updates": []              // array: 权限更新（目前观察均为空数组）
  }
}
```

**内层（error，用户拒绝）**：

```jsonc
{
  "type": "permission_response",
  "request_id": "perm-1770980484351-5rzn82g",
  "subtype": "error",
  "error": "拒绝原因字符串"               // string: 人类可读的拒绝理由
}
```

**条件字段对比**：

| 字段 | `subtype: "success"` | `subtype: "error"` |
|---|---|---|
| `response` | ✓（含 `updated_input` + `permission_updates`） | ✗ |
| `error` | ✗ | ✓（拒绝原因字符串） |

#### C.1.11 teammate_terminated（非 Inbox 投递）

**投递方式**：系统级会话注入（conversation turn injection），**不经过 inbox 文件**

**格式**：纯文本字符串
```
Task {id} (type: in_process_teammate) (status: completed)
```

---

### C.2 SendMessage 响应格式

#### C.2.1 message 响应

```jsonc
{
  "success": true,
  "message": "Message sent to {name}'s inbox",        // string: 描述文本
  "routing": {
    "sender": "team-lead",                             // string: 发送方名称
    "target": "@verifier-01",                          // string: "@" + 接收方名称
    "targetColor": "blue",                             // string: 接收方颜色
    "summary": "格式验证测试",                          // string: SendMessage summary 参数
    "content": "这是格式验证消息，请确认收到。"          // string: SendMessage content 参数
  }
}
```

#### C.2.2 broadcast 响应

```jsonc
{
  "success": true,
  "message": "Message broadcast to 2 teammate(s): verifier-01, verifier-02",
  "recipients": ["verifier-01", "verifier-02"],        // string[]: 实际接收者列表
  "routing": {
    "sender": "team-lead",
    "target": "@team",                                 // string: 固定值 "@team"
    "summary": "广播测试",
    "content": "全体注意：广播测试消息"
    // 无 targetColor（广播无单一目标色）
  }
}
```

#### C.2.3 shutdown_request 响应

```jsonc
{
  "success": true,
  "message": "Shutdown request sent to verifier-01. Request ID: shutdown-1770977603516@verifier-01",
  "request_id": "shutdown-1770977603516@verifier-01",  // string: shutdown-{epoch_ms}@{agentName}
  "target": "verifier-01"                              // string: 接收方名称
}
```

#### C.2.4 三种响应格式对比

| 字段 | `message` | `broadcast` | `shutdown_request` |
|---|---|---|---|
| `routing` | ✓ | ✓ | ✗ |
| `routing.target` | `"@{name}"` | `"@team"` | — |
| `routing.targetColor` | ✓ | ✗ | — |
| `recipients` | ✗ | ✓（数组） | ✗ |
| `request_id` | ✗ | ✗ | ✓ |
| `target` | ✗ | ✗ | ✓ |

---

### C.3 config.json 完整 Schema

**路径**：`~/.claude/teams/{team_name}/config.json`

```jsonc
{
  // === 顶层字段（5 个） ===
  "name": "analysis-team",                // string: 团队名称（唯一标识）
  "description": "团队描述",               // string: 用途说明
  "createdAt": 1770535107409,             // number: Unix 毫秒时间戳
  "leadAgentId": "team-lead@analysis-team", // string: "{name}@{team}" 格式
  "leadSessionId": "c93b690c-...",        // string: Lead 的会话 UUID

  // === 成员列表 ===
  "members": [
    // --- Team Lead（8 个字段） ---
    {
      "agentId": "team-lead@analysis-team",  // string: {name}@{team}
      "name": "team-lead",                   // string: 通信标识
      "agentType": "team-lead",              // string: 固定值
      "model": "claude-opus-4-6",            // string: 模型 ID
      "joinedAt": 1770535107409,             // number: Unix 毫秒时间戳
      "tmuxPaneId": "",                      // string: Lead 为空字符串
      "cwd": "/path/to/project",             // string: 工作目录
      "subscriptions": []                    // array: 预留字段（运行时无效果）
    },

    // --- Teammate（13 个字段） ---
    {
      "agentId": "researcher-01@analysis-team", // string: {name}@{team}
      "name": "researcher-01",               // string: SendMessage recipient 使用此名称
      "agentType": "general-purpose",        // string: "general-purpose"|"Explore"|"Plan"|"Bash"
      "model": "claude-opus-4-6",            // string: 模型 ID
      "prompt": "你是 analysis-team 的成员...", // string: 初始指令（Lead 无此字段）
      "color": "blue",                       // string: UI 颜色标识（Lead 无此字段）
      "planModeRequired": false,             // boolean: 默认 false（Lead 无此字段）
      "joinedAt": 1770535144590,             // number: Unix 毫秒时间戳
      "tmuxPaneId": "%14",                   // string: tmux pane ID（Lead 无此字段实质值）
      "cwd": "/path/to/project",             // string: 工作目录
      "subscriptions": [],                   // array: 预留字段
      "backendType": "tmux",                 // string: "tmux"|"in-process"（Lead 无此字段）
      "isActive": true                       // boolean: 活跃 true / idle false（Lead 无此字段）
    }
  ]
}
```

**Lead vs Teammate 字段对比**：

| 字段 | Lead (8) | Teammate (13) | 说明 |
|---|---|---|---|
| `agentId` | ✓ | ✓ | 共有 |
| `name` | ✓ | ✓ | 共有 |
| `agentType` | ✓ (`"team-lead"`) | ✓ (`"general-purpose"` 等) | 共有 |
| `model` | ✓ | ✓ | 共有 |
| `joinedAt` | ✓ | ✓ | 共有 |
| `tmuxPaneId` | ✓ (`""`) | ✓ (`"%14"` 等) | 共有（Lead 为空串） |
| `cwd` | ✓ | ✓ | 共有 |
| `subscriptions` | ✓ (`[]`) | ✓ (`[]`) | 共有（预留字段） |
| `prompt` | ✗ | ✓ | Teammate 专有 |
| `color` | ✗ | ✓ | Teammate 专有 |
| `planModeRequired` | ✗ | ✓ | Teammate 专有（默认 `false`） |
| `backendType` | ✗ | ✓ | Teammate 专有 |
| `isActive` | ✗ | ✓ | Teammate 专有 |

**成员生命周期**：Agent 终止后，其条目从 `members` 数组中**完全移除**（非标记为 inactive）。

---

### C.4 Task 文件完整 Schema

**路径**：`~/.claude/tasks/{team_name}/{id}.json`

```jsonc
{
  "id": "1",                              // string: 自增 ID
  "subject": "分析配置文件结构",           // string: 任务标题（祈使句式）
  "description": "详细描述...",            // string: 任务详情和验收标准
  "activeForm": "分析配置文件结构中",      // string: 进行中的显示文本（现在进行时）
  "status": "pending",                    // string: "pending"|"in_progress"|"completed"|"deleted"
  "owner": "researcher-01",              // string|undefined: 负责人名称（新建时不存在）
  "blocks": ["4"],                        // string[]: 本任务阻塞的下游任务 ID
  "blockedBy": [],                        // string[]: 阻塞本任务的上游任务 ID
  "metadata": {}                          // object|undefined: 附加元数据（新建时不存在）
}
```

**状态机**：`pending` → `in_progress` → `completed`（`deleted` 为终态，可从任意状态进入）

**依赖关系**：`blocks` / `blockedBy` 为系统自动维护的双向引用（设置一方时另一方自动更新）。

**并发控制**：同目录下 `.lock` 文件（0 字节），用于文件系统级锁。

---

### C.5 Agent CLI 参数完整列表

```bash
/Users/night/.local/share/claude/versions/{version} \
  --agent-id {name}@{team}             \  # 全局唯一 ID（必选）
  --agent-name {name}                  \  # 人类可读名称（必选）
  --team-name {team}                   \  # 所属团队（必选）
  --agent-color {color}                \  # UI 颜色（必选）
  --parent-session-id {uuid}           \  # Lead 会话 ID（必选）
  --agent-type {type}                  \  # Agent 类型（必选）
  --model {model_id}                   \  # LLM 模型（必选）
  --plan-mode-required                 \  # 条件性：Task mode="plan" 时传递（独立布尔标志）
  --dangerously-skip-permissions       \  # 条件性：Lead 使用该模式时继承
  --permission-mode {mode}             \  # 条件性：Task mode="delegate" → "acceptEdits"
  --allowedTools "..."                 \  # 条件性：工具白名单
  --disallowedTools "..."                 # 条件性：工具黑名单
```

**Task tool `mode` 参数与 CLI 参数的映射**：

| Task tool `mode` | CLI 参数 | config.json 字段 |
|---|---|---|
| `"plan"` | `--plan-mode-required` | `planModeRequired: true` |
| `"delegate"` | `--permission-mode acceptEdits` | 无（不记录在 config.json） |
| `"acceptEdits"` | `--permission-mode acceptEdits` | 无 |
| `"bypassPermissions"` | `--dangerously-skip-permissions` | 无 |
| 未指定 | 无 `--permission-mode` | 无 |

---

### C.6 requestId 命名约定与格式

| 消息类型 | ID 字段名 | 格式模板 | 命名风格 |
|---|---|---|---|
| shutdown_request | `requestId` | `shutdown-{epoch_ms}@{agentName}` | camelCase |
| shutdown_approved | `requestId` | 同 request | camelCase |
| plan_approval_request | `requestId` | `plan_approval-{epoch_ms}@{agentName}@{teamName}` | camelCase |
| plan_approval_response | `requestId` | 同 request | camelCase |
| permission_request | `request_id` | `perm-{epoch_ms}-{random_7char}` | **snake_case** |
| permission_response | `request_id` | 同 request | **snake_case** |

> shutdown/plan 系列使用 camelCase `requestId`，permission 系列使用 snake_case `request_id`，属协议层面的命名不一致。

---

### C.7 关键目录与文件结构

```
~/.claude/
├── teams/{team_name}/
│   ├── config.json                    # 团队配置（见 C.3）
│   └── inboxes/
│       ├── team-lead.json             # Lead inbox（见 C.1）
│       └── {agentName}.json           # Agent inbox（见 C.1）
│
├── tasks/{team_name}/
│   ├── .lock                          # 并发锁（0 字节）
│   └── {id}.json                      # 任务文件（见 C.4）
│
├── plans/
│   └── {random-name}.md               # Plan Mode 计划文件
│
├── shell-snapshots/
│   └── snapshot-zsh-{ts}-{id}.sh      # Shell 环境快照
│
├── session-env/{session-uuid}/         # 会话环境
├── debug/{session-uuid}.txt            # 调试日志（latest 符号链接指向最新）
│
├── projects/{project-path}/
│   └── {session-id}.jsonl             # 对话历史（JSONL 格式）
│
└── /tmp/claude-{uid}/{project-path-escaped}/
    └── tasks/{hash}.output            # Task 子任务中间输出
```

---

### C.8 消息类型速查总表

| 消息类型 | `text` 格式 | 触发来源 | 投递方式 | 内层字段数 |
|---|---|---|---|---|
| 普通消息 | 纯文本 | `SendMessage(message/broadcast)` | inbox 文件 | — |
| `task_assignment` | JSON | `TaskUpdate(owner=X)` | inbox 文件 | 6 |
| `idle_notification` | JSON | Agent turn 结束（系统） | inbox 文件 | 4~5 |
| `shutdown_request` | JSON | `SendMessage(shutdown_request)` | inbox 文件 | 5 |
| `shutdown_approved` | JSON | Agent approve 关闭 | inbox 文件 | 6 |
| `plan_approval_request` | JSON | Agent `ExitPlanMode` | inbox 文件 | 6 |
| `plan_approval_response` | JSON | 系统自动审批 / Lead 手动 | inbox 文件 | 4~5 |
| `permission_request` | JSON | Agent 执行受限操作 | inbox 文件 | 8 |
| `permission_response` | JSON | 用户 UI 审批 | inbox 文件 | 3~4 |
| `teammate_terminated` | — | 进程终止（系统） | **会话注入**（非 inbox） | — |
