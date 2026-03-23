# 核心概念与术语

## 概念关系图

```
┌──────────────────────────────────────────────────────────┐
│                    Agent Harness                          │
│                                                          │
│  ┌─────────┐    ┌──────────────┐    ┌────────────────┐  │
│  │ Context  │◄──►│  Agent Loop   │◄──►│ Tool Registry  │  │
│  │Engineering│   │              │    │                │  │
│  └────┬─────┘   └──────┬───────┘    └───────┬────────┘  │
│       │                │                     │           │
│  ┌────▼─────┐   ┌──────▼───────┐    ┌───────▼────────┐  │
│  │ Memory   │   │ Verification │    │ MCP Protocol   │  │
│  │ System   │   │ & Guardrails │    │ (工具标准)      │  │
│  └──────────┘   └──────────────┘    └────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## 核心术语表

### A. Agent Loop (代理循环)

Harness 的执行引擎。每次迭代通常包含：

```
┌─────────────────────────────────────┐
│ 1. Context Compaction (如需)         │
│    ↓                                │
│ 2. Thinking Phase (思考/推理)        │
│    ↓                                │
│ 3. Self-Critique Phase (自我审视)    │
│    ↓                                │
│ 4. Action Phase (调用工具)           │
│    ↓                                │
│ 5. Verification (验证结果)           │
│    ↓                                │
│ 6. Termination Check (是否结束)      │
└─────────────────────────────────────┘
```

**关键特性**: 模型自己决定调用什么工具、以什么顺序、何时结束。

### B. Context Engineering (上下文工程)

精心构建和管理提供给模型的信息。核心活动：

| 活动 | 说明 |
|------|------|
| **System Prompt 动态组装** | 按优先级模块化组装 (安全→工具→标准→环境) |
| **自适应压缩** | 渐进式摘要旧对话、截断工具输出、卸载记忆 |
| **System Reminder** | 事件驱动的 prompt 注入，对抗长会话中的"指令衰减" |
| **Prompt Caching** | 静态部分放前面，变量内容放后面，命中前缀缓存 |

### C. Tool Registry (工具注册表)

管理所有可用工具的中央调度器：

```
Tool Registry
├── File Operations (Read, Write, Edit, Glob)
├── Shell Execution (Bash, 沙箱化)
├── Web Interaction (Fetch, Search)
├── Code Analysis (LSP, Grep)
├── User Interaction (AskQuestion, PlanMode)
├── SubAgent Spawning (Agent Tool)
└── MCP External Tools (延迟加载)
```

### D. MCP — Model Context Protocol

Anthropic 发起的开放标准，作为工具连接的"USB-C 接口"：

```
Agent ←──MCP──→ GitHub
Agent ←──MCP──→ Database
Agent ←──MCP──→ Slack
Agent ←──MCP──→ 任何外部服务
```

**特点**: 标准化接口、延迟加载、关键字评分发现（而非穷举枚举）

### E. Memory Tiers (记忆层级)

```
┌─────────────────────────┐
│ Working Context (工作上下文) │ ← 当前对话/prompt，临时性
├─────────────────────────┤
│ Session State (会话状态)    │ ← 进度日志、git提交、任务状态，会话级
├─────────────────────────┤
│ Long-term Memory (长期记忆) │ ← 项目知识、模式、配置，跨会话持久
└─────────────────────────┘
```

### F. Doom Loop Detection (死循环检测)

Agent 反复尝试相同操作而无进展的检测机制：

- 跟踪每个文件的编辑次数
- 检测重复的工具调用模式
- 注入 "reconsider" 提示，引导 Agent 换一种方式

### G. Defense in Depth (纵深防御)

五层独立安全层，任何单层都不被单独依赖：

```
Layer 1: Prompt Guardrails      (提示词护栏)
Layer 2: Schema Restrictions    (Schema 级限制)
Layer 3: Runtime Approval       (运行时审批)
Layer 4: Tool Validation        (工具级校验)
Layer 5: Lifecycle Hooks        (生命周期钩子)
```

### H. Scaffolding vs Harness

| 维度 | Scaffolding | Harness |
|------|-------------|---------|
| **时机** | 首次 prompt 之前 | Agent 运行中 |
| **目的** | 组装 Agent | 运行和管理 Agent |
| **产出** | System Prompt、工具配置 | 工具调用结果、状态变更 |
| **频率** | 每次会话一次 | 持续循环 |

### I. Initializer-Executor Split (初始化-执行分离)

Anthropic 推荐的模式：

```
Session 1 (Initializer)          Session 2+ (Executor)
─────────────────────          ──────────────────────
• 搭建环境                      • 读取状态
• 建立脚本                      • 增量推进
• 创建 progress file            • 更新 progress
• 建立基线 commit               • 提交变更
• 生成 feature list             • 勾选完成项
```

### J. Middleware Composition (中间件组合)

LangChain 风格的可组合中间件：

```
Agent Request
    ↓
LocalContextMiddleware        ← 映射本地上下文
    ↓
LoopDetectionMiddleware       ← 检测死循环
    ↓
ReasoningSandwichMiddleware   ← 分配推理资源
    ↓
PreCompletionChecklistMiddleware ← 完成前检查
    ↓
Agent Response
```

## 下一步

→ 阅读 [02-architecture.md](./02-architecture.md) 深入了解完整架构
