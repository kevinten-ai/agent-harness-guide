# 上下文工程 (Context Engineering)

## 为什么上下文工程如此重要

> "Prompt Engineering 是写一条好消息给模型；Context Engineering 是构建一个完整的信息系统给模型。"

Agent 的行为质量直接取决于它"看到"了什么。上下文工程是 Harness 工程中与 LLM 行为交叉最深的领域。

## 上下文工程的三大支柱

```
                Context Engineering
                       │
          ┌────────────┼────────────┐
          │            │            │
    ┌─────▼─────┐ ┌───▼────┐ ┌────▼─────┐
    │ 构建       │ │ 管理    │ │ 优化      │
    │ (Assembly) │ │(Manage) │ │(Optimize) │
    └─────┬─────┘ └───┬────┘ └────┬─────┘
          │            │            │
    • Prompt 组装  • 压缩策略    • Prompt Cache
    • 工具 Schema  • Token 预算  • 批量调用
    • 环境注入     • Reminder    • 选择性加载
    • 动态裁剪     • 卸载存储    • 前缀匹配
```

## 一、System Prompt 动态组装

### 优先级分层

System Prompt 不是一个静态字符串，而是按优先级动态组装的模块：

```
Priority  Module                    内容
────────  ──────────────────────    ────────────────────────
10-30     Identity & Safety         身份定义、安全策略
                                    "You are Claude Code..."
                                    "NEVER run rm -rf..."

40-50     Tool Definitions          工具名称、Schema、使用指南
                                    所有可用工具的完整定义

55-65     Code Standards            项目标准 (CLAUDE.md)
                                    编码规范、测试要求

70-80     Provider Specific         模型特定指令
                                    针对不同模型的优化提示

85-95     Environment Info          工作目录、git 状态
                                    平台信息、Shell 环境
                                    当前日期、模型信息
```

**设计原则**: 高优先级内容放在 prompt 前部，低优先级放后部。这样当上下文压缩时，高优先级内容更可能被保留。

### 动态裁剪

不是所有工具都需要放在每次调用的 prompt 中：

```
场景: 100+ MCP 工具可用

策略:
├── 内置工具: 始终包含 (Read, Write, Edit, Bash...)
├── 常用 MCP: 按使用频率包含
├── 其他 MCP: 仅 Deferred (只有名字)
└── 按需展开: 模型请求时才加载完整 Schema

实现:
ToolSearch 工具 → 模型搜索关键字 → 返回匹配的完整 Schema
```

## 二、自适应上下文管理

### Token Budget Monitor

```
┌─────────────────────────────────────────────┐
│ Token Budget = Context Window - Reserved     │
│                                              │
│ 例: 1M tokens window - 10K reserved          │
│     = 990K available for conversation        │
│                                              │
│ 监控策略:                                     │
│ ├── < 70%: 正常运行                           │
│ ├── 70-85%: 开始温和压缩                      │
│ ├── 85-95%: 积极压缩                          │
│ └── > 95%: 强制压缩 / 会话分割                │
└─────────────────────────────────────────────┘
```

### 渐进式压缩策略

```
Stage 1: 工具输出截断
├── 长文件只保留关键行
├── 测试输出只保留 pass/fail 摘要
└── 搜索结果只保留前 N 项

Stage 2: 历史对话摘要
├── 早期对话轮次 → 简短摘要
├── 保留关键决策点
└── 保留错误和修复记录

Stage 3: 记忆卸载
├── 项目知识 → 持久存储 (MEMORY.md)
├── 中间状态 → Progress files
└── 大型上下文 → 子 Agent 隔离

Stage 4: 历史裁剪
├── 删除冗余的探索过程
├── 只保留最终结果
└── 保留最近 N 轮完整对话
```

### System Reminders (系统提醒)

对抗长对话中的"指令衰减"现象：

```
问题: 模型在长对话中会逐渐"忘记"早期的指令

解决: System Reminders — 事件驱动的提示注入

触发时机:
├── 每 N 轮对话自动注入
├── 特定工具调用后注入
├── 检测到偏离行为时注入
└── 重要状态变化时注入

示例:
<system-reminder>
  记住：修改文件前必须先读取。
  当前工作目录: /Users/user/project
  可用 Skills: commit, review-pr, ...
</system-reminder>
```

## 三、Prompt Cache 优化

### 工作原理

```
Prompt 结构 (为缓存优化):

┌─────────────────────────────────────┐
│ 静态前缀 (不变)                      │ ← 缓存命中区
│ ├── System Prompt                    │
│ ├── Tool Schemas                     │
│ └── 项目标准                         │
├─────────────────────────────────────┤
│ 动态后缀 (每次变化)                   │ ← 每次新计算
│ ├── 对话历史                         │
│ ├── 最新用户消息                     │
│ └── 环境状态更新                     │
└─────────────────────────────────────┘

效果:
• 静态前缀命中缓存 → 大幅减少计算
• 首次调用: 完整计算
• 后续调用: 只计算动态部分 (更快更便宜)
```

### 优化技巧

1. **稳定的前缀**: System Prompt 的前 N 个 token 尽量保持不变
2. **变化的后缀**: 把频繁变化的内容放在 prompt 最后
3. **缓存断点**: 某些 API 支持显式标记缓存断点
4. **避免不必要的变更**: 不要在每次调用时重新格式化静态内容

## 四、上下文工程与 Agent 质量

### LangChain 的实证

LangChain 在 Terminal Bench 2.0 上的改进全部来自上下文工程：

```
优化前: 排名 #30
优化后: 排名 #5 (+13.7%)
模型: 未更换

具体优化:
1. LocalContextMiddleware: 更好的本地上下文映射
2. ReasoningSandwichMiddleware: 规划和验证阶段分配更多推理资源
3. PreCompletionChecklistMiddleware: 完成前强制检查

核心洞见: Reasoning Sandwich (推理三明治)
├── 高推理力度用于: 规划 + 最终验证
├── 标准推理力度用于: 实施执行
└── 全程高推理反而降低性能 (超时+过度思考)
```

### Anthropic 的指导

```
System Prompt 组装原则:
1. 最重要的指令放最前面
2. 工具描述要精确、完整
3. 包含"做什么"和"不做什么"
4. 提供失败时的处理策略
5. 使用结构化格式 (而非自由文本)
6. 定期注入 Reminders 对抗衰减
```

## 实战模式

### 模式：增量上下文加载

```
不要一次性加载所有信息!

错误方式:
  Read(file1) + Read(file2) + ... + Read(file100) → 消耗全部上下文

正确方式:
  Step 1: Glob("**/*.ts") → 了解文件结构
  Step 2: Read(关键文件) → 理解核心逻辑
  Step 3: Grep(关键词) → 定位相关代码
  Step 4: Read(相关部分) → 深入了解

  只在需要时加载，而不是预加载一切
```

### 模式：上下文隔离

```
主 Agent 上下文即将满 → 产生 SubAgent

SubAgent:
├── 继承: 任务描述 + 关键上下文
├── 独立: 有自己的上下文窗口
├── 返回: 精简的结果摘要
└── 释放: 完成后回收上下文

效果: 主 Agent 的上下文保持精简
```

## 下一步

→ 阅读 [06-memory-state.md](./06-memory-state.md) 了解记忆与状态管理
