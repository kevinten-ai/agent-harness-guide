# 架构：两阶段六子系统

## 全局架构

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Agent Harness                                │
│                                                                      │
│  ┌─── Phase 1: Scaffolding ────┐   ┌─── Phase 2: Runtime ────────┐ │
│  │                              │   │                              │ │
│  │  System Prompt Assembly      │   │  ┌──────────────────────┐   │ │
│  │  Tool Schema Registration    │   │  │    Agent Loop         │   │ │
│  │  SubAgent Registry Setup     │──►│  │  (Execution Engine)   │   │ │
│  │  Environment Discovery       │   │  └──────────┬───────────┘   │ │
│  │  Plugin/Skill Loading        │   │             │               │ │
│  │                              │   │  ┌──────────▼───────────┐   │ │
│  └──────────────────────────────┘   │  │  Tool Integration    │   │ │
│                                     │  │  Layer               │   │ │
│                                     │  └──────────┬───────────┘   │ │
│                                     │             │               │ │
│                                     │  ┌──────────▼───────────┐   │ │
│                                     │  │  Context Engine       │   │ │
│                                     │  │  (Compaction/Caching) │   │ │
│                                     │  └──────────┬───────────┘   │ │
│                                     │             │               │ │
│                                     │  ┌──────────▼───────────┐   │ │
│                                     │  │  Memory & State      │   │ │
│                                     │  │  Manager             │   │ │
│                                     │  └──────────┬───────────┘   │ │
│                                     │             │               │ │
│                                     │  ┌──────────▼───────────┐   │ │
│                                     │  │  Verification &      │   │ │
│                                     │  │  Guardrails          │   │ │
│                                     │  └──────────┬───────────┘   │ │
│                                     │             │               │ │
│                                     │  ┌──────────▼───────────┐   │ │
│                                     │  │  Safety Architecture │   │ │
│                                     │  │  (Defense in Depth)  │   │ │
│                                     │  └──────────────────────┘   │ │
│                                     └──────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

## 六大子系统详解

### 1. Agent Loop (执行引擎)

这是使 Agent "自主"的核心循环。

```
                 ┌──────────────┐
                 │ User Request │
                 └──────┬───────┘
                        ▼
              ┌─────────────────────┐
              │ Phase 0: Compaction  │ ← 上下文接近极限时触发
              │ (if near token limit)│
              └─────────┬───────────┘
                        ▼
              ┌─────────────────────┐
              │ Phase 1: Thinking   │ ← 深度推理/规划
              │ (Deliberation)      │
              └─────────┬───────────┘
                        ▼
              ┌─────────────────────┐
              │ Phase 2: Critique   │ ← 质疑自己的计划
              │ (Self-evaluation)   │
              └─────────┬───────────┘
                        ▼
              ┌─────────────────────┐
              │ Phase 3: Action     │ ← 调用 LLM + Tool Schemas
              │ (Tool invocation)   │
              └─────────┬───────────┘
                        ▼
              ┌─────────────────────┐
              │ Decision & Dispatch │ ← 执行工具 + 审批检查
              │ (Execute + Approve) │
              └─────────┬───────────┘
                        ▼
              ┌─────────────────────┐
         ┌────│ Termination Check   │
         │    │ • Final response?   │
         │    │ • Safety cap breach?│
         │    │ • Doom loop?        │
         │    └─────────────────────┘
         │              │
         │ Not Done     │ Done
         │              ▼
         │    ┌─────────────────┐
         └───►│ Next Iteration  │
              └─────────────────┘
```

### 2. Tool Integration Layer (工具集成层)

```
┌─────────────────────────────────────────────────────┐
│                   Tool Registry                      │
│                                                      │
│  Built-in Tools          MCP Tools (延迟加载)          │
│  ┌──────────────┐       ┌──────────────────────┐    │
│  │ Read/Write   │       │ GitHub MCP            │    │
│  │ Edit         │       │ Database MCP          │    │
│  │ Bash         │       │ Slack MCP             │    │
│  │ Glob/Grep    │       │ Custom MCP...         │    │
│  │ WebFetch     │       └──────────────────────┘    │
│  │ Agent        │                                    │
│  │ LSP          │       Discovery: 关键字评分         │
│  │ ...          │       (非穷举枚举)                  │
│  └──────────────┘                                    │
│                                                      │
│  Dispatch: ToolName → TypedHandler → Result          │
└─────────────────────────────────────────────────────┘
```

### 3. Context Engine (上下文引擎)

```
System Prompt 组装 (按优先级排序):

Priority 10-30:  Identity & Safety    (身份和安全策略)
Priority 40-50:  Tool Definitions     (工具定义)
Priority 55-65:  Code Standards       (代码标准/CLAUDE.md)
Priority 70-80:  Provider Specific    (模型特定指令)
Priority 85-95:  Environment Info     (环境信息)

Runtime 管理:
┌────────────────────────────────────┐
│ Token Budget Monitor               │
│   ↓ (接近极限)                      │
│ Progressive Summarization          │
│   • 旧对话渐进摘要                   │
│   • 工具输出截断                     │
│   • 记忆卸载到持久存储               │
│   • 历史裁剪                        │
│   ↓                                │
│ System Reminders 注入               │
│   • 对抗指令衰减                     │
│   • 事件驱动触发                     │
│   ↓                                │
│ Prompt Cache 优化                   │
│   • 静态前缀 → 缓存命中             │
│   • 动态后缀 → 每次变化             │
└────────────────────────────────────┘
```

### 4. Memory & State Manager (记忆与状态管理器)

```
┌──────────────────────────────────────────────┐
│              Memory Architecture              │
│                                              │
│  Tier 1: Working Context (临时)               │
│  ├── 当前对话消息                              │
│  ├── 工具调用结果                              │
│  └── 内部推理状态                              │
│                                              │
│  Tier 2: Session State (会话级)               │
│  ├── 任务进度 (TaskList)                      │
│  ├── Git commits                             │
│  ├── Progress files (推荐 JSON 格式)          │
│  └── 临时工作区状态                            │
│                                              │
│  Tier 3: Long-term Memory (跨会话)           │
│  ├── 用户偏好 & 反馈                          │
│  ├── 项目知识                                │
│  ├── 外部资源索引                             │
│  └── CLAUDE.md / MEMORY.md                   │
│                                              │
│  格式选择: JSON > Markdown                    │
│  原因: 模型不太可能意外覆盖/重格式化 JSON       │
└──────────────────────────────────────────────┘
```

### 5. Verification Layer (验证层)

```
代码变更后自动验证:
┌──────────────────────────────┐
│ 1. 运行测试套件               │
│ 2. 与任务规格对比              │
│ 3. 执行完成前检查清单          │
│ 4. 类型检查 / Lint 通过       │
│ 5. Doom Loop 检测             │
│    • 跟踪每文件编辑次数        │
│    • 检测重复工具调用模式      │
│    • 注入 "reconsider" 提示   │
└──────────────────────────────┘
```

### 6. Safety Architecture (安全架构)

```
纵深防御: 五层独立保护

Layer 1: Prompt Guardrails
├── 安全策略嵌入 System Prompt
├── "先读后改" 规则
└── 危险操作警告

Layer 2: Schema Restrictions
├── Plan-mode 只暴露只读工具
├── SubAgent 工具白名单
└── 不在 Schema 中 = 不可调用

Layer 3: Runtime Approval
├── 手动 / 半自动 / 全自动 三级
├── 模式匹配审批规则
└── 敏感操作拦截

Layer 4: Tool Validation
├── 危险模式黑名单 (rm -rf, --force 等)
├── 过期读取检测
├── 超时保护
└── 命令注入防御

Layer 5: Lifecycle Hooks
├── PreToolUse  → 调用前拦截/修改
├── PostToolUse → 调用后验证
├── Stop        → 任务结束审计
└── SessionStart/End → 会话边界
```

## 数据流全景

```
User Input
    │
    ▼
[Scaffolding] → System Prompt + Tools + Environment
    │
    ▼
[Agent Loop] ─── Think → Critique → Act ───┐
    │                                       │
    ▼                                       ▼
[Tool Registry] ─── Dispatch ──► [Tool Execution]
    │                                       │
    ▼                                       ▼
[Safety Layers] ── Check Each Layer ──► [Approve/Block]
    │                                       │
    ▼                                       ▼
[Context Engine] ── Manage Tokens ──► [Compact if needed]
    │                                       │
    ▼                                       ▼
[Memory Manager] ── Persist State ──► [Working/Session/Long-term]
    │                                       │
    ▼                                       ▼
[Verification] ── Tests + Checks ──► [Pass/Fail → Loop Back]
    │
    ▼
Agent Response / Task Complete
```

## 下一步

→ 阅读 [03-agent-loop.md](./03-agent-loop.md) 深入了解 Agent Loop
