# Agent Harness 总览

## 定义

**Agent Harness** 是围绕 LLM 的完整软件基础设施，负责管理模型推理以外的一切——工具执行、记忆、状态持久化、上下文管理、验证和错误恢复。

它将一个无状态的语言模型转化为一个有能力的、可长时间运行的 Agent。

## 为什么需要 Agent Harness？

### 裸 LLM 的局限

| 局限 | 说明 |
|------|------|
| **无状态** | 每次调用都是独立的，没有持续性 |
| **无工具** | 无法读写文件、执行代码、访问网络 |
| **无验证** | 无法确认自己的输出是否正确 |
| **无安全** | 无法阻止自己执行危险操作 |
| **上下文有限** | 窗口有限，无法处理超长任务 |

### Harness 如何解决

```
裸 LLM API 调用          Agent Harness
─────────────────      ─────────────────────────
单次请求-响应      →    自主多步执行
无状态            →    三层记忆 (工作/会话/长期)
无工具            →    文件I/O、Shell、Web、API...
人类控制          →    模型自主决策执行顺序
无验证            →    内置测试执行、输出校验
无错误恢复        →    每步重试、死循环检测、降级链
无安全            →    纵深防御 (5+ 独立层)
```

## 术语起源

"Harness Engineering" 一词在 2025 年末由 OpenAI (Codex)、Anthropic (Claude Code) 和 LangChain 团队实践中明确提出，并在 Martin Fowler 的网站上被正式定义记录。

核心共识：**Harness，而非模型本身，才是决定 Agent 可靠性的主要因素。**

LangChain 的验证：他们的 Deep Agent 在 Terminal Bench 2.0 上从第 30 名提升到第 5 名，纯粹通过 Harness 改进（模型未变），得分提升 13.7 个百分点。

## 与相关概念的区别

```
                    抽象程度
         低 ◄────────────────────► 高
         │                         │
    Agent Loop    Agent Harness    Agent Framework
    (执行循环)     (完整基础设施)     (开发框架)
         │              │               │
    只负责循环     循环+工具+记忆     Harness+脚手架
    调用模型和     +安全+上下文+      +预置组件+
    工具          验证+状态管理       开发者API
```

- **Agent Loop**: Harness 的核心组件之一，只负责模型推理和工具调用的循环
- **Agent Harness**: 完整的基础设施层，包含 Loop + 所有支撑系统
- **Agent Framework**: 提供开发者 API 来构建 Harness 的工具包 (如 LangChain, CrewAI)
- **Agent Runtime**: 执行环境，侧重沙箱隔离和资源管理

## Harness 的两个阶段

### 阶段一：Scaffolding (构建)

在第一次 prompt 之前完成，组装 Agent：
- 构建 System Prompt
- 注册 Tool Schemas
- 初始化 SubAgent 注册表
- 环境发现 (工作目录、git 状态、平台信息等)

### 阶段二：Harness (运行)

Agent 启动后持续运行：
- 编排工具调度
- 管理上下文 (压缩、裁剪、注入)
- 执行安全检查
- 管理会话生命周期

## 下一步

→ 阅读 [01-core-concepts.md](./01-core-concepts.md) 了解核心概念与术语
