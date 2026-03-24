# Dapr Agents、Layotto 与 Agent Harness 生态全景

## 一、Dapr Agents：Multi-Runtime 正式拥抱 Agent

### 基本信息

| 维度 | 信息 |
|------|------|
| **仓库** | github.com/dapr/dapr-agents |
| **GA 发布** | v1.0.0, 2026-03-19 (KubeCon Europe 2026 正式宣布) |
| **Stars** | 630 (父项目 dapr/dapr 34K+) |
| **License** | Apache 2.0 |
| **语言** | Python 3.11+ |
| **核心依赖** | dapr==1.17.3, openai>=1.75.0, mcp>=1.26.0 |
| **起源** | Floki (Microsoft Security AI), 由 Roberto Rodriguez 创建后合入 Dapr |
| **合作** | NVIDIA (Rodriguez 现已加入 NVIDIA)、Diagrid |

### Dapr Agents 与 Agent Harness 的关系

**Dapr Agents 不是一个独立的 Agent Harness，而是 Agent Harness 的分布式基础设施层。**

```
┌────────────────────────────────────────────────────────┐
│                  Agent Harness 层级                      │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │ 完整 Harness (DeerFlow, Claude Code)              │  │
│  │ Prompt管理 + 上下文工程 + 生命周期 + 安全纵深      │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ Agent Framework (LangChain, CrewAI)               │  │
│  │ 推理抽象 + 工具链 + Agent 模式                     │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ 分布式 Agent Runtime (Dapr Agents) ← 这一层       │  │
│  │ 状态持久 + 消息路由 + 服务发现 + 韧性 + 可观测     │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ 沙箱 / 执行环境 (E2B, Docker, K8s)               │  │
│  │ 进程隔离 + 资源限制                               │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘

Mark Fussell (Dapr 维护者):
"Many agent frameworks focus on logic alone.
 Dapr Agents delivers the infrastructure that keeps agents
 reliable through failures, timeouts and crashes."

Yaron Schneider (Dapr CTO):
"Dapr is becoming the resilience layer for AI systems."
```

**关键洞察**：DeerFlow 底层用 LangGraph (Framework 层)；如果 DeerFlow 的 Agent 需要跨节点分布式运行，它就需要 Dapr Agents 这样的 Runtime 层。**Harness 和 Runtime 不冲突，它们是不同层级。**

### Building Blocks → Agent 需求映射

这是 Dapr Agents 最核心的架构思想——**不造新轮子，复用已有的 Building Blocks**：

```
┌────────────────────────────────────────────────────────────┐
│          Dapr Building Block → Agent 能力映射               │
│                                                            │
│  State Management ──────────→ Agent Memory                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ ConversationDaprStateMemory                          │  │
│  │ 使用 Dapr State API → 28+ 后端 (Redis, PG, Cosmos..)│  │
│  │ 切换后端只需改 YAML 配置，Agent 代码零修改            │  │
│  │ 另有: ListMemory (开发), VectorMemory (语义检索)     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Pub/Sub ───────────────────→ 事件驱动 Agent 协作          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ @message_router 装饰器注册 topic                     │  │
│  │ CloudEvent 驱动的松耦合工作流                         │  │
│  │ Agent 独立伸缩，消息路由和验证由 Dapr 管理            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Virtual Actors ────────────→ 有状态 Agent 实例            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 每个 Agent = 一个 Virtual Actor                      │  │
│  │ 线程安全 + 原生分布式                                │  │
│  │ Scale-to-Zero: 不用时回收,保留状态,~50ms 重新激活    │  │
│  │ 单机可运行数千个 Agent (虚拟 Actor 的优势)            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Service Invocation ────────→ Agent-to-Agent 调用          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ agent_to_tool(): 将其他 Agent (包括非 Dapr Agent)    │  │
│  │ 暴露为当前 Agent 的工具                               │  │
│  │ 内置服务发现 + 错误处理 + 分布式追踪                  │  │
│  │ 支持 mDNS, K8s DNS, Consul, AWS Cloud Map            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Workflows ─────────────────→ 持久化 Agent 管道           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ DurableAgent (v1.0 核心类)                           │  │
│  │ 确定性执行 + 检查点 + 自动重试                        │  │
│  │ 工作流状态在重启和故障后自动恢复                       │  │
│  │ 两种编排: Workflow-based (确定性) / PubSub (动态)     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Configuration ─────────────→ Agent 热更新配置             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ v0.13.0 新增: 通过 Dapr Configuration Store          │  │
│  │ 支持 Agent 运行时热重载配置                           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Secrets Management ────────→ Agent 凭证管理              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 集成 Vault, K8s Secrets, AWS/Azure/GCP               │  │
│  │ Agent 不直接接触 API Key                              │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### MCP + A2A 支持

```
MCP (Model Context Protocol): ✅ 原生支持
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• pyproject.toml: "mcp>=1.26.0,<2.0.0"
• 3 种传输: stdio / SSE / Streamable HTTP
• 官方示例:
  ├── 06-agent-mcp-client-stdio
  ├── 06-agent-mcp-client-sse
  ├── 06-agent-mcp-client-streamablehttp
  └── 07-data-agent-mcp-chainlit

A2A (Agent-to-Agent Protocol): ⚡ Dapr 作为 A2A 的基础设施层
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
不直接实现 A2A 协议，而是填充 A2A 留给实现者的基础设施:

A2A 需求              Dapr 提供
───────              ─────────
Transport Security   mTLS (Sentry 自动签发 + SPIFFE ID)
Authentication       OAuth2/OIDC 中间件 + mTLS + API Token
Authorization        Sidecar 级 ACL (请求到达 Agent 前拦截)
Observability        OpenTelemetry 原生集成
Resiliency           声明式 retry / timeout / circuit breaker
Service Discovery    app-id 调用 + 可插拔 name resolution
Rate Limiting        app-max-concurrency + HTTP 限流中间件

使用方式: 在现有 A2A Agent 旁部署 Dapr Sidecar → 零代码修改
```

### Dapr Agents 支持的 7 种 Agent 模式

```
1. Augmented LLM       — 带工具的增强 LLM 调用
2. Durable Agent       — 持久化、可恢复的自主 Agent
3. Prompt Chaining     — 串联多个 LLM 调用
4. Evaluator-Optimizer — 生成+评估循环
5. Parallelization     — 并行工具/Agent 执行
6. Routing             — 动态路由到不同 Agent
7. Orchestrator-Workers — 编排者-工作者模式
```

---

## 二、Layotto 社区：停滞与潜在路径

### 现状：一个已关闭的 Issue

```
仓库: mosn/layotto
Stars: 853
状态: 低活跃维护 (主要是依赖更新)

唯一 AI 相关讨论: Issue #1090 "Support MCP"
├── 创建: 2024-12-12 by CrazyHZM
├── 关闭: 2025-05-10 (stale bot 自动关闭)
├── 结果: 无代码、无 RFC、无路线图
└── 核心观点: "Layotto 作为 MCP Server 暴露 Building Block API 给 Agent"
```

社区讨论中的关键洞察（中文原文）:

> "如果 Layotto 这种运行时能够支持 MCP，感觉是很强大的呀，比如可以把现有的服务 API 都集成到大模型 Agent 中？"
>
> "是的，Layotto 作为一个 MCP Server 把现有的 api 作为 tools 给 agent 用，真的是完美的适配了"

**社区看到了方向，但没有执行。**

### 对比：Dapr 已经做到的

| 维度 | Dapr Agents | Layotto |
|------|-------------|---------|
| Agent 支持 | GA v1.0 (2026-03-19) | 一个已关闭的 Issue |
| MCP 集成 | 原生支持 (3 种传输) | 社区建议但未采纳 |
| LLM 集成 | 多模型 + Conversation Building Block | 无 |
| Virtual Actor for Agent | 单机数千 Agent | 无 |
| A2A 协议 | Dapr 作为基础设施层 | 无 |
| NVIDIA 合作 | 核心贡献者 | 无 |
| 社区活跃度 | 持续发版 (v0.8→v1.0) | 依赖更新为主 |

### 蚂蚁的 AI 投入与 Layotto 的缺位

```
蚂蚁在 AI Agent 领域非常活跃:
├── 蚂蚁阿福: 健康 AI Agent, 月活 1500 万+
├── 蚂蚁数科: 金融 Agent 矩阵, 100% 国有银行覆盖
├── CodeFuse: 代码生成大模型
└── 但这些都与 Layotto 没有交集

蚂蚁的 AI Agent 走的是垂直领域应用路线,
不是通过 Layotto 基础设施层来支持。
```

### Layotto 如果要追赶的潜在路径

```
路径 1: Building Block API → MCP Server
├── 将 State/PubSub/Lock/Config API 暴露为 MCP 工具
├── Agent 通过 MCP 协议调用 Layotto 的分布式能力
├── 复用已有的 28+ 组件实现
└── 最小改动,最大价值

路径 2: Sidecar → Agent 沙箱
├── 复用 Sidecar 进程隔离能力
├── 为 Agent 提供安全的执行环境
└── 扩展为 Agent Runtime (类似 Dapr Agents)

路径 3: MOSN 流量管理 → Agent Gateway
├── 利用 MOSN 的 L4/L7 流量管理
├── 构建 Agent 间通信的网关 (A2A)
└── 类似 Higress 的 MCP Server Gateway 方向

路径 4: WASM → Agent 插件隔离
├── Layotto 已有 WASM 执行能力
├── 可用于 Agent Skill/Plugin 的隔离执行
└── 比 Docker 更轻量的沙箱
```

---

## 三、Agent Harness 生态全景

### 分类框架

```
                    ┌──────────────────────────┐
                    │    完整 Agent Harness     │
                    │                          │
                    │  DeerFlow  Claude Code   │
                    │  OpenHands  OpenClaw     │
                    │  Goose (部分)            │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Agent Framework          │
                    │                          │
                    │  LangChain  CrewAI       │
                    │  AutoGen    Google ADK   │
                    │  OpenAI Agents SDK       │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Agent Runtime /          │
                    │  Infrastructure           │
                    │                          │
                    │  Dapr Agents  E2B        │
                    │  NVIDIA OpenShell        │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  Agent Application       │
                    │  (使用上述层构建)          │
                    │                          │
                    │  Manus  bolt.new  v0     │
                    │  Lovable  Devin          │
                    └──────────────────────────┘
```

### 真正的 Agent Harness 实现

| 项目 | Stars | 类型 | 核心特色 | Harness 完整度 |
|------|-------|------|---------|---------------|
| **Claude Code** | N/A | 终端 Harness | 5层安全, SubAgent, Skill/Hook, 记忆 | ★★★★★ |
| **DeerFlow 2.0** | 37K+ | SuperAgent Harness | 14阶段中间件, Docker隔离, 后端轮询 | ★★★★★ |
| **OpenHands V1** | 64K+ | 云端 Coding Harness | 事件流架构, 远程Runtime, ICLR 论文 | ★★★★☆ |
| **OpenClaw** | 210K+ | 个人 Agent Harness | Hub-Spoke, 多渠道, 史上增长最快 | ★★★★☆ |
| **Cursor 2.0** | N/A | IDE Harness (闭源) | Composer MoE, 并行Agent, $2B ARR | ★★★★☆ |
| **Goose** | 30K+ | 本地 Agent | MCP原生, AAIF 捐赠 | ★★★☆☆ |
| **Warp 2.0** | N/A | 终端 Harness (闭源) | Oz 编排, PTY 会话, WARP.md | ★★★☆☆ |
| **Cline/Roo** | 高 | IDE Agent | Plan+Act, CLI 2.0, 透明操作 | ★★★☆☆ |
| **KWeaver** | 293 | 企业决策 Harness | Harness-first, 知识网络 | ★★☆☆☆ |

### 基础设施层 (支撑 Harness 但本身不是 Harness)

| 项目 | 角色 | 说明 |
|------|------|------|
| **Dapr Agents** | 分布式 Agent Runtime | Building Blocks 映射到 Agent 需求 |
| **E2B** | 沙箱基础设施 | Firecracker microVM, <200ms 启动, ~50% Fortune 500 |
| **NVIDIA OpenShell** | Agent 治理运行时 | Out-of-process, GTC 2026 发布 |
| **Microsoft Agent Framework** | 统一 Agent 框架 | AutoGen + Semantic Kernel 合并 |

### 重要理论贡献 (影响 Harness 设计但非通用实现)

| 项目 | 贡献 |
|------|------|
| **SWE-agent** (Princeton) | ACI (Agent-Computer Interface) 概念 |
| **AutoCodeRover** | 上下文感知的自主程序改进 |
| **Agentless** | 证明简单方法有时优于复杂 Agent |

---

## 四、协议层正在标准化

```
Bilgin Ibryam (Multi-Runtime 提出者) 的 Agent Communication Protocols Landscape:

协议              Stars    职责                       Multi-Runtime 类比
─────             ─────    ─────                      ──────────────
MCP (Anthropic)   100K+    Agent ↔ 工具/数据          ≈ Dapr Bindings
A2A (Google→LF)   20K+     Agent ↔ Agent             ≈ Dapr Service Invocation
AG-UI             4K+      Agent ↔ 前端              ≈ Dapr SDK/Client API

三个协议都已纳入 Linux Foundation Agentic AI Foundation (AAIF)
共建者: OpenAI, Anthropic, Google, Microsoft, AWS

Ibryam 的洞见:
MCP = Context-Oriented (连接外部资源)
A2A = Inter-Agent (Agent 间协作)
Dapr = 两者的基础设施层 (不是协议本身,而是让协议可靠运行)
```

---

## 五、从你的经验看整个版图

```
你参与过的项目         在 Agent 时代的位置           状态
━━━━━━━━━━━━         ━━━━━━━━━━━━━━━━━━━          ━━━━
Dapr                 → Dapr Agents (GA v1.0)       ✅ 已完成转型
Layotto              → (Issue #1090 已关闭)         ❌ 停滞
Capa                 → (无 Agent 相关动态)          ❌ 停滞
Dubbo                → (部分模型推理集成探索)        ⚠️ 边缘探索

启示:
├── Dapr 因为有 Diagrid + NVIDIA 的商业驱动力,转型最成功
├── Layotto/Capa 缺少商业驱动力,在 Agent 浪潮中掉队
├── 但 Layotto 的技术基础 (Sidecar + WASM + MOSN) 完全可以支撑转型
└── 关键瓶颈不是技术,是社区和商业投入
```

### 如果你要推动 Layotto 进入 Agent Harness 领域

```
最小可行路径 (MVP):

Step 1: Layotto as MCP Server
├── 将 State/PubSub/Config Building Block API 暴露为 MCP 工具
├── Agent 通过标准 MCP 协议调用 Layotto 的分布式能力
├── 工作量: 中等 (MCP SDK + API 适配层)
└── 价值: 立即可用,任何 MCP 兼容 Agent 都能连接

Step 2: Agent Sidecar Mode
├── 在 Sidecar 模式中增加 Agent Runtime 功能
├── 参考 Dapr Agents 的 DurableAgent 设计
├── 复用 Layotto 已有的 State/PubSub 组件
└── 价值: Agent 获得完整的分布式能力

Step 3: WASM Agent Skills
├── 利用 Layotto 的 WASM 能力运行 Agent 技能
├── 比 Docker 更轻量的隔离执行环境
└── 价值: 差异化优势 (Dapr 没有 WASM)

Step 4: MOSN 作为 Agent Gateway
├── 基于 MOSN 的 L7 能力构建 Agent 路由
├── 实现 A2A 协议的基础设施支持
└── 价值: Service Mesh + Agent Mesh 融合 (Layotto 的原始愿景升级版)
```
