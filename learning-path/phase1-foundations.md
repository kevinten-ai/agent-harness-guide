# Phase 1: 基础认知

## 学习目标

理解 Agent Harness 的"是什么"和"为什么"，建立正确的心智模型。

## Day 1-2: 概念理解

### 核心问题

1. **什么是 Agent？**
   - 不是 Chatbot（人类驱动对话）
   - 是自主决策执行的系统（模型决定下一步）
   - Agent = LLM + 工具 + 循环 + 自主性

2. **什么是 Harness？**
   - Agent 周围的基础设施
   - 管理模型以外的一切
   - 类比：LLM 是 CPU，Harness 是操作系统

3. **为什么需要 Harness？**
   - 裸 LLM 无状态、无工具、无验证、无安全
   - Harness 提供连续性、能力和护栏
   - LangChain 实证：同模型、不同 Harness → 性能差 13.7%

### 阅读清单

```
1. knowledge-graph/00-overview.md (本项目)
2. knowledge-graph/01-core-concepts.md (本项目)
3. Anthropic: "Building Effective Agents"
   https://www.anthropic.com/research/building-effective-agents
```

### 思考练习

- 用自己的经历想一个例子：你使用 ChatGPT/Claude 时，什么时候觉得"要是它能直接帮我改代码就好了"？那个"直接改"就需要 Harness。
- 画一个简图：裸 LLM API 调用 vs Agent (有什么不同？)

## Day 3-4: 架构认知

### 核心问题

1. **Agent Loop 怎么工作？**
   - 模型推理 → 输出工具调用 → Harness 执行 → 结果反馈 → 循环
   - 关键：一个"turn"可能包含数十次迭代

2. **Harness 有哪些子系统？**
   - Agent Loop（执行引擎）
   - Tool Integration（工具集成）
   - Context Engine（上下文引擎）
   - Memory Manager（记忆管理）
   - Verification（验证层）
   - Safety（安全层）

3. **Scaffolding vs Runtime？**
   - Scaffolding = 构建阶段（组装 Agent）
   - Runtime = 运行阶段（运行和管理 Agent）

### 阅读清单

```
1. knowledge-graph/02-architecture.md (本项目)
2. knowledge-graph/03-agent-loop.md (本项目)
3. Martin Fowler: "Harness Engineering"
   https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html
```

### 实践练习

**使用 Claude Code 完成一个编码任务，观察 Agent Loop：**

```
建议任务: 让 Claude Code 为你写一个简单的工具函数并加上测试

观察点:
├── Agent 读取了哪些文件？(Context Gathering)
├── Agent 思考了什么？(Thinking Phase)
├── Agent 调用了哪些工具？(Tool Use)
├── 工具调用的顺序是什么？(Execution Order)
├── 总共经历了几次迭代？(Loop Count)
├── 最终如何判断任务完成？(Termination)
└── 有没有遇到错误并自我修正？(Error Recovery)
```

## Day 5-7: 生态了解

### 核心问题

1. **有哪些主流框架？** 各自定位是什么？
2. **MCP 是什么？** 为什么是重要标准？
3. **"Harness Engineering" 这个领域在解决什么问题？**

### 阅读清单

```
1. knowledge-graph/09-frameworks-comparison.md (本项目)
2. Claude Code 文档: "How Claude Code Works"
   https://code.claude.com/docs/en/how-claude-code-works
3. OpenAI: "Unrolling the Codex Agent Loop"
   https://openai.com/index/unrolling-the-codex-agent-loop/
```

### 输出练习

**用自己的话写一段 Agent Harness 的定义** (200-300 字)

要求包含：
- 是什么（定义）
- 解决什么问题（动机）
- 核心组件有哪些（架构）
- 为什么重要（价值）

## Phase 1 自检清单

```
□ 能解释 Agent 和 Chatbot 的根本区别
□ 能说出 Agent Harness 的定义
□ 理解 "LLM = CPU, Harness = OS" 的比喻
□ 能画出 Agent Loop 的基本流程
□ 知道 Scaffolding 和 Runtime 的区别
□ 能列出 Harness 的 6 个核心子系统
□ 了解至少 3 个主流框架的定位
□ 知道 MCP 是什么、解决什么问题
□ 观察过一次真实的 Agent Loop 执行
□ 写出了自己的 Agent Harness 定义
```

## 进入 Phase 2 的信号

当你能自信地向同事解释"什么是 Agent Harness，为什么它重要"时，你就可以进入 Phase 2 了。
