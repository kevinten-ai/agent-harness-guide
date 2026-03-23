# Multi-Runtime 与 Agent Harness：同一哲学的两个时代

## 核心论点

> **Multi-Runtime 之于分布式系统，正如 Agent Harness 之于 AI Agent。**
> 两者遵循同一架构哲学：**将核心逻辑与基础设施关注点分离，通过伴生运行时提供可插拔的能力。**

这不是巧合——它们解决的是同构问题：一个"大脑"（业务逻辑/LLM）需要很多"身体"能力（分布式原语/工具执行），而这些能力不应该嵌入大脑本身。

---

## 概念映射总览

```
Multi-Runtime (2020)              Agent Harness (2025)
═══════════════════               ═══════════════════

Micrologic (业务逻辑)        ←→   LLM (推理能力)
Mecha (伴生运行时)           ←→   Harness (Agent 基础设施)

Building Blocks API          ←→   Tool Registry / MCP
Components (可插拔实现)      ←→   MCP Servers / 插件
State Management             ←→   Memory System (三层记忆)
Bindings (外部集成)          ←→   Tool Integration (工具集成)
Middleware Pipeline          ←→   Middleware Pipeline (14阶段)
Pub/Sub (事件驱动)           ←→   Lifecycle Hooks (事件钩子)
Configuration                ←→   config.yaml / CLAUDE.md
Observability (可观测性)     ←→   Agent Tracing (行为追踪)
Resiliency (韧性)            ←→   Doom Loop / 重试 / 降级
Sidecar Pattern              ←→   Harness 包裹 LLM
Service Mesh + Runtime       ←→   Safety Layer + Harness

Dapr                         ←→   DeerFlow / Claude Code
Layotto                      ←→   (Mesh+Runtime 融合)
Capa                         ←→   Claude Agent SDK (Rich SDK)
```

---

## 架构同构分析

### 一、核心分离：Micrologic ↔ LLM

**Bilgin Ibryam 的洞见 (2020)**:

```
传统方式: 业务逻辑 + 分布式关注点 = 一个胖运行时
                                     (Spring Cloud)

Multi-Runtime: 业务逻辑 (Micrologic) + 伴生运行时 (Mecha)
               只关心领域逻辑          处理所有分布式原语
```

**Agent Harness 的同构 (2025)**:

```
传统方式: LLM + 简单 API 调用 = 一个无能力的 chatbot

Agent Harness: LLM (推理) + Harness (基础设施)
               只关心思考     处理所有执行/记忆/安全
```

**深层对应**:

| Micrologic 特征 | LLM 特征 |
|-----------------|----------|
| 只含业务逻辑 | 只负责推理和决策 |
| 剥离了所有分布式关注点 | 剥离了所有执行关注点 |
| 通过标准协议与 Mecha 通信 (HTTP/gRPC) | 通过标准格式与 Harness 通信 (tool_call JSON) |
| 多语言支持 (任何能说 HTTP 的语言) | 多模型支持 (任何能输出 tool_call 的 LLM) |
| 不能独立运行 (需要 Mecha) | 不能独立执行 (需要 Harness) |

### 二、Building Blocks API ↔ Tool Registry / MCP

**Dapr Building Blocks**:

```
Application ──HTTP/gRPC──→ Dapr Sidecar ──Component──→ Redis/Kafka/S3/...
                           │
                           │  Building Block = 抽象 API
                           │  Component = 具体实现
                           │  配置切换，代码不变
```

**Agent Harness Tool System**:

```
LLM ──tool_call JSON──→ Tool Registry ──Handler/MCP──→ GitHub/DB/Shell/...
                         │
                         │  Tool Schema = 抽象 API (JSON Schema)
                         │  MCP Server = 具体实现
                         │  配置切换，模型不感知
```

**对应表**:

| Dapr Building Block | Agent Harness 对应 |
|--------------------|--------------------|
| State Management | Memory System (state store) |
| Pub/Sub | Lifecycle Hooks / Event System |
| Service Invocation | SubAgent Spawning (Agent-to-Agent) |
| Bindings (Input/Output) | Tool Integration (读/写外部系统) |
| Actors | Stateful Agent (有状态的 Agent 实例) |
| Configuration | CLAUDE.md / config.yaml |
| Secrets | 敏感数据保护层 |
| Workflows | Agent Task Pipeline / Plan Mode |
| Distributed Lock | 并发控制 (SubagentLimit) |
| Cryptography | 安全层 (Runtime Approval) |
| **Conversation (新)** | **LLM 抽象 (Agent Loop 核心)** |
| **Agentic AI (新)** | **Agent 工作流 (直接对应!)** |

**关键洞察**: Dapr 在 2025 年添加了 Conversation 和 Agentic AI Building Blocks，**这正是 Multi-Runtime 向 Agent Harness 融合的证据**。

### 三、Component Pluggability ↔ MCP 可插拔性

```
Dapr 组件模型:                     MCP 协议:
──────────────                     ─────────
接口: Building Block API           接口: MCP Protocol
实现: Component (Redis/PG/...)     实现: MCP Server (GitHub/Supabase/...)
配置: YAML 文件                    配置: .mcp.json / extensions_config.json
切换: 改配置不改代码               切换: 改配置不改 Agent

Dapr:                              MCP:
apiVersion: dapr.io/v1alpha1       {
kind: Component                      "github": {
metadata:                              "type": "stdio",
  name: statestore                     "command": "npx",
spec:                                  "args": ["@mcp/server-github"]
  type: state.redis                  }
  metadata:                        }
  - name: redisHost
    value: localhost:6379

换成 PostgreSQL?                   换成 GitLab?
只改 type: state.postgresql        只改 MCP server 配置
代码零修改                         Agent 零感知
```

**Capa 的独特视角**:

```
Capa (你参与的项目):
├── 问题: 企业不能一步迁到 Sidecar
├── 方案: Rich SDK 作为过渡桥梁
├── 设计: Standard API + SPI 实现
└── 价值: 渐进式演进

Claude Agent SDK (类似角色):
├── 问题: 不是所有场景需要完整 Harness
├── 方案: SDK 级别的 Harness 能力
├── 设计: Standard API + 可自定义
└── 价值: 从 SDK 到完整 Harness 的渐进路径

Capa           ≈  Claude Agent SDK
Dapr Sidecar   ≈  DeerFlow (完整 Harness)
Layotto        ≈  Claude Code (Mesh + Runtime 融合)
```

### 四、Middleware Pipeline ↔ Middleware Pipeline

这是最直接的 1:1 映射：

```
Dapr Middleware Pipeline:              DeerFlow 14-Stage Pipeline:
━━━━━━━━━━━━━━━━━━━━━━━                ━━━━━━━━━━━━━━━━━━━━━━━━━━

Request                                Request
  │                                      │
  ▼                                      ▼
[OAuth2 Middleware]                    [ThreadData Middleware]
  │                                      │
  ▼                                      ▼
[Rate Limiting]                        [Sandbox Middleware]
  │                                      │
  ▼                                      ▼
[Bearer Auth]                          [Guardrail Middleware]
  │                                      │
  ▼                                      ▼
[OPA Policy]                           [Summarization Middleware]
  │                                      │
  ▼                                      ▼
[Tracing]                              [Memory Middleware]
  │                                      │
  ▼                                      ▼
Application Logic                      [LoopDetection Middleware]
                                         │
                                         ▼
                                       LLM 推理

共同特征:
├── 有序执行 (顺序敏感)
├── 可条件包含 (按配置动态组装)
├── 职责单一 (每层解决一个关注点)
├── 可独立开发和测试
└── 链式处理 (request → middleware chain → core → response chain)
```

### 五、State Management ↔ Memory System

```
Dapr State Management:               Agent Memory System:
━━━━━━━━━━━━━━━━━━━━━                ━━━━━━━━━━━━━━━━━━━━

API:                                  API:
  SaveState(key, value)                 Save(fact, confidence)
  GetState(key)                         Retrieve(query, top_k)
  DeleteState(key)                      Delete(fact_id)

后端可插拔:                            后端可插拔:
  Redis / PostgreSQL /                  JSON File / SQLite /
  Cosmos DB / DynamoDB /                PostgreSQL / Vector DB /
  MongoDB / ...                         ...

一致性模型:                            一致性模型:
  强一致 / 最终一致                      置信度评分 (0-1)
  乐观并发 (ETag)                       事实去重 + 原子写入

TTL:                                  TTL:
  State 可设置过期时间                   记忆衰减 + 定期清理

事务:                                 事务:
  Multi-item transactions              Batch fact update
                                       (newFacts + factsToRemove)

持久化层级:                            持久化层级:
  Memory / Disk / Remote DB            Working / Session / Long-term
```

### 六、Resiliency ↔ Agent 韧性

```
Dapr Resiliency:                      Agent Harness 韧性:
━━━━━━━━━━━━━━━                       ━━━━━━━━━━━━━━━━━━

Circuit Breaker                       Doom Loop Detection
(连续失败 N 次 → 熔断)                 (同一操作重复 N 次 → 强制停止)

Retry (指数退避)                       Tool Retry (参数调整重试)

Timeout                               Tool Timeout (2min default)

Bulkhead (隔离)                        SubAgent Isolation (Schema 隔离)

Fallback                              Graceful Degradation (降级)

声明式配置:                            声明式配置:
resiliencySpec:                        loop_detection:
  policies:                              window_size: 20
    retries:                             warn_threshold: 3
      retryForever:                      stop_threshold: 5
        maxRetries: -1
    circuitBreakers:
      simpleCB:
        maxRequests: 1
        trip: consecutiveFailures > 5
```

---

## 四象限分析: Ibryam 的分布式需求 → Agent 需求

Bilgin Ibryam 识别的分布式应用四类需求，在 Agent 世界中有精确对应：

```
┌─────────────────────────────┬──────────────────────────────┐
│      Lifecycle (生命周期)     │      Networking (网络)        │
│                             │                              │
│ 分布式:                      │ 分布式:                       │
│ • 容器化打包                 │ • 服务发现                    │
│ • 自动部署恢复               │ • 负载均衡 + 路由             │
│ • 健康监控 + 自愈            │ • 熔断 + 重试 + 超时          │
│ • 弹性伸缩                   │ • 可观测性 (tracing)          │
│ • 配置管理                   │ • 安全 (mTLS)                │
│                             │ • 多种交互模式 (req/resp, pub/sub)│
│ Agent:                      │                              │
│ • Scaffolding (环境发现)     │ Agent:                       │
│ • Session 生命周期管理       │ • SubAgent 发现 + 路由        │
│ • 上下文自动压缩 (自愈)      │ • 并发控制 + 负载均衡         │
│ • SubAgent 弹性产生          │ • Doom Loop + 重试 + 超时     │
│ • config.yaml / CLAUDE.md   │ • Agent Tracing               │
│                             │ • 5 层安全                    │
│                             │ • 人机交互 / Agent-Agent / Event│
├─────────────────────────────┼──────────────────────────────┤
│      State (状态)            │      Binding (集成)           │
│                             │                              │
│ 分布式:                      │ 分布式:                       │
│ • 工作流编排                 │ • 外部系统连接器              │
│ • 分布式单例                 │ • 协议转换                    │
│ • 幂等消息处理               │ • 消息格式转换                │
│ • 定时调度                   │ • 错误恢复                    │
│ • Saga 事务                  │ • 安全中介                    │
│ • 缓存                      │                              │
│                             │ Agent:                       │
│ Agent:                      │ • MCP Server (外部连接)       │
│ • Plan Mode (工作流)         │ • 协议适配 (stdio/SSE/HTTP)  │
│ • Agent Loop (单线程执行)    │ • 格式转换 (JSON Schema)     │
│ • 去重 (fact deduplication)  │ • 工具错误处理               │
│ • Cron/Scheduled tasks       │ • OAuth 安全认证             │
│ • Init-Executor (Saga 式)    │                              │
│ • Memory Cache (mtime 缓存)  │                              │
└─────────────────────────────┴──────────────────────────────┘
```

---

## 演进融合：两个世界正在合并

```
时间线:

2020  Bilgin Ibryam 提出 Multi-Runtime / Mecha 架构
      ↓
2021  Dapr 1.0 / Layotto 开源
      ↓
2022  Capa SDK 探索 Rich SDK 路径
      ↓
2023  MCP 协议提出 (Anthropic)
      ↓
2024  MCP 成为事实标准
      Dapr 成为 CNCF Graduated 项目
      ↓
2025.3  Dapr Agents 发布
        ├── Building Blocks 直接映射到 Agent 需求
        ├── State Management → Agent Memory
        ├── Pub/Sub → Agent Event-Driven Workflows
        ├── Service Invocation → Agent-to-Agent Calls
        └── Actors → Stateful Agent Instances
      ↓
2025.12  Agentic AI Foundation 成立 (Linux Foundation)
         MCP + A2A 纳入标准化
         OpenAI, Anthropic, Google, Microsoft, AWS 共建
      ↓
2026.2  DeerFlow 2.0 发布 ("SuperAgent Harness")
        Bilgin Ibryam 开始研究 Agent Communication Protocols
      ↓
2026.3  两个世界正式融合
        Multi-Runtime + Agent Harness = 同一架构哲学
```

**Bilgin Ibryam 现在的工作**:

```
Agent Communication Protocols Landscape (他正在绘制的图):

MCP  (Model Context Protocol)  — 工具集成协议 ≈ Dapr Bindings
A2A  (Agent-to-Agent)          — Agent 间通信 ≈ Dapr Service Invocation
AG-UI (Agent-to-UI)            — Agent 到界面 ≈ Dapr SDK/API

这三个协议 + Agent Harness = AI 时代的 Multi-Runtime
```

---

## 关键差异：不能忽略的区别

虽然同构性很强，但有重要差异：

```
维度              Multi-Runtime              Agent Harness
─────             ─────────────              ─────────────
核心逻辑          确定性 (业务代码)           概率性 (LLM 推理)
控制流            应用控制执行顺序            模型控制执行顺序 (Agent Loop)
失败模式          明确的异常/错误码           模糊的"不正确"输出
隔离方式          进程级 (Sidecar)            架构层级 (In-Process/Service)
通信协议          HTTP/gRPC (标准化)          tool_call JSON (模型特定)
上下文限制        无 (应用有完整状态)          有 (Token 窗口限制)
自愈方式          K8s 重启 + 健康检查         上下文压缩 + 记忆卸载
安全模型          mTLS + RBAC (标准)          纵深防御 (5 层, 部分启发式)
可预测性          高 (相同输入→相同输出)      低 (相同输入→不同输出)
调试方式          日志 + 链路追踪             Trace + 对话回放
```

**概率性是最核心的差异**：

```
Multi-Runtime:
  Application 调 StateStore.save(key, value)
  → 要么成功,要么失败,行为确定

Agent Harness:
  LLM 决定 "我应该调 Edit(file, old, new)"
  → 可能选错文件
  → 可能 old_string 不精确
  → 可能 new_string 有 bug
  → 可能重复调用 (Doom Loop)
  → 需要额外的验证/护栏层

因此 Agent Harness 多了:
├── Doom Loop Detection (Multi-Runtime 不需要)
├── Self-Critique Phase (Multi-Runtime 不需要)
├── Verification Layer (Multi-Runtime 不需要)
└── Context Compaction (Multi-Runtime 不需要)

这些都是为了应对 LLM 的概率性本质
```

---

## 对从业者的启示

### 从 Multi-Runtime 到 Agent Harness：你已有的知识

```
如果你理解 Dapr...              你已经理解了 Agent Harness 的:

Building Blocks API      →     工具抽象层 (Tool Registry)
Component YAML           →     MCP Server 配置
Middleware Pipeline      →     中间件管道 (DeerFlow 14 阶段)
State Store              →     记忆系统
Pub/Sub                  →     事件驱动的 Hook 系统
Service Invocation       →     SubAgent 调用
Actor Model              →     有状态的 Agent 实例
Resiliency Spec          →     韧性设计 (Doom Loop / Retry)
Observability            →     Agent Tracing
Sidecar ↔ App 边界      →     Harness ↔ App 边界 (DeerFlow 的架构测试)
```

### 需要额外学习的 (Multi-Runtime 没有的)

```
1. Agent Loop (模型自主控制流)
   Multi-Runtime 中应用控制流是确定的
   Agent 中 LLM 决定下一步做什么 — 这是全新的

2. Context Engineering (上下文工程)
   Multi-Runtime 没有 "上下文窗口" 的限制
   Agent 需要精心管理 Token 预算

3. 概率性输出的验证
   Multi-Runtime 用类型系统+测试保证正确性
   Agent 需要额外的验证层 (测试+Self-Critique+人审)

4. Prompt Engineering
   Multi-Runtime 的 API 调用是精确的
   Agent 的 System Prompt 是"模糊的 API 文档"
   工具 Schema 的 description 质量直接影响行为

5. Safety (纵深防御)
   Multi-Runtime 的安全是标准化的 (mTLS, RBAC)
   Agent 的安全需要启发式 + 多层冗余
```

### Capa 的启发：渐进式演进

```
Capa 的核心理念:
  不能一步到位 → Rich SDK 作为过渡 → 最终到 Sidecar

Agent Harness 的渐进路径:
  简单 API 调用
    → Claude Agent SDK (Rich SDK, 类似 Capa)
    → Claude Code (完整 Harness, 类似 Dapr)
    → DeerFlow (生产级 Harness, 类似 Dapr + Layotto)
    → 自建 Harness (深度定制, 类似企业自建 Mecha)

你在 Capa 上的经验直接适用:
├── Standard API 的设计理念
├── SPI 可插拔实现的模式
├── 渐进式迁移策略
└── SDK → Runtime 的演进路径
```

---

## 预测：下一步融合

```
2026-2027 趋势:

1. Dapr 将成为 Agent 基础设施的一部分
   Dapr Agents 已经开始,State/PubSub/Actors 直接服务 Agent
   未来 Dapr Sidecar = Agent Harness 的分布式层

2. MCP + A2A 将标准化为 Agent 的"Building Blocks"
   就像 Dapr 标准化了分布式原语的 API
   MCP 标准化工具集成,A2A 标准化 Agent 间通信

3. Harness 将分层
   ├── 本地 Harness (Claude Code 级) — 单机 Agent
   ├── 分布式 Harness (Dapr 级)     — 多 Agent 协作
   └── 平台 Harness (K8s 级)        — Agent 编排和治理

4. Multi-Runtime 的人会成为 Agent Harness 的核心贡献者
   因为他们已经理解了:
   ├── 关注点分离
   ├── 可插拔组件模型
   ├── 中间件管道
   ├── 状态管理抽象
   └── 韧性工程

   他们只需要额外学习:
   ├── LLM 的概率性
   ├── 上下文窗口管理
   └── Agent Loop 的控制流反转
```

---

## 总结

```
Multi-Runtime 的架构直觉:
"业务逻辑不应该关心分布式基础设施怎么工作"

Agent Harness 的架构直觉:
"LLM 不应该关心工具执行/记忆/安全怎么工作"

同一个直觉. 不同的时代. 相同的解法.

Micrologic + Mecha = Multi-Runtime Application
LLM + Harness = AI Agent

你在 Dapr/Layotto/Capa 上积累的架构思维,
是理解和构建 Agent Harness 的最大优势。
```
