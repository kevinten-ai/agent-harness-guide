# 主流框架对比

## 框架全景图

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent 框架生态 (2025-2026)                 │
│                                                             │
│  ┌─── 平台原生 ─────────┐  ┌─── 开源框架 ─────────────┐    │
│  │ Claude Code (Anthropic)│  │ LangChain / LangGraph    │    │
│  │ Codex CLI (OpenAI)    │  │ CrewAI                   │    │
│  │ ADK (Google)          │  │ AutoGen (Microsoft)      │    │
│  │ Agent SDK (Anthropic) │  │ Phidata                  │    │
│  │ Agents SDK (OpenAI)   │  │ DSPy                     │    │
│  └───────────────────────┘  └──────────────────────────┘    │
│                                                             │
│  ┌─── 标准协议 ─────────┐  ┌─── 终端 Agent ────────────┐   │
│  │ MCP (工具连接)        │  │ Claude Code               │    │
│  │ A2A (Agent 间通信)    │  │ Codex CLI                 │    │
│  │                       │  │ Aider                     │    │
│  └───────────────────────┘  │ OpenDev                   │    │
│                              └──────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## 核心框架详细对比

### Claude Code (Anthropic)

```
定位: 终端 Agent + Agent Harness 参考实现
模型: Claude 系列

架构亮点:
├── 完整的 Agent Loop (Think → Act → Verify)
├── 丰富的内置工具 (Read, Write, Edit, Bash, Glob, Grep...)
├── MCP 工具延迟加载 (ToolSearch)
├── 多层安全 (5 层纵深防御)
├── SubAgent 产生 (Agent 工具)
├── 上下文自动压缩
├── 持久记忆系统 (MEMORY.md)
├── Skill 系统 (可扩展能力)
├── Hook 系统 (生命周期事件)
├── Plan Mode (只读规划)
└── Plugin 生态

Harness 工程价值:
• 它本身就是一个 Agent Harness 的完整实现
• 可作为学习 Harness 工程的"活教材"
• 开源,可研究其内部架构
```

### Codex CLI (OpenAI)

```
定位: 终端 Agent
模型: OpenAI 系列

架构亮点:
├── 迭代式 Agent Loop
│   user input → prompt → LLM → tool_call → execute → append → re-query
├── 沙箱化 Shell 执行
├── 文件 I/O 工具
├── 测试套件集成
├── Linter/Type Checker 集成
└── "Harness Engineering" 方法论

OpenAI 的 Harness Engineering 三支柱:
1. Context Engineering: 持续增强的知识库
2. Architectural Constraints: LLM Agent + 确定性 Linter
3. Entropy Management: "垃圾回收" Agent 查找不一致
```

### Claude Agent SDK (Anthropic)

```
定位: 通用 Agent 开发 SDK
模型: Claude 系列

特点:
├── 轻量级 Python SDK
├── 内置上下文压缩
├── Tool-use 能力
├── 可自定义 Agent Loop
├── 适合构建自定义 Harness
└── API 级别的细粒度控制

适用: 需要构建自定义 Agent 应用的开发者
```

### OpenAI Agents SDK

```
定位: 轻量级 Agent 开发框架
模型: OpenAI 系列
GitHub Stars: 19K+ (2025.3)

特点:
├── Agent Loop 管理
├── Python-first 设计
├── 工具定义和调用
├── Agent 之间的 Handoff
├── Guardrails (输入/输出验证)
├── Tracing 和可观测性
└── 支持 MCP

适用: 快速构建基于 OpenAI 的 Agent 应用
```

### LangChain / LangGraph

```
定位: 模型无关的 Agent 开发框架
模型: 支持任意 LLM

LangChain 核心:
├── Chain: 线性工作流
├── Agent: 自主决策
├── Tools: 工具集成
├── Memory: 记忆管理
└── Retrieval: 检索增强

LangGraph 核心:
├── 有向图工作流 (DAG)
├── Node = Agent/Function
├── Edge = 转换/条件
├── State = 流转数据
├── Persistence = 内置持久化
└── Streaming = 流式输出

Harness 工程贡献:
• Middleware Composition 模式
• Reasoning Sandwich 发现
• Terminal Bench 2.0 排名提升 (纯 Harness 改进)
```

### CrewAI

```
定位: 多 Agent 角色扮演框架
模型: 支持任意 LLM
GitHub Stars: 44K+ (2025.3)

特点:
├── Role-based Agents (基于角色的 Agent)
│   每个 Agent 有:
│   ├── Role (角色)
│   ├── Goal (目标)
│   ├── Backstory (背景)
│   └── Tools (工具)
├── Task 定义和分配
├── Process 流程 (Sequential / Hierarchical)
├── Memory (短期/长期/实体记忆)
└── 人类反馈集成

适用: 模拟团队协作的多 Agent 场景
```

### Google ADK (Agent Development Kit)

```
定位: 代码优先的 Agent 开发框架
模型: Gemini 系列 (也支持其他模型)

特点:
├── 代码优先 (非低代码)
├── Python 框架
├── Multi-Agent 编排
├── 工具集成
├── 评估框架
└── Cloud 部署 (Vertex AI)

发布: Google Cloud NEXT 2025
```

## 对比矩阵

```
维度              Claude Code   Codex CLI   LangChain   CrewAI   Agent SDK
──────            ──────────    ─────────   ─────────   ──────   ─────────
定位              终端 Agent    终端 Agent  开发框架    多Agent  开发 SDK
模型锁定          Claude        OpenAI      无关        无关     Claude
工具丰富度        ★★★★★        ★★★★       ★★★★       ★★★     ★★★
安全层数          5 层          3 层        自定义      基础     自定义
上下文管理        ★★★★★        ★★★★       ★★★★       ★★★     ★★★★
记忆系统          3 层          2 层        可插拔      3 类     自定义
MCP 支持          ✓             ✓           ✓           ✓        ✓
多 Agent          SubAgent      有限        LangGraph   核心     自定义
可扩展性          Plugin+Skill  配置        自由        中等     完全
学习曲线          低            低          中-高       中       中-高
开源              ✓             ✓           ✓           ✓        ✓
```

## 标准协议

### MCP (Model Context Protocol)

```
发起: Anthropic (2024)
定位: 工具连接的"USB-C 接口"
采用: Microsoft, Replit, 大量开源项目

作用: 标准化 Agent 与外部服务的连接方式
传输: stdio, SSE, HTTP, WebSocket
状态: 事实标准 (de facto standard)
```

### A2A (Agent-to-Agent Protocol)

```
发起: Google (2025)
定位: Agent 间通信标准
作用: 让不同框架的 Agent 能互相通信和协作
状态: 早期阶段
```

## 如何选择

```
场景                              推荐
────                              ────
快速上手终端 Agent                → Claude Code
学习 Harness 工程                → Claude Code (作为实例研究)
构建自定义 Agent 应用             → Claude Agent SDK / OpenAI Agents SDK
模型无关的灵活框架                → LangChain / LangGraph
多 Agent 角色协作                 → CrewAI
Google 生态集成                   → Google ADK
最大控制力                        → 自建 Harness (参考模式)
```

## 趋势

```
2024: MCP 发布,工具连接标准化
2025: "Harness Engineering" 概念普及
2026:
├── Harness 可靠性成为核心竞争力
├── 框架整合 (MCP 成为事实标准)
├── 可观测性成为刚需
├── 安全和合规要求提高
└── 从"能工作"到"可靠地工作"
```
