# Phase 3: 进阶主题

## 学习目标

深入可靠性工程、可观测性和性能优化，从"能工作"到"可靠地工作"。

## Week 1: 可靠性工程

### Day 1-2: 纵深防御实现

**目标**: 为你的 Agent 实现完整的 5 层安全架构

```
实现计划:

Layer 1: Prompt Guardrails
├── 在 System Prompt 中嵌入安全规则
├── 定义 "绝对不能做" 清单
└── 添加 "做之前必须确认" 清单

Layer 2: Schema Restrictions
├── 为不同角色定义不同的工具集
├── Plan Mode: 只暴露 Read, Grep, Glob
├── Execute Mode: 完整工具集
└── SubAgent: 最小权限原则

Layer 3: Runtime Approval
├── 实现模式匹配审批引擎
├── 定义 Allow/Deny/Ask 规则
├── 记录所有审批决策
└── 支持批量审批同类操作

Layer 4: Tool Validation
├── 实现命令注入检测
├── 实现过期读取检测
├── 添加超时保护
└── 实现输出大小限制

Layer 5: Lifecycle Hooks
├── PreToolUse: 参数验证和修改
├── PostToolUse: 结果审计
├── Stop: 生成操作摘要
└── SessionStart: 环境检查
```

### Day 3-4: 错误处理与恢复

**关键模式**: 每个失败都应该产生永久的环境修复

```
实现重点:

1. 重试策略
├── 指数退避 (API 限流)
├── 参数调整重试 (工具调用失败)
├── 替代方案 (主路径失败 → 备用路径)
└── 最大重试次数限制

2. Doom Loop 检测器
class DoomLoopDetector:
    def __init__(self):
        self.file_edit_counts = {}
        self.tool_call_history = []
        self.max_same_file_edits = 5
        self.max_identical_calls = 3

    def check(self, tool_name, params):
        # 检查同一文件编辑次数
        if tool_name == "Edit":
            file_path = params["file_path"]
            self.file_edit_counts[file_path] = \
                self.file_edit_counts.get(file_path, 0) + 1
            if self.file_edit_counts[file_path] > self.max_same_file_edits:
                return "DOOM_LOOP: Too many edits to same file"

        # 检查重复调用模式
        call_sig = f"{tool_name}:{hash(str(params))}"
        recent = self.tool_call_history[-10:]
        if recent.count(call_sig) >= self.max_identical_calls:
            return "DOOM_LOOP: Repeated identical calls"

        self.tool_call_history.append(call_sig)
        return None  # OK

3. 优雅降级
├── 工具超时 → 简化请求重试
├── 上下文溢出 → 压缩后继续
├── API 错误 → 缓存结果 + 跳过
└── 无法修复 → 清晰报告 + 保存进度
```

### Day 5-7: 可观测性

**目标**: 看见 Agent 内部的行为

```
实现 Tracing 系统:

Trace 结构:
{
  "session_id": "abc123",
  "task": "Fix auth bug",
  "iterations": [
    {
      "iteration": 1,
      "thinking": "Need to understand the bug...",
      "tool_calls": [
        {
          "tool": "Read",
          "params": {"file_path": "/src/auth.ts"},
          "duration_ms": 45,
          "result_summary": "File read, 234 lines",
          "tokens_used": 1200
        }
      ],
      "tokens_total": 5400,
      "duration_ms": 2300
    },
    ...
  ],
  "total_iterations": 5,
  "total_tokens": 28000,
  "total_duration_ms": 15600,
  "outcome": "success"
}

分析维度:
├── 每次迭代的耗时和 token
├── 工具调用频率和成功率
├── Doom Loop 触发次数
├── 上下文压缩频率
├── 安全层拦截次数
└── 任务完成率
```

## Week 2: 性能优化

### Day 1-2: Prompt Cache 优化

```
优化策略:

1. 测量基线
├── 记录每次 API 调用的缓存命中情况
├── 计算缓存命中率
└── 识别缓存失效的原因

2. 优化 Prompt 结构
├── 静态内容移到前面 (System Prompt, Tool Schemas)
├── 动态内容放后面 (对话历史, 用户消息)
├── 避免不必要地重组静态内容
└── 使用 cache control breakpoints (如果 API 支持)

3. 量化效果
├── 延迟减少: 约 30-50% (对大型 prompt)
├── 成本减少: 缓存命中的 token 价格更低
└── 持续监控缓存命中率
```

### Day 3-4: 多 Agent 编排

```
实现并行 SubAgent:

场景: 代码库分析

Main Agent:
├── 分析任务 → 拆解为 3 个独立子任务
├── 并行产生 3 个 SubAgent:
│   ├── Agent A: 搜索 API 端点
│   ├── Agent B: 分析数据库 Schema
│   └── Agent C: 检查依赖版本
├── 等待所有完成
└── 汇总结果

实现要点:
├── 并发控制: 最大同时运行的 SubAgent 数量
├── 超时处理: 单个 SubAgent 超时不影响其他
├── 结果合并: 设计统一的结果格式
├── 上下文隔离: 每个 SubAgent 有独立上下文
└── 错误隔离: 一个 Agent 失败不影响其他
```

### Day 5-7: 综合项目

**构建一个多功能 Dev Agent**

```
项目: Full-Stack Dev Agent

能力矩阵:
├── 代码修改: Read → Edit → Write
├── 测试执行: Bash("npm test")
├── 代码审查: Grep → Analyze → Report
├── Git 操作: Bash("git ...") (有安全限制)
└── 文档生成: Read → Analyze → Write

Harness 组件:
├── Agent Loop: Extended ReAct
├── 工具: 完整工具集 + MCP
├── 安全: 5 层纵深防御
├── 记忆: 3 层 + Progress File
├── 上下文: 动态组装 + 自适应压缩
├── 验证: 测试套件 + Lint + 类型检查
├── 观测: 完整 Tracing
└── 优化: Prompt Cache + 并行 SubAgent

评估指标:
├── 任务完成率
├── 首次正确率 (减少迭代次数)
├── 平均迭代次数
├── Token 效率 (每成功任务的 token)
├── 安全拦截统计
└── Doom Loop 触发率
```

## Phase 3 自检清单

```
□ 实现了完整的 5 层安全架构
□ 能检测和处理 Agent 死循环
□ 实现了工具执行的重试和降级策略
□ 实现了 Agent 行为 Tracing 系统
□ 能分析 Trace 数据找到优化点
□ 优化了 Prompt Cache 命中率
□ 实现了多 SubAgent 并行编排
□ 构建了一个多能力 Agent
□ 建立了评估指标体系
□ 理解了 "Harness 决定可靠性" 的深层含义
```

## 进入 Phase 4 的信号

当你的 Agent 能可靠地完成复杂任务，并且你能通过 Trace 数据分析和改进它的行为时，你就可以进入 Phase 4 了。
