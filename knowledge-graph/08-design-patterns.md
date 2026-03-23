# 设计模式

## 模式总览

```
┌──────────────────────────────────────────────────────────────┐
│                    Agent Harness 设计模式                      │
│                                                              │
│  ┌─── 执行模式 ──────┐  ┌─── 架构模式 ──────┐               │
│  │ Extended ReAct     │  │ Orchestrator-Worker│               │
│  │ Reasoning Sandwich │  │ Evaluator-Optimizer│               │
│  │ Dual-Mode          │  │ Middleware Compose │               │
│  └───────────────────┘  └───────────────────┘               │
│                                                              │
│  ┌─── 可靠性模式 ────┐  ┌─── 协作模式 ──────┐               │
│  │ Init-Executor Split│  │ Human-in-the-Loop │               │
│  │ Schema Isolation   │  │ Multi-Agent        │               │
│  │ Artifact Driven    │  │ Agent Handoff      │               │
│  └───────────────────┘  └───────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

## 一、执行模式

### 1. Extended ReAct Loop

在标准 ReAct (Think → Act) 基础上增加 Critique 和 Verify：

```
标准 ReAct:
  Observe → Think → Act → Observe → ...

Extended ReAct:
  Observe → Think → Critique → Act → Verify → Observe → ...
                     ↑                  ↑
                  防止冲动行动       确认操作正确

效果: 减少无效迭代，提高首次正确率
成本: 每次迭代增加一些推理开销
权衡: 对复杂任务值得，对简单任务可跳过 Critique
```

### 2. Reasoning Sandwich (推理三明治)

LangChain 发现的关键模式：不同阶段分配不同的推理资源。

```
         推理资源
高 ████████████████    ← 规划阶段 (高推理)
   │              │
中 ████████████████    ← 执行阶段 (标准推理)
   │              │
高 ████████████████    ← 验证阶段 (高推理)

关键发现:
✗ 全程高推理 → 性能反而下降 (超时 + 过度思考)
✓ 三明治式分配 → 最佳平衡

实现方式:
├── 规划阶段: 使用更长的 thinking budget
├── 执行阶段: 标准 thinking budget
└── 验证阶段: 使用更长的 thinking budget
```

### 3. Dual-Mode (Plan/Execute 双模式)

分离规划和执行，各自有不同的工具权限：

```
┌─── Plan Mode (规划模式) ────┐    ┌─── Execute Mode (执行模式) ────┐
│                              │    │                                │
│ 可用工具: 只读                │    │ 可用工具: 完整                  │
│ ├── Read                     │    │ ├── Read, Write, Edit          │
│ ├── Glob, Grep               │    │ ├── Bash                       │
│ ├── WebSearch, WebFetch       │    │ ├── Agent (子Agent)            │
│ └── AskUserQuestion          │    │ └── 所有工具                    │
│                              │    │                                │
│ 产出: 结构化计划              │    │ 输入: 计划                     │
│ ├── 步骤列表                 │    │ 行为: 按计划执行               │
│ ├── 文件变更清单              │    │ 验证: 每步验证通过再继续       │
│ └── 风险评估                 │    │                                │
│                              │    │                                │
│ 安全: 不可能意外修改代码      │    │ 安全: 有计划指导,操作有据       │
└──────────────────────────────┘    └────────────────────────────────┘

工作流: Plan → Review (人工) → Approve → Execute
```

## 二、架构模式

### 4. Orchestrator-Worker (编排者-工作者)

```
┌───────────────────────────────────────────┐
│        Orchestrator (编排 Agent)            │
│                                           │
│  接收任务 → 分解 → 分配 → 汇总 → 输出     │
│                                           │
└───┬───────────────┬───────────────┬───────┘
    │               │               │
    ▼               ▼               ▼
┌────────┐   ┌────────────┐   ┌──────────┐
│Worker 1 │   │ Worker 2    │   │ Worker 3  │
│Research │   │ Implement   │   │ Test      │
│(只读)   │   │ (读写)       │   │ (执行)    │
└────────┘   └────────────┘   └──────────┘

特点:
├── 编排者: 高级 LLM, 负责规划和汇总
├── 工作者: 可以是更小/更快的模型
├── 工具隔离: 每个 Worker 只有需要的工具
└── 并行: 独立的 Worker 可并行执行
```

### 5. Evaluator-Optimizer (评估者-优化者)

```
┌──────────────────────────────────────────┐
│ Generator (生成器)                        │
│ 接收任务 → 生成初始方案                    │
└─────────────┬────────────────────────────┘
              │
              ▼ 方案
┌──────────────────────────────────────────┐
│ Evaluator (评估器)                        │
│ 评估方案 → 生成反馈 (基于明确标准)         │
└─────────────┬────────────────────────────┘
              │
         ┌────┴────┐
         │通过?    │
    Yes ──┘         └── No
    │                    │
    ▼                    ▼
  输出               反馈给 Generator
                    (包含具体改进建议)
                         │
                         └──► 循环直到通过

应用: 代码生成+代码审查、文档编写+事实核查
```

### 6. Middleware Composition (中间件组合)

```
LangChain 风格的可组合管道:

Agent Request
    │
    ▼
┌─────────────────────────────────────────┐
│ LocalContextMiddleware                   │
│ • 注入本地文件上下文                      │
│ • 添加项目特定信息                       │
└─────────────────────┬───────────────────┘
                      ▼
┌─────────────────────────────────────────┐
│ LoopDetectionMiddleware                  │
│ • 检测重复模式                           │
│ • 跟踪编辑计数                           │
│ • 触发策略时注入 "reconsider"            │
└─────────────────────┬───────────────────┘
                      ▼
┌─────────────────────────────────────────┐
│ ReasoningSandwichMiddleware              │
│ • 规划阶段: max reasoning               │
│ • 执行阶段: standard reasoning          │
│ • 验证阶段: max reasoning               │
└─────────────────────┬───────────────────┘
                      ▼
┌─────────────────────────────────────────┐
│ PreCompletionChecklistMiddleware         │
│ • 完成前运行检查清单                     │
│ • 测试是否通过                           │
│ • 类型检查是否通过                       │
└─────────────────────┬───────────────────┘
                      ▼
Agent Response

优势:
├── 每个中间件职责单一
├── 可独立开发和测试
├── 可灵活组合和排序
└── 新需求 = 新增中间件,不改现有代码
```

## 三、可靠性模式

### 7. Initializer-Executor Split

(详见 [06-memory-state.md](./06-memory-state.md))

核心: 第一个会话建立环境和状态文件,后续会话读取状态增量执行。

### 8. Schema-Level Isolation (Schema 级隔离)

```
比运行时权限检查更安全的隔离方式:

原则: 不在 Schema 中的工具 = 不可能被调用

传统方式 (运行时检查):
  Agent 请求 Bash("rm -rf /")
  → 运行时检查 → 拒绝
  问题: 依赖检查逻辑的正确性

Schema 隔离:
  SubAgent 的可用工具 = [Read, Grep, Glob]
  → 模型根本无法生成 Bash 调用 (不在 Schema 中)
  → 从根本上不可能调用未授权的工具

应用:
├── Plan Mode: 只注册只读工具
├── Research SubAgent: 只注册搜索工具
├── Code Review SubAgent: 只注册读取工具
└── 不同信任级别: 不同工具集
```

### 9. Artifact-Driven Continuity (制品驱动的连续性)

```
每个有意义的操作都产生持久化制品:

行动               制品              用途
──────            ──────            ──────
代码变更     →    Git Commit     →   可回溯、可恢复
任务进度     →    progress.json  →   跨会话恢复
决策记录     →    decisions.json →   上下文保持
环境状态     →    baseline.txt   →   基线比对

新会话的第一步: 读取所有制品 → 恢复上下文

优势:
├── Agent 崩溃不丢失进度
├── 新会话快速恢复
├── 人类可审计所有操作
└── 多 Agent 可共享状态
```

## 四、协作模式

### 10. Human-in-the-Loop (人在回路中)

```
三种介入程度:

Fully Autonomous (全自动):
  Agent 独立执行 → 完成后报告
  适用: 低风险、有充分测试覆盖

Checkpoint-Based (检查点式):
  Agent 执行 → 到达检查点暂停 → 人类审查 → 继续
  适用: 中等风险、多步骤任务

Approval-Based (审批式):
  Agent 每步都请求批准 → 人类确认 → 执行
  适用: 高风险操作、生产环境变更
```

### 11. Multi-Agent (多 Agent 协作)

```
模式 A: Pipeline (管道式)
  Agent1 → Agent2 → Agent3 → 输出
  (每个 Agent 处理后交给下一个)

模式 B: Parallel (并行式)
  Input → Agent1 ──┐
       → Agent2 ──┤→ Merge → Output
       → Agent3 ──┘
  (独立任务并行执行)

模式 C: Hierarchical (层级式)
  Orchestrator
  ├── Team Lead A
  │   ├── Worker A1
  │   └── Worker A2
  └── Team Lead B
      ├── Worker B1
      └── Worker B2
  (复杂项目的分层管理)
```

### 12. Agent Handoff (Agent 交接)

```
Agent A 在特定条件下将控制权交给 Agent B:

┌──────────┐     handoff     ┌──────────┐
│ General   │ ──────────────► │ Specialist│
│ Agent     │     条件:       │ Agent     │
│           │  "需要专业知识"  │           │
└──────────┘                  └──────────┘

交接内容:
├── 任务描述
├── 当前上下文摘要
├── 已完成的步骤
└── 需要的专业能力

类比: 客服将复杂问题转给技术专家
```

## 模式选择指南

```
任务类型                        推荐模式
────────────                    ─────────
简单单步                    →   直接 Agent Loop
复杂多步                    →   Extended ReAct + Plan/Execute
需要可靠性                  →   Init-Executor + Artifact Driven
需要并行                    →   Orchestrator-Worker
需要质量保证                →   Evaluator-Optimizer
需要安全隔离                →   Schema Isolation
需要可扩展                  →   Middleware Composition
需要人类参与                →   Human-in-the-Loop
超大任务                    →   Multi-Agent Hierarchical
```

## 下一步

→ 阅读 [09-frameworks-comparison.md](./09-frameworks-comparison.md) 了解主流框架对比
