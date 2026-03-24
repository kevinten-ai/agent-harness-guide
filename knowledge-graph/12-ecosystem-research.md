# Agent Harness 生态研究：Layotto AI 动态 + 全球 Agent Harness 实现

> 研究日期: 2026-03-24
> 范围: (1) Layotto/MOSN 社区的 AI/Agent 响应 (2) DeerFlow/Claude Code 之外的 Agent Harness 实现

---

## 第一部分：Layotto 社区的 AI/Agent 活动

### 1.1 核心发现：Issue #1090 — Support MCP (Model Context Protocol)

Layotto 社区唯一与 AI Agent 直接相关的讨论是 [Issue #1090](https://github.com/mosn/layotto/issues/1090)，由贡献者 CrazyHZM 于 2024-12-12 创建，2025-05-10 被 stale bot 自动关闭。

**Issue 内容:**
> "Layotto has shown significant advantages in multi-language. In the AI scenario, MCP, as a standardized protocol that can connect the external data sources and tools of LLM applications, can be considered as a reference to realize the new prospect and power of Layotto in the AI scenario."

**社区讨论（中文原文 + 翻译）:**

- 一位社区成员提议："如果 layotto 这种运行时能够支持 MCP 的话，感觉是很强大的呀，比如可以把现有的服务 API 都集成到大模型 Agent 中？"
- 回复认同："是的呀，Layotto 作为一个 MCP Server 把现有的 api 作为 tools 给 agent 用，真的是完美的适配了"
- 另一位成员指出 Higress（阿里巴巴）已有 MCP Server 实现，且 [dapr-agents](https://github.com/dapr/dapr-agents) 也提到了 MCP 规划

**关键洞察:** 社区清楚地看到了 Layotto 作为 MCP Server 的潜力——将现有的 Building Block API 暴露为 Agent 工具。但该 issue 因 30 天无活动被 stale bot 关闭，没有产出任何代码或 RFC。

### 1.2 Layotto 仓库当前状态

```
仓库: mosn/layotto
Stars: 853
语言: Go
Topics: cloud-native, configuration-management, distributed-lock,
        microservice, pubsub, service-mesh, sidecar, state-management
最近推送: 2026-03-23
```

**最近提交活动分析（2025-06 至 2026-03）：**
- 大部分为依赖更新（Dependabot），如 grpc、webpack、lodash 升级
- 唯一的功能提交是 heartbeat 相关修复（2026-01-26）
- 无任何 AI/Agent/MCP 相关代码合并
- Topics 列表中没有 "AI"、"agent"、"LLM"、"MCP" 等关键词

**结论：Layotto 处于低活跃维护状态，没有 AI/Agent 方向的实际推进。**

### 1.3 MOSN/蚂蚁集团的 AI 战略与 Layotto 的缺位

蚂蚁集团在 AI Agent 领域高度活跃：
- **蚂蚁阿福**: 健康 AI Agent，月活突破 1500 万
- **蚂蚁数科**: 金融 Agent 矩阵，覆盖 100% 国有和股份制银行
- **CodeFuse**: 蚂蚁的代码生成大模型

但这些 AI 投入与 Layotto 完全没有交集。蚂蚁的 AI Agent 走的是垂直领域应用路线，而不是通过 Layotto 这种基础设施层来支持。

### 1.4 对比：Dapr Agents 已经做到了

与 Layotto 的停滞形成鲜明对比的是，Dapr（同为 Multi-Runtime 范式）已经推出了完整的 AI Agent 支持：

| 维度 | Dapr Agents | Layotto |
|------|-------------|---------|
| 发布状态 | GA v1.0（2026-03-23，就在昨天） | 无任何进展 |
| AI Agent 支持 | 完整 Python 框架 | 仅有一个已关闭的 issue |
| MCP 支持 | 规划中 | 社区建议但未采纳 |
| LLM 集成 | 支持多模型 | 无 |
| Virtual Actor | 支持千级 Agent 单机运行 | 无 |
| CNCF 背景 | CNCF Graduated | CNCF Sandbox |
| 合作伙伴 | NVIDIA、企业用户 | — |

**Dapr Agents v1.0 核心能力:**
- Scale-to-Zero 架构：虚拟 Actor 表示 Agent，单机可运行数千个
- 持久化工作流引擎：保持上下文、持久化记忆、恢复长时间任务
- 安全连接 50+ 企业数据源
- 供应商中立（CNCF 项目）

### 1.5 Layotto 在 Agent Harness 时代的定位分析

```
┌──────────────────────────────────────────────────────┐
│           Multi-Runtime → Agent Harness 演进         │
│                                                      │
│  已完成转型:                                         │
│  ├── Dapr → Dapr Agents (GA v1.0, 2026-03)         │
│  ├── Higress → MCP Server Gateway (阿里巴巴)        │
│  └── Envoy → 探索 AI Gateway 方向                   │
│                                                      │
│  停留原地:                                           │
│  └── Layotto (Issue #1090 已关闭，无实际推进)        │
│                                                      │
│  潜在路径（如果 Layotto 要追赶）:                    │
│  ├── 1. 将 Building Block API 暴露为 MCP Server     │
│  ├── 2. 复用 Sidecar 模式做 Agent 沙箱              │
│  ├── 3. 利用 MOSN 的流量管理做 Agent 网关            │
│  └── 4. WASM 能力可用于 Agent 插件隔离执行           │
└──────────────────────────────────────────────────────┘
```

---

## 第二部分：全球 Agent Harness 实现全景

### 2.0 定义澄清：Harness vs Framework vs Application

在开始分类之前，明确术语定义（基于 2025-2026 行业共识）：

```
Framework（框架）:
  提供构建 Agent 的库和 API
  例: LangChain, CrewAI, AutoGen
  类比: 编程语言的标准库

Runtime（运行时）:
  Agent 执行的环境和基础设施
  例: E2B Sandbox, Dapr Agents Runtime
  类比: JVM, Node.js

Harness（线束/挽具）:
  围绕 LLM 的完整软件基础设施
  = Tools + Knowledge + Observation + Action Interfaces + Permissions
  包含: prompt 管理、上下文工程、工具调用、安全层、生命周期管理
  类比: 操作系统

关系: Framework → Runtime → Harness（层级递进，越往上越 opinionated）
```

> "While a framework provides the building blocks for tools or implements the agentic loop, the harness provides prompt presets, opinionated handling for tool calls, lifecycle hooks or ready-to-use capabilities like planning, filesystem access or sub-agent management."

### 2.1 真正的 Agent Harness 实现

#### A. DeerFlow 2.0 (ByteDance) — SuperAgent Harness

```
定位: 开源 SuperAgent Harness
Stars: 37K+
协议: MIT
基础: LangGraph 1.0 + LangChain

架构要点:
├── 两层设计: Harness 层 (发布为框架) + App 层 (应用特定服务)
├── make_lead_agent(): 11 阶段中间件管道
├── Docker 隔离容器: "computer-in-a-box"
├── 子 Agent 编排: 规划 → 分解 → 委派
├── Skills 系统: 可扩展的 Agent 能力
├── 记忆管理: 跨会话持久化
└── 消息网关: 多渠道输入输出

版本演进:
  v1 (2025-05): 专注深度研究的框架
  v2 (2026-03): 完全重写，通用 SuperAgent Harness
```

**Harness 判定: 是 (明确自称 "SuperAgent Harness")**

#### B. Claude Code (Anthropic) — Terminal Agent Harness

```
定位: 终端 Agent + Harness 参考实现
基础: Claude Agent SDK

Harness 公式:
  Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions

架构要点:
├── Agent Loop: Think → Critique → Act → Verify → Compact
├── 内置工具: Read, Write, Edit, Bash, Glob, Grep, WebSearch...
├── MCP 延迟加载: ToolSearch 按需发现工具
├── 5 层安全: 纵深防御
├── SubAgent: Agent 工具产生子 Agent
├── 上下文自动压缩: 长会话管理
├── 持久记忆: CLAUDE.md / MEMORY.md
├── Skill 系统: /commit, /review-pr 等
├── Hook 系统: 生命周期事件
├── Agent Teams: 并行 Agent 协作
└── Git Worktree 隔离: 并行任务
```

**Harness 判定: 是 (行业公认的 Harness 参考实现)**

#### C. OpenHands (原 OpenDevin) — Cloud Coding Agent Platform

```
定位: 云端编码 Agent 开放平台
Stars: 64K+
协议: MIT
论文: ICLR 2025

架构要点:
├── 事件流抽象: Action/Observation 感知-行动循环
├── Docker 沙箱: 完整 OS（bash, browser, IPython）
├── V1 SDK 重构 (2025-11):
│   ├── 4 个包: SDK, Tools, Workspace, Server
│   ├── 本地执行 → 远程运行时: WebSocket 双向通信
│   ├── 有状态、事件溯源、可组合
│   └── 可选沙箱: 本地 ↔ Docker/K8s
├── Agent Hub: 10+ 预构建 Agent
├── CodeAct 架构: 通过代码执行行动
├── 层级 Agent: 委派子任务
└── MCP 支持

与 DeerFlow/Claude Code 对比:
  ✓ 沙箱化执行环境 (类似 DeerFlow)
  ✓ 事件流架构 (类似 Claude Code 的 event log)
  ✓ 模块化 SDK (比 Claude Code 更解耦)
  ✗ 无内置 Skill 系统 (不同于 DeerFlow/Claude Code)
  ★ 最接近"Agent Harness"的开源实现之一
```

**Harness 判定: 是 (尤其是 V1 SDK 重构后，明确是可组合的 Agent Harness)**

#### D. Goose (Block / AAIF) — On-Machine Agent

```
定位: 可扩展的本地 AI Agent
Stars: 30K+
协议: Apache 2.0
归属: Block → Linux Foundation AAIF

架构要点:
├── 本地运行: on-machine Agent
├── 任意 LLM: 多模型配置
├── MCP 深度集成: 原生 MCP Server 支持
├── CLI + Desktop App: 双界面
├── 自主执行: 构建、编码、调试、编排工作流
├── AAIF 贡献: 与 MCP, AGENTS.md 共同治理
└── Grant Program: 社区资助生态

与 DeerFlow/Claude Code 对比:
  ✓ 终端 Agent (类似 Claude Code)
  ✓ MCP 原生支持 (类似 Claude Code)
  ✓ 模型无关 (优于 Claude Code 的 Claude-only)
  ✗ 无 SubAgent 编排 (不同于 DeerFlow)
  ✗ Harness 层不如 Claude Code 丰富
```

**Harness 判定: 部分是 (更接近带 Harness 特征的 Agent Application)**

#### E. OpenClaw — 个人 AI 助手

```
定位: 个人 AI 助手 Agent
Stars: 196K+ → 210K+ (2026-02 至 2026-03，史上增长最快的开源项目之一)
协议: 开源
创建者: Steinberger (已加入 OpenAI，项目转为独立基金会)

架构要点:
├── Hub-and-Spoke 架构:
│   ├── Gateway: WebSocket 控制平面
│   ├── Brain: ReAct 推理循环
│   ├── Memory: Markdown 持久化上下文
│   ├── Skills: 插件能力
│   └── Heartbeat: 调度与监控
├── 多渠道: WhatsApp, iMessage, Slack, macOS, Web, CLI
├── Agent Runtime: 组装上下文 → 调用模型 → 执行工具 → 持久化状态
├── 本地优先: Gateway 在本地运行
└── 浏览器自动化, 文件操作, Canvas, 定时任务

与 DeerFlow/Claude Code 对比:
  ✓ 完整的 Harness 结构 (Gateway + Brain + Memory + Skills)
  ✓ 多渠道接入 (远超 Claude Code 的终端-only)
  ✗ 非开发者聚焦 (更通用的个人助手)
  ✗ 不如 DeerFlow 的研究深度
  ★ 个人 Agent Harness 的标杆
```

**Harness 判定: 是 (个人 Agent 领域的完整 Harness)**

#### F. KWeaver — Enterprise Decision Agent Harness

```
定位: Harness-First 企业决策 Agent 基础设施
Stars: 293
协议: 开源

描述: "KWeaver Core is a harness-first foundation for enterprise decision agents.
It turns fragmented data, knowledge, tools, and policies into governed context,
safe execution, and verifiable feedback loops."

架构要点:
├── 语义建模
├── 实时访问
├── 运行时控制
├── TraceAI: 可审计追踪
└── 知识网络 (BKN)
```

**Harness 判定: 是 (明确的 "harness-first" 定位，但社区规模小)**

### 2.2 Harness 特征显著的 IDE/编辑器 Agent

#### G. Cursor 2.0 — Agent-First IDE

```
定位: Agent-First 代码编辑器
收入: $2B+ ARR (2026)
模型: 专有 Composer (MoE架构，RL训练)

架构要点:
├── Composer Agent: 专用 MoE 模型，4x 低延迟
├── 并行 Agent: 最多 8 个同时运行，Git Worktree 隔离
├── 异步 SubAgent: 子 Agent 可再产生子 Agent（Agent 树）
├── Automations (2026-03): 事件触发 Agent（Slack、代码变更、定时器）
├── 全代码库理解
├── Diff 预览 + 人类审批
└── Composer 2.0: 每小时运行数百个自动化

与 DeerFlow/Claude Code 对比:
  ✓ Agent Loop + 并行执行 (类似 Claude Code Agent Teams)
  ✓ SubAgent 产生 (类似 DeerFlow)
  ✓ Git Worktree 隔离 (同 Claude Code)
  ✗ 闭源，不可分析内部 Harness
  ✗ IDE 绑定，不是通用 Harness
  ★ 商业化最成功的 Agent Harness
```

**Harness 判定: 是 (IDE 形态的 Agent Harness，但闭源)**

#### H. Cline / Roo Code — VS Code Agent Extension

```
定位: 开源 VS Code Agent 扩展
Stars: 高增长 (VS Code Marketplace 上最受欢迎的 Agent 扩展)
协议: 开源

架构要点:
├── Plan + Act 工作流: 先规划再执行
├── Diff 预览 + 审批: "Agent proposes; you approve"
├── 模型无关: OpenAI, Anthropic, Google, Ollama
├── 跨仓库推理
├── 文件/依赖分析 → 构建代码库心理模型
├── CLI 2.0 (2026-02): 终端作为一等开发表面
├── 子 Agent (v3.58)
└── 全部操作可追溯日志

与 DeerFlow/Claude Code 对比:
  ✓ 全透明操作 (类似 Claude Code 的 Ask/Approve)
  ✓ 模型无关 (优于 Claude Code)
  ✓ CLI 模式 (2026 新增，对标 Claude Code)
  ✗ VS Code 绑定（虽然有 CLI 2.0）
  ✗ 无内置记忆系统
```

**Harness 判定: 部分是 (IDE 嵌入式 Harness，CLI 2.0 向独立 Harness 演进)**

#### I. Windsurf (原 Codeium, 现 Cognition AI) — Agentic IDE

```
定位: AI 原生 Agentic IDE
收购: Cognition AI 于 2025-12 收购

架构要点:
├── Cascade Agent: 理解全代码库，多文件编辑，终端命令
├── Codemaps: 代码架构可视化
├── 渐进式学习: ~48小时学习项目模式
├── 基于 VS Code 但完全重构
├── Devin 集成: Cognition 将 Devin 能力整合进 IDE
└── 自然语言 → 跨文件协调变更

与 DeerFlow/Claude Code 对比:
  ✓ 全代码库理解 (类似)
  ✗ 闭源
  ✗ IDE 形态，非通用 Harness
  ★ 与 Devin 合并后可能成为强力 Agent Harness
```

**Harness 判定: 部分是 (IDE Harness，与 Devin 合并后潜力大)**

#### J. Continue.dev — 开源 IDE Agent

```
定位: 开源 AI 编码助手
协议: 开源

架构要点:
├── Agent Mode: 工具增强的 Chat
├── System Message Tools: XML 格式确保跨模型一致性
├── YAML 配置: 完全可配置
├── VS Code + JetBrains 双平台
├── 模型无关
└── 源码控制的 AI 检查 (CI 可执行)

与 DeerFlow/Claude Code 对比:
  ✓ 开源 + 模型无关
  ✗ Harness 层较薄 (更像工具增强的 Chat)
  ✗ 无记忆系统、无 SubAgent
```

**Harness 判定: 否 (更像 Agent-enabled IDE 插件，而非独立 Harness)**

#### K. Warp — Terminal-First ADE

```
定位: Agentic Development Environment (终端)
模式: 商业

架构要点 (2025-2026):
├── Oz 平台: 云 Agent 编排基础设施
│   ├── Full Terminal Use: Agent 附着 PTY 会话
│   └── Computer Use: 沙箱桌面 GUI 环境
├── 5 组件 Agent: instructions, profile, trigger, environment, host
├── Warp Code: 内置文件编辑器 + WARP.md 规则文件
├── 交互式命令处理: Agent 能识别等待输入并处理
├── 多模型: OpenAI, Anthropic, Google
└── 上下文即一等公民: 收集、索引、管理信息

与 DeerFlow/Claude Code 对比:
  ✓ 终端优先 (与 Claude Code 最接近)
  ✓ 规则文件 (WARP.md ≈ CLAUDE.md)
  ✓ 云 Agent 执行 (类似 DeerFlow 的 Docker 隔离)
  ✗ 闭源
  ★ 终端 Agent Harness 的强力竞争者
```

**Harness 判定: 是 (终端形态的 Agent Harness，闭源)**

### 2.3 SWE Agent 系列（软件工程专用）

#### L. SWE-agent (Princeton/Stanford) — Agent-Computer Interface

```
定位: 自动修复 GitHub Issue 的 Agent
Stars: 高
论文: NeurIPS 2024
协议: MIT

架构要点:
├── ACI (Agent-Computer Interface): 自定义的 Agent-计算机交互界面
├── SWE-ReX: 远程执行容器内 Shell 会话
├── Docker 隔离: 本地或远程容器
├── 命令行可执行: sweagent
├── 支持任意 LLM
└── 分阶段: 代码搜索 → 理解 → 编辑 → 测试

与 DeerFlow/Claude Code 对比:
  ✓ Docker 沙箱 (类似 DeerFlow)
  ✓ ACI 设计理念先进
  ✗ 专注 bug 修复，非通用 Agent
  ✗ 无记忆系统、无 SubAgent
  ★ ACI 概念对 Harness 工程有重要启发
```

**Harness 判定: 部分是 (ACI 是 Harness 工程的重要理论贡献，但产品形态偏窄)**

#### M. AutoCodeRover — 自主程序改进

```
定位: AI Agent 用于大型软件项目的程序改进
架构: bash + python 工具 → 搜索代码 → 理解上下文 → 生成补丁

与 Agentless 的关键区别:
├── AutoCodeRover: 自主 Agent 决策 + 专用 API
└── Agentless: 移除 Agent 自主性，结构化两阶段流程
    (1) 层级定位 bug 位置
    (2) 采样多个补丁

与 DeerFlow/Claude Code 对比:
  ✗ 窄领域 (仅程序修复)
  ✗ 无 Harness 层 (直接的 Agent → Tools 调用)
```

**Harness 判定: 否 (Agent Application，非 Harness)**

### 2.4 平台/基础设施层

#### N. E2B — Agent 沙箱基础设施

```
定位: AI Agent 的云端沙箱基础设施
Stars: 高
协议: 开源
采用: ~50% 的 Fortune 500

架构要点:
├── Firecracker 微 VM: <200ms 启动
├── 隔离沙箱: 安全执行 AI 生成的代码
├── SDK: JavaScript + Python
├── Code Interpreter: 运行 AI 生成代码
├── Desktop: 图形环境（Computer Use）
└── 每周生成数百万沙箱

与 DeerFlow/Claude Code 对比:
  ★ 不是 Harness，是 Harness 的底座
  DeerFlow 的 Docker 容器 ≈ E2B 沙箱
  Claude Code 的 Bash tool ≈ 本地版的 E2B
```

**Harness 判定: 否 (是 Harness 的 Runtime 基础设施层)**

#### O. NVIDIA OpenShell — Agent 治理运行时

```
定位: Agent 安全治理运行时
发布: GTC 2026
协议: Apache 2.0
关联: NemoClaw 技术栈

架构要点:
├── 三组件:
│   ├── Sandbox: 专用沙箱
│   ├── Policy Engine: 文件系统/网络/进程级治理
│   └── Privacy Router: 控制推理数据流向
├── 进程外强制: 控制在 Agent 进程外，不可被 Agent 覆盖
├── 细粒度策略:
│   ├── Per-binary: 限制可执行文件
│   ├── Per-endpoint: 限制网络目标
│   └── Per-method: 限制 API 调用
└── JetPatch Enterprise Control Plane: 实时部署/限流/停止 Agent

与 DeerFlow/Claude Code 对比:
  ★ 不是 Harness，是 Harness 的安全层
  Claude Code 的 5 层安全 ≈ OpenShell 的理念
  但 OpenShell 更企业级、进程外强制
```

**Harness 判定: 否 (是 Harness 的 Safety/Governance 基础设施)**

#### P. Dapr Agents — 分布式 Agent 运行时

```
定位: 分布式 Agent 运行时框架
状态: GA v1.0 (2026-03-23)
协议: Apache 2.0 (CNCF)

架构要点:
├── Virtual Actor: Agent 表示为虚拟 Actor
├── Scale-to-Zero: 单机千级 Agent
├── 持久化工作流引擎: 上下文保持、记忆持久化、故障恢复
├── 50+ 企业数据源连接
├── MCP 规划中
├── 当前: Python SDK; 计划: .NET, Java, JS, Go
└── NVIDIA 合作

与 DeerFlow/Claude Code 对比:
  ★ 不是 Harness，是 Harness 的分布式 Runtime
  DeerFlow 单机 Docker ↔ Dapr Agents 分布式编排
  Claude Code 单会话 ↔ Dapr Agents 千级并发
```

**Harness 判定: 否 (是分布式 Agent Runtime，Harness 可以构建在其之上)**

### 2.5 应用层 Agent（非 Harness）

#### Q. Manus (Monica.im) — 通用自主 Agent

```
定位: 通用自主 AI Agent
来源: 中国创业公司 Monica.im
模型: Claude 3.5 Sonnet + Qwen (推测)

架构要点:
├── 多 Agent 系统:
│   ├── Planner Agent: 策略制定
│   ├── 信息检索 Agent
│   ├── 代码生成 Agent
│   └── 验证 Agent
├── CodeAct 范式: 生成 Python 脚本而非固定 token
├── 云端 VM: 每个会话一个独立虚拟机
├── GAIA 基准: 超越 GPT-4
└── Context Engineering: 公开博客分享经验

与 DeerFlow/Claude Code 对比:
  ✓ 多 Agent 编排 (类似 DeerFlow)
  ✓ 沙箱 VM (类似 DeerFlow Docker)
  ✗ 闭源
  ✗ 无法作为 Harness 使用
```

**Harness 判定: 否 (是基于内部 Harness 构建的 Agent Application)**

#### R. bolt.new / v0 / Lovable — AI App Builder

```
定位: AI 应用构建器
形态: Web 平台

bolt.new:
├── WebContainer 技术: 浏览器内运行
├── Agents-of-Agents 架构
├── Bolt v2: 数据库、托管、认证、SEO、支付一体化
└── "Claude Agent" 作为底层

Lovable:
├── Chat Mode Agent: 多步推理
├── 搜索项目文件、检查日志、查询数据库
├── 审批后才修改代码
└── 2.0 版本

v0 (Vercel):
├── React 组件生成
└── 更像 Code Generator
```

**Harness 判定: 否 (是 Application，底层使用 Harness 技术但不暴露)**

### 2.6 Devika / Devon — Devin 开源替代

```
Devika:
├── Stars: 15K+
├── 开源的 Agentic 软件工程师
├── 对标 Cognition Devin
└── 停滞状态 (2025 后活跃度下降)

Devon (Entropy Research):
├── Stars: ~200
├── 轻量级 Python Agent
├── 任务规划 + 记忆 + 多文件编辑 + Git 集成
└── 不太活跃
```

**Harness 判定: 否 (Agent Applications，Harness 层不成熟)**

---

## 第三部分：综合分析

### 3.1 Agent Harness 实现分类矩阵

```
                    通用性
                      ▲
                      │
        OpenClaw ●    │         ● DeerFlow 2.0
                      │
                      │    ● Goose
     Warp ●           │
                      │         ● OpenHands V1
     Cline ●          │
                      │    ● Claude Code
     Cursor ●         │
                      │
                      │    ● SWE-agent
     ──────────────────┼─────────────────────────→ 开放性
           闭源        │         开源
                      │
        Manus ●       │    ● AutoCodeRover
                      │    ● Devika
     Windsurf ●       │
                      │
     bolt.new ●       │
        v0 ●          │
```

### 3.2 真正的 Agent Harness 排名（按 Harness 完整度）

| 排名 | 项目 | Harness 完整度 | 开源 | Stars | 特色 |
|------|------|---------------|------|-------|------|
| 1 | Claude Code | ★★★★★ | 是 | — | 行业参考实现，最完整的 Harness |
| 2 | DeerFlow 2.0 | ★★★★★ | 是 | 37K+ | SuperAgent Harness，11阶段管道 |
| 3 | OpenHands V1 | ★★★★☆ | 是 | 64K+ | 学术+工业，SDK 架构最模块化 |
| 4 | OpenClaw | ★★★★☆ | 是 | 210K+ | 个人Agent Harness，多渠道 |
| 5 | Cursor 2.0 | ★★★★☆ | 否 | — | 商业最成功，Agent-First IDE |
| 6 | Goose | ★★★☆☆ | 是 | 30K+ | AAIF 支持，MCP 原生 |
| 7 | Warp 2.0 | ★★★☆☆ | 否 | — | 终端Harness，Oz 平台 |
| 8 | Cline | ★★★☆☆ | 是 | — | VS Code Agent，CLI 2.0 |
| 9 | SWE-agent | ★★☆☆☆ | 是 | — | ACI 理论贡献，窄领域 |
| 10 | KWeaver | ★★☆☆☆ | 是 | 293 | 企业决策 Harness，早期 |

### 3.3 基础设施层排名（支撑 Harness 的底座）

| 项目 | 层级 | 作用 |
|------|------|------|
| E2B | Runtime/Sandbox | Agent 执行沙箱 (Firecracker microVM) |
| OpenShell | Safety/Governance | Agent 安全治理运行时 (进程外策略) |
| Dapr Agents | Distributed Runtime | 分布式 Agent 编排 (Virtual Actor) |
| Microsoft Agent Framework | SDK + Runtime | 统一 AutoGen + Semantic Kernel |

### 3.4 关键趋势

```
2025 年趋势:
├── "Harness Engineering" 概念确立
├── MCP 成为事实标准 (97M+ 月下载)
├── 开源 Agent 爆发 (OpenHands, Goose, DeerFlow v1)
└── Dapr 发布 AI Agent 支持

2026 年趋势:
├── Harness 分层成熟: Framework → Runtime → Harness
├── Agent Harness 成为独立品类
├── AAIF (Linux Foundation): MCP + Goose + AGENTS.md 统一治理
├── OpenShell/NemoClaw: Agent 安全从理论到生产
├── OpenClaw 引爆个人 Agent Harness
├── DeerFlow v2: SuperAgent Harness 概念成型
├── Cursor $2B ARR: 证明 Agent-First IDE 商业价值
├── Dapr Agents GA: 分布式 Agent Runtime 成熟
└── "Harness 可靠性" 成为核心竞争力
```

### 3.5 Layotto 的机会窗口

```
如果 Layotto 要进入 Agent Harness 生态，可行路径:

路径 A: MCP Server Gateway（最小投入）
  └── 将现有 Building Block API 暴露为 MCP Server
  └── 投入: 中等
  └── 价值: Agent 可直接使用 Layotto 管理的基础设施
  └── 竞品: Higress 已在做

路径 B: Agent Sidecar Runtime（发挥 Sidecar 优势）
  └── 像 Dapr Agents 一样，在 Sidecar 中运行 Agent Runtime
  └── 投入: 大
  └── 价值: Mesh + Agent Runtime 融合
  └── 竞品: Dapr Agents GA 已领先

路径 C: WASM Agent Plugin（利用现有 WASM 支持）
  └── 用 WASM 沙箱运行 Agent 插件/Skills
  └── 投入: 中等
  └── 价值: 安全隔离的 Agent 能力扩展
  └── 竞品: 无直接竞品

路径 D: Agent Network Mesh（流量管理延伸）
  └── MOSN 数据面做 Agent-to-Agent 通信网格
  └── 投入: 大
  └── 价值: A2A 协议的基础设施层
  └── 竞品: 尚无成熟方案

现实评估:
├── Layotto 团队活跃度低（Ant Group 似乎已转移重心）
├── Issue #1090 的命运 = 社区信号的最佳指标
├── Dapr Agents 已有 1 年领先优势
└── 机会窗口正在关闭
```

---

## 参考来源

### Topic 1: Layotto/MOSN AI Activities
- [GitHub Issue #1090: Support MCP](https://github.com/mosn/layotto/issues/1090)
- [mosn/layotto Repository](https://github.com/mosn/layotto)
- [Dapr Agents GA Announcement (CNCF)](https://www.cncf.io/announcements/2026/03/23/general-availability-of-dapr-agents-delivers-production-reliability-for-enterprise-ai/)
- [Announcing Dapr AI Agents (CNCF)](https://www.cncf.io/blog/2025/03/12/announcing-dapr-ai-agents/)
- [Dapr Agents on TechCrunch](https://techcrunch.com/2025/03/12/daprs-microservices-runtime-now-supports-ai-agents/)

### Topic 2: Agent Harness Implementations
- [DeerFlow GitHub](https://github.com/bytedance/deer-flow)
- [DeerFlow on VentureBeat](https://venturebeat.com/orchestration/what-is-deerflow-and-what-should-enterprises-know-about-this-new-local-ai)
- [Claude Code GitHub](https://github.com/anthropics/claude-code)
- [Anthropic: Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [OpenHands ICLR 2025 Paper](https://arxiv.org/abs/2407.16741)
- [OpenHands Software Agent SDK Paper](https://arxiv.org/abs/2511.03690)
- [OpenHands GitHub](https://github.com/OpenHands/OpenHands)
- [SWE-agent GitHub](https://github.com/SWE-agent/SWE-agent)
- [SWE-agent NeurIPS 2024 Paper](https://arxiv.org/abs/2405.15793)
- [Manus Architecture Analysis (arxiv)](https://arxiv.org/html/2505.02024v1)
- [Manus Context Engineering Blog](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- [Goose GitHub](https://github.com/block/goose)
- [AAIF Launch (Linux Foundation)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw Wikipedia](https://en.wikipedia.org/wiki/OpenClaw)
- [E2B GitHub](https://github.com/e2b-dev/E2B)
- [OpenShell (NVIDIA/MarkTechPost)](https://www.marktechpost.com/2026/03/18/nvidia-ai-open-sources-openshell-a-secure-runtime-environment-for-autonomous-ai-agents/)
- [Cursor 2.0 Announcement](https://cursor.com/changelog/2-0)
- [Cursor Agent Product Page](https://cursor.com/product)
- [Cursor Agentic Coding (TechCrunch)](https://techcrunch.com/2026/03/05/cursor-is-rolling-out-a-new-system-for-agentic-coding/)
- [Cline CLI 2.0 (DevOps.com)](https://devops.com/cline-cli-2-0-turns-your-terminal-into-an-ai-agent-control-plane/)
- [Windsurf Review](https://www.taskade.com/blog/windsurf-review)
- [Warp Agentic Development](https://thenewstack.io/how-warp-went-from-terminal-to-agentic-development-environment/)
- [Continue.dev Docs](https://docs.continue.dev/ide-extensions/agent/how-it-works)
- [AutoCodeRover Paper](https://arxiv.org/pdf/2404.05427)
- [Agentless Paper](https://arxiv.org/html/2407.01489v1)
- [Agent Harness Definition (Inngest)](https://www.inngest.com/blog/your-agent-needs-a-harness-not-a-framework)
- [Harness Engineering (Martin Fowler)](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [OpenAI Harness Engineering](https://openai.com/index/harness-engineering/)
- [Agent Frameworks vs Runtimes vs Harnesses (Analytics Vidhya)](https://www.analyticsvidhya.com/blog/2025/12/agent-frameworks-vs-runtimes-vs-harnesses/)
- [Phil Schmid: Agent Harness 2026](https://www.philschmid.de/agent-harness-2026)
- [LangChain: Anatomy of an Agent Harness](https://blog.langchain.com/the-anatomy-of-an-agent-harness/)
- [KWeaver GitHub](https://github.com/kweaver-ai/kweaver)
- [Microsoft Agent Framework (InfoQ)](https://www.infoq.com/news/2025/10/microsoft-agent-framework/)
- [MCP on Wikipedia](https://en.wikipedia.org/wiki/Model_Context_Protocol)
