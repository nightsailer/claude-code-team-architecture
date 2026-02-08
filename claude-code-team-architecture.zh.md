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
6. [底层存储架构](#6-底层存储架构)
7. [实际运行示例](#7-实际运行示例)
8. [Prompt 工程细节](#8-prompt-工程细节)

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
| `tmuxPaneId` | string | — | tmux pane ID（如 `%14`），lead 为空字符串 |
| `cwd` | string | — | Agent 的工作目录 |
| `subscriptions` | array | — | 消息订阅列表（当前均为空） |
| `prompt` | string | ✓ | Agent 的初始任务指令（由 Task 工具传入） |
| `color` | string | ✓ | UI 颜色标识，按加入顺序分配（见 2.5 颜色分配规则） |
| `planModeRequired` | boolean | ✓ | 是否要求先提交计划再执行 |
| `backendType` | string | ✓ | 运行后端（当前为 `tmux`） |
| `isActive` | boolean | ✓ | 当前是否处于活跃状态 |

### 2.4 团队命名

- **命名团队**：`TeamCreate` 指定 `team_name`，如 `analysis-team`
- **匿名团队**：历史上也存在以 UUID 命名的团队（如 `e132a923-...`），通常是自动生成的

### 2.5 颜色分配规则（已验证）

成员颜色按 spawn 顺序自动分配，Team Lead 无颜色标记：

| 加入顺序 | 颜色 | 验证实例（analysis-team） | 验证实例（verify-team） |
|----------|------|--------------------------|------------------------|
| 第 1 个成员 | `blue` | researcher-config | tester-01 |
| 第 2 个成员 | `green` | researcher-tasks | tester-02 |
| 第 3 个成员 | `yellow` | researcher-comms | — |

颜色序列推测为 `blue → green → yellow → ...`（更多颜色需更多成员验证）。

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
    │                           ├── 检查接收方状态 ─────→│
    │                           │                           │
    │                           │ [接收方空闲]              │
    │                           ├── 立即投递 ────────────→│ (唤醒处理)
    │                           │                           │
    │                           │ [接收方忙碌]              │
    │                           ├── 加入队列               │
    │                           │   (等待 turn 结束)        │
    │                           ├── turn 结束 ───────────→│ (投递排队消息)
    │                           │                           │
```

**关键特性：**

1. **自动投递**：接收方无需主动检查收件箱，消息自动作为新 conversation turn 注入
2. **队列缓冲**：如果接收方正在执行（mid-turn），消息排队，在当前 turn 结束后投递
3. **Idle 通知**：Agent 每个 turn 结束后自动进入 idle，系统向 Lead 发送 idle 通知
4. **Peer DM 可见性**：队员之间的 DM 摘要会包含在 idle 通知中，Lead 可见但无需回应
5. **收件箱机制**：消息投递到接收方的 inbox（`"Message sent to tester-01's inbox"`），由系统负责唤醒

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

### 4.5 通信拓扑

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
- **无中央消息总线**：消息不经过中间存储，直接由系统路由

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

## 6. 底层存储架构

### 6.1 `~/.claude/` 目录结构全景

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
│       └── config.json           # 团队元数据 + 成员列表
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

### 6.2 数据流

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
    └── 系统内存路由（不持久化消息内容到文件）

TeamDelete
    │
    ├── 删除 ~/.claude/teams/{name}/
    └── 删除 ~/.claude/tasks/{name}/
```

### 6.3 关键发现

1. **消息不落盘**：`SendMessage` 的消息通过系统内存路由，不在文件系统中持久化。这意味着会话结束后消息历史不可恢复（除了在 `debug/*.txt` 日志中可能有间接记录）。

2. **任务即文件**：每个任务是一个独立的 JSON 文件，通过 `.lock` 文件实现简单的并发控制。这是一种轻量级的"数据库"设计。

3. **tmux 作为进程管理器**：Agent 进程运行在 tmux pane 中，利用 tmux 的进程隔离和终端复用能力。

4. **Shell 快照**：每次 spawn agent 时，系统会保存当前 shell 环境的快照（`shell-snapshots/`），确保新 Agent 继承正确的环境变量。

---

## 7. 实际运行示例

### 7.1 本次分析团队的完整时间线

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

### 7.2 团队配置变化对比

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

### 7.3 验证团队（verify-team）完整测试记录

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

## 8. Prompt 工程细节

### 8.1 Agent 创建时的 Prompt 结构

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

### 8.2 系统注入的上下文

除了用户提供的 `prompt`，系统还会自动注入：

| 上下文 | 来源 | 说明 |
|--------|------|------|
| CLAUDE.md | 项目根目录 | 项目级指令 |
| ~/.claude/CLAUDE.md | 用户主目录 | 用户级全局指令 |
| 工具定义 | 系统 | 可用工具列表和参数 |
| 团队上下文 | config.json | 团队名、成员列表 |
| 环境变量 | shell-snapshots | Shell 环境 |

### 8.3 Agent 类型与工具权限

| agentType | 可用工具 | 适用场景 |
|-----------|----------|----------|
| `general-purpose` | 全部工具（Read, Write, Edit, Bash, Task...） | 编码、分析、综合任务 |
| `Explore` | 只读工具（Read, Grep, Glob, WebFetch...） | 代码搜索、信息收集 |
| `Plan` | 只读工具 | 架构设计、方案规划 |
| `Bash` | 仅 Bash 命令 | 系统操作、脚本执行 |

---

## 附录 A：与传统微服务架构的类比

| Claude Code 团队系统 | 微服务架构 |
|----------------------|------------|
| Team Lead | API Gateway / Orchestrator |
| Agent (tmux pane) | 微服务实例 (Container) |
| config.json | Service Registry |
| tasks/*.json | Task Queue (简化版) |
| SendMessage | Message Broker (内存版) |
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
