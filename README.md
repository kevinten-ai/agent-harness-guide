# Agent Harness 知识图谱与学习指南

> **Agent Harness**: 围绕 LLM 的完整软件基础设施，管理工具执行、记忆、状态持久化、上下文管理、验证和错误恢复——将无状态的语言模型转化为可靠的、长时间运行的 Agent。

## 核心比喻

```
LLM = CPU (提供原始智能)
Agent Harness = 操作系统 (提供连续性、能力和护栏)
```

## 知识图谱总览

```
                        ┌─────────────────────┐
                        │   Agent Harness      │
                        │   (操作系统层)        │
                        └──────────┬──────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
    ┌─────▼─────┐          ┌──────▼──────┐          ┌──────▼──────┐
    │ Scaffolding │          │   Runtime    │          │  Persistence │
    │ (构建阶段)   │          │  (运行阶段)   │          │  (持久化层)   │
    └─────┬─────┘          └──────┬──────┘          └──────┬──────┘
          │                        │                        │
    ┌─────▼─────┐    ┌────────────┼────────────┐    ┌──────▼──────┐
    │• System    │    │            │            │    │• Working     │
    │  Prompt    │    │            │            │    │  Context     │
    │• Tool      │    │            │            │    │• Session     │
    │  Schemas   │    │            │            │    │  State       │
    │• SubAgent  │    │            │            │    │• Long-term   │
    │  Registry  │    │            │            │    │  Memory      │
    │• Env       │    │            │            │    └─────────────┘
    │  Discovery │  ┌─▼──┐  ┌─────▼────┐  ┌───▼────┐
    └────────────┘  │Agent│  │  Tool     │  │Safety  │
                    │Loop │  │Integration│  │Layer   │
                    │     │  │  Layer    │  │        │
                    └──┬──┘  └─────┬────┘  └───┬────┘
                       │           │            │
                 ┌─────▼─────┐     │      ┌─────▼─────┐
                 │• Think     │     │      │• Prompt    │
                 │• Critique  │     │      │  Guard     │
                 │• Act       │     │      │• Schema    │
                 │• Verify    │     │      │  Restrict  │
                 │• Compact   │     │      │• Runtime   │
                 └────────────┘     │      │  Approval  │
                              ┌─────▼─────┐│• Tool     │
                              │• File I/O  ││  Validate │
                              │• Shell     ││• Lifecycle│
                              │• Web       ││  Hooks    │
                              │• LSP/Code  │└──────────┘
                              │• MCP       │
                              │• SubAgents │
                              └────────────┘
```

## 目录结构

```
harness/
├── README.md                      ← 你在这里
├── knowledge-graph/               ← 知识图谱 (概念详解)
│   ├── 00-overview.md             ← 总览：什么是 Agent Harness
│   ├── 01-core-concepts.md        ← 核心概念与术语
│   ├── 02-architecture.md         ← 架构：两阶段六子系统
│   ├── 03-agent-loop.md           ← Agent Loop：执行引擎详解
│   ├── 04-tool-integration.md     ← 工具集成层
│   ├── 05-context-engineering.md  ← 上下文工程
│   ├── 06-memory-state.md         ← 记忆与状态管理
│   ├── 07-safety-guardrails.md    ← 安全与护栏
│   ├── 08-design-patterns.md      ← 设计模式
│   ├── 09-frameworks-comparison.md← 主流框架对比
│   ├── 10-deerflow-case-study.md ← 案例：DeerFlow 2.0 深度解析
│   └── 11-multi-runtime-parallel.md ← Multi-Runtime 与 Harness 同构分析
├── learning-path/                 ← 学习路径
│   ├── roadmap.md                 ← 学习路线图 (4 阶段)
│   ├── phase1-foundations.md      ← 阶段 1：基础认知
│   ├── phase2-core-patterns.md    ← 阶段 2：核心模式
│   ├── phase3-advanced.md         ← 阶段 3：进阶主题
│   └── phase4-production.md       ← 阶段 4：生产实践
└── references/
    └── resources.md               ← 权威资源索引
```

## 推荐阅读顺序

1. **入门** → `knowledge-graph/00-overview.md` → `01-core-concepts.md`
2. **架构** → `02-architecture.md` → `03-agent-loop.md`
3. **能力** → `04-tool-integration.md` → `05-context-engineering.md` → `06-memory-state.md`
4. **可靠性** → `07-safety-guardrails.md` → `08-design-patterns.md`
5. **实战** → `09-frameworks-comparison.md` → `learning-path/roadmap.md`

## 一句话总结

> 2025 年证明了 Agent 可以工作；2026 年的核心命题是让它们**可靠地**工作——而 Harness 是决定可靠性的关键。
