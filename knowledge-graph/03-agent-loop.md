# Agent Loop: 执行引擎详解

## 什么是 Agent Loop

Agent Loop 是使 Agent "自主" 的核心机制。它是一个迭代循环，在每次迭代中：
1. 模型接收上下文
2. 模型决定做什么（思考+行动）
3. Harness 执行工具调用
4. 结果反馈给模型
5. 重复，直到任务完成

**关键区别**: 在 Agent Loop 中，**模型控制执行流程**，而不是调用方。

## 典型实现：以 Codex CLI 为例

OpenAI Codex CLI 公开的 Agent Loop 流程：

```
用户输入
   │
   ▼
构建 Prompt (系统提示 + 历史 + 用户消息)
   │
   ▼
┌──────────────────────────────────┐
│ LLM 推理                         │
│ 输出: Text + ToolCalls           │
└──────────┬───────────────────────┘
           │
     ┌─────┴──────┐
     │ 有 ToolCall? │
     └─────┬──────┘
       No  │  Yes
     ┌─────┘   └─────┐
     ▼               ▼
 [返回文本]    [执行每个 ToolCall]
 [任务结束]       │
                  ▼
           [将结果追加到历史]
                  │
                  ▼
           [重新调用 LLM] ──► 循环回去
```

**单个"turn"可能涉及数十次迭代。** 这是 Agent 与 Chatbot 的根本区别。

## Extended ReAct Loop

业界最佳实践是在标准 ReAct (Reason + Act) 基础上扩展：

```
标准 ReAct:          Extended ReAct:
────────────         ────────────────
Think → Act          Think → Critique → Act → Verify
```

### 各阶段详解

#### Phase 0: Context Compaction

```
触发条件: token 使用量接近窗口上限

动作:
├── 渐进摘要旧对话轮次
├── 截断冗长的工具输出
├── 将记忆卸载到持久存储
└── 裁剪历史保留关键信息

实现示例 (Claude Code):
• 自动压缩 prior messages
• 保留最近 N 轮完整对话
• 摘要更早的对话为简短描述
```

#### Phase 1: Thinking (深度推理)

```
目的: 在行动前进行深度思考

活动:
├── 理解当前任务状态
├── 分析已有信息
├── 规划下一步行动
├── 考虑多种方案
└── 评估风险和约束

关键: 某些模型支持 "extended thinking" 模式
      如 Claude 的 thinking 块、OpenAI 的 reasoning tokens
```

#### Phase 2: Self-Critique (自我审视)

```
目的: 在执行前质疑自己的计划

检查项:
├── 这个方案是否是最简单的？
├── 有没有遗漏边界情况？
├── 我是否在重复之前失败的尝试？
├── 是否需要更多信息才能继续？
└── 风险是否可接受？

重要性: 防止 Agent 冲动行动，减少无效迭代
```

#### Phase 3: Action (工具调用)

```
目的: 执行具体操作

流程:
1. LLM 生成 tool_call (工具名 + 参数)
2. Harness 检查安全层 (5层纵深防御)
3. 如需审批 → 请求用户确认
4. 执行工具
5. 收集结果
6. 将结果追加到对话历史

并行调用: 多个独立工具可并行执行
  例: 同时 Read 多个文件、同时 Grep 多个模式
```

#### Phase 4: Verification (验证)

```
目的: 确认操作结果正确

自动验证:
├── 运行测试 (如果存在)
├── 类型检查
├── Lint 检查
├── 输出与预期对比
└── 状态一致性检查

人类验证:
├── 展示 diff 供审查
├── 请求确认关键操作
└── 可交互式修正
```

## Doom Loop 检测与处理

Agent 可能陷入的死循环场景：

```
场景 1: 反复编辑同一文件
┌─────────────────────────────────────┐
│ Edit file.js → Test fails           │
│ Edit file.js → Test fails           │
│ Edit file.js → Test fails           │
│ 检测: 同一文件编辑次数 > 阈值         │
│ 处理: 注入 "reconsider your approach"│
└─────────────────────────────────────┘

场景 2: 重复相同的工具调用
┌─────────────────────────────────────┐
│ Search("foo") → no results          │
│ Search("foo") → no results          │
│ 检测: 相同调用模式重复               │
│ 处理: 建议尝试不同的搜索策略         │
└─────────────────────────────────────┘

场景 3: 安全上限
┌─────────────────────────────────────┐
│ 总迭代次数 > MAX_ITERATIONS         │
│ 处理: 强制停止 + 报告当前状态        │
└─────────────────────────────────────┘
```

## 终止条件

Agent Loop 在以下条件下终止：

| 条件 | 类型 | 说明 |
|------|------|------|
| 模型返回纯文本（无工具调用） | 正常 | 模型认为任务完成 |
| 达到最大迭代次数 | 安全 | 防止无限循环 |
| 检测到死循环 | 安全 | 多次重复无效操作 |
| 用户中断 | 人工 | Ctrl+C 或 ESC |
| Token 预算耗尽 | 资源 | 上下文窗口已满且无法压缩 |
| 工具执行超时 | 异常 | 单个工具执行超时 |

## 实际案例：Claude Code 的 Agent Loop

```
用户: "修复 auth.ts 中的 bug"

Iteration 1:
  Think: 需要先了解 bug 的具体表现
  Act: Read("auth.ts")

Iteration 2:
  Think: 看到了问题——token 验证逻辑有错误
  Act: Grep("verifyToken", glob="**/*.ts")

Iteration 3:
  Think: 找到了所有调用点，确认只需修改 auth.ts
  Act: Edit("auth.ts", old="...", new="...")

Iteration 4:
  Think: 修改完成，需要验证
  Act: Bash("npm test")

Iteration 5:
  Think: 测试通过，任务完成
  Response: "已修复 auth.ts 中的 token 验证 bug，所有测试通过。"
  [Loop 终止 - 模型返回纯文本]
```

## SubAgent 模式

Agent 可以产生子 Agent，形成树状执行：

```
Main Agent Loop
├── SubAgent: Research (只读工具)
│   └── Agent Loop: Search → Read → Summarize
├── SubAgent: Code Review (只读工具)
│   └── Agent Loop: Read → Analyze → Report
└── SubAgent: Implementation (完整工具)
    └── Agent Loop: Edit → Test → Fix → Commit
```

**关键设计**: Schema-Level Isolation
- 子 Agent 只能看到被授予的工具 Schema
- 不在 Schema 中的工具 = 不可调用
- 比运行时权限检查更安全

## 下一步

→ 阅读 [04-tool-integration.md](./04-tool-integration.md) 了解工具集成的详细设计
