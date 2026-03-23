# Agent Harness 学习路线图

## 总览

```
Phase 1          Phase 2          Phase 3          Phase 4
基础认知          核心模式          进阶主题          生产实践
(1 周)            (2 周)           (2 周)           (持续)

┌────────┐     ┌────────┐      ┌────────┐      ┌────────┐
│理解     │     │动手     │      │深入     │      │实战     │
│什么是   │────►│构建     │─────►│优化     │─────►│部署     │
│Harness  │     │Agent    │      │Harness  │      │运维     │
└────────┘     └────────┘      └────────┘      └────────┘

前置知识: LLM API 基础, Python/TS, 基本的 CLI 使用
```

## Phase 1: 基础认知 (1 周)

**目标**: 理解 Agent Harness 的概念、动机和架构

```
Day 1-2: 概念理解
├── 读: 00-overview.md (什么是 Agent Harness)
├── 读: 01-core-concepts.md (核心术语)
├── 读: Anthropic "Building Effective Agents"
└── 思考: Agent vs Chatbot 的根本区别是什么?

Day 3-4: 架构认知
├── 读: 02-architecture.md (两阶段六子系统)
├── 读: 03-agent-loop.md (Agent Loop 详解)
├── 读: Martin Fowler "Harness Engineering"
└── 练习: 用 Claude Code 完成一个实际编码任务,观察 Agent Loop

Day 5-7: 生态了解
├── 读: 09-frameworks-comparison.md (框架对比)
├── 浏览: Claude Code 文档
├── 浏览: OpenAI Codex Agent Loop 文章
└── 输出: 用自己的话写一段 Agent Harness 的定义
```

**Phase 1 完成标志**:
- [ ] 能清楚解释 Agent Harness 的定义和价值
- [ ] 理解 Agent Loop 的完整流程
- [ ] 知道至少 3 个主流框架的特点

→ 详见 [phase1-foundations.md](./phase1-foundations.md)

## Phase 2: 核心模式 (2 周)

**目标**: 掌握 Harness 的核心组件和设计模式

```
Week 1: 核心组件
├── Day 1-2: 工具集成
│   ├── 读: 04-tool-integration.md
│   ├── 练习: 用 Claude Agent SDK 注册自定义工具
│   └── 练习: 配置一个 MCP Server
│
├── Day 3-4: 上下文工程
│   ├── 读: 05-context-engineering.md
│   ├── 练习: 构建一个动态 System Prompt
│   └── 练习: 实现简单的上下文压缩
│
└── Day 5-7: 记忆系统
    ├── 读: 06-memory-state.md
    ├── 练习: 实现 Initializer-Executor 模式
    └── 练习: 设计 Progress File 格式

Week 2: 设计模式
├── Day 1-2: 安全与护栏
│   ├── 读: 07-safety-guardrails.md
│   ├── 练习: 实现一个 PreToolUse Hook
│   └── 练习: 设计工具审批流程
│
├── Day 3-4: 设计模式
│   ├── 读: 08-design-patterns.md
│   ├── 练习: 实现 Orchestrator-Worker 模式
│   └── 练习: 实现 Evaluator-Optimizer 模式
│
└── Day 5-7: 综合练习
    ├── 项目: 构建一个简单的 Code Review Agent
    │   ├── Agent Loop (基于 Claude API)
    │   ├── 工具: 文件读取 + Git diff
    │   ├── 安全: 只读权限
    │   └── 输出: 审查报告
    └── 回顾: 这个 Agent 的 Harness 包含哪些组件?
```

**Phase 2 完成标志**:
- [ ] 能独立注册和使用自定义工具
- [ ] 理解并能实现基本的上下文管理策略
- [ ] 能设计 Progress File 实现跨会话连续性
- [ ] 能实现至少 2 种设计模式
- [ ] 构建了一个简单但完整的 Agent

→ 详见 [phase2-core-patterns.md](./phase2-core-patterns.md)

## Phase 3: 进阶主题 (2 周)

**目标**: 深入理解可靠性、可观测性和优化

```
Week 1: 可靠性工程
├── Day 1-2: 纵深防御
│   ├── 实现 5 层安全架构
│   ├── 研究: Claude Code 的安全设计
│   └── 练习: 为 Agent 添加 Doom Loop 检测
│
├── Day 3-4: 错误处理与恢复
│   ├── 实现: 工具执行重试策略
│   ├── 实现: 死循环检测和恢复
│   └── 实现: 优雅降级机制
│
└── Day 5-7: 可观测性
    ├── 实现: Agent 行为追踪 (tracing)
    ├── 实现: 工具调用日志
    └── 分析: 从 trace 中发现优化点

Week 2: 性能优化
├── Day 1-2: Prompt Cache 优化
│   ├── 实现: 静态前缀 + 动态后缀
│   ├── 测量: 缓存命中率
│   └── 优化: 减少前缀变动
│
├── Day 3-4: 多 Agent 编排
│   ├── 实现: SubAgent 并行执行
│   ├── 实现: Agent Handoff
│   └── 实现: 结果合并策略
│
└── Day 5-7: 综合项目
    ├── 项目: 构建一个多功能 Dev Agent
    │   ├── 能力: 代码修改 + 测试 + 审查
    │   ├── 安全: 完整的纵深防御
    │   ├── 记忆: 跨会话连续性
    │   ├── 观测: 完整的 tracing
    │   └── 优化: Prompt Cache + 上下文管理
    └── 回顾: 哪些改进对可靠性影响最大?
```

**Phase 3 完成标志**:
- [ ] 实现了完整的安全架构
- [ ] 能检测和处理 Agent 死循环
- [ ] 实现了 Agent 行为追踪
- [ ] 能优化 Prompt Cache 命中率
- [ ] 构建了一个多能力 Agent

→ 详见 [phase3-advanced.md](./phase3-advanced.md)

## Phase 4: 生产实践 (持续)

**目标**: 将 Agent Harness 应用到实际生产场景

```
主题 1: 评估与基准
├── 建立 Agent 评估指标
├── 设计回归测试套件
├── 持续监控性能指标
└── A/B 测试不同 Harness 策略

主题 2: 团队协作
├── 为团队建立 Agent 使用规范
├── 设计 CLAUDE.md / AGENTS.md 模板
├── 建立 Hook 和 Plugin 库
└── 分享最佳实践

主题 3: 持续改进
├── 分析失败 trace → 改进 Harness
├── 跟踪框架和模型更新
├── 参与社区讨论
└── "每个失败都应该产生永久的环境修复"

主题 4: 前沿探索
├── 新模型能力的利用 (更长上下文, 更好推理)
├── 新工具协议的集成
├── 多 Agent 系统的规模化
└── Agent 安全性的新挑战
```

→ 详见 [phase4-production.md](./phase4-production.md)

## 学习资源优先级

```
必读 (★★★★★):
1. Anthropic: Building Effective Agents
2. Anthropic: Effective Harnesses for Long-Running Agents
3. Martin Fowler: Harness Engineering
4. OpenAI: Unrolling the Codex Agent Loop

推荐 (★★★★):
5. LangChain: Improving Deep Agents with Harness Engineering
6. OpenAI: Harness Engineering in a Codex-First World
7. Claude Code 文档: How Claude Code Works
8. Building AI Coding Agents (arXiv 2603.05344)

参考 (★★★):
9. LangChain: State of Agent Engineering
10. Databricks: Agent System Design Patterns
11. 各框架官方文档
```
