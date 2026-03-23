# 记忆与状态管理

## Agent 的记忆问题

LLM 天生是无状态的——每次 API 调用都是全新的。要让 Agent 跨多步、跨会话保持连续性，需要在 Harness 层实现记忆系统。

## 三层记忆架构

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌── Tier 1: Working Context (工作上下文) ───────────────┐   │
│  │ 生命周期: 当前对话/迭代                                │   │
│  │ 内容:                                                  │   │
│  │ ├── 当前对话消息历史                                   │   │
│  │ ├── 本轮工具调用和结果                                 │   │
│  │ ├── 模型内部推理状态                                   │   │
│  │ └── 临时计算结果                                       │   │
│  │ 管理: 上下文压缩和裁剪 (自动)                          │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌── Tier 2: Session State (会话状态) ───────────────────┐   │
│  │ 生命周期: 当前会话 (从启动到退出)                      │   │
│  │ 内容:                                                  │   │
│  │ ├── 任务列表和进度 (TaskList)                          │   │
│  │ ├── Git commits (代码变更的持久记录)                    │   │
│  │ ├── Progress files (结构化进度)                         │   │
│  │ ├── 临时文件和工作区状态                                │   │
│  │ └── Plan mode 中的计划                                 │   │
│  │ 管理: 任务工具 + 文件系统 + Git                        │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌── Tier 3: Long-term Memory (长期记忆) ────────────────┐   │
│  │ 生命周期: 跨会话、跨时间持久                            │   │
│  │ 内容:                                                  │   │
│  │ ├── 用户偏好和反馈 (user/feedback memory)              │   │
│  │ ├── 项目知识 (project memory)                          │   │
│  │ ├── 外部资源索引 (reference memory)                    │   │
│  │ ├── CLAUDE.md (项目指导)                               │   │
│  │ └── MEMORY.md (记忆索引)                               │   │
│  │ 管理: 文件系统 + 结构化 Markdown                       │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## JSON vs Markdown

实践证明 JSON 比 Markdown 更适合存储结构化状态：

```
Markdown 的问题:
├── 模型可能意外覆盖或重新格式化
├── 解析不够精确 (歧义边界)
├── 嵌套结构难以表达
└── 追加更新容易破坏格式

JSON 的优势:
├── 结构明确，边界清晰
├── 模型不太可能意外重格式化
├── 支持嵌套和复杂数据
└── 程序化读写更可靠

实际选择:
├── 结构化数据 (进度、配置、状态) → JSON
├── 自由文本 (知识、笔记、说明) → Markdown
└── 混合 (YAML frontmatter + Markdown body) → 最佳实践
```

## Initializer-Executor 模式

Anthropic 推荐的跨会话状态管理模式：

```
┌──────────────────────────────────────────────────────┐
│ Session 1: Initializer (初始化器)                     │
│                                                      │
│ 1. 环境搭建                                          │
│    • 安装依赖、配置环境                               │
│    • 建立基线 commit                                  │
│                                                      │
│ 2. 任务分析                                          │
│    • 分析需求文档/Issue                               │
│    • 拆解为具体子任务                                 │
│                                                      │
│ 3. 状态文件创建                                      │
│    • progress.json: 任务列表和状态                     │
│    • features.json: 功能清单                          │
│    • baseline.txt: 基线信息                           │
│                                                      │
│ 输出: 结构化的进度文件 + 基线 commit                  │
└──────────────────────────┬───────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────┐
│ Session 2+: Executor (执行器)                         │
│                                                      │
│ 1. 状态恢复                                          │
│    • 读取 progress.json                              │
│    • 确认当前位置                                    │
│                                                      │
│ 2. 增量执行                                          │
│    • 从上次停止的地方继续                             │
│    • 完成一个子任务 → 更新 progress.json              │
│    • 提交代码变更                                    │
│                                                      │
│ 3. 状态更新                                          │
│    • 更新 progress.json 中的完成状态                  │
│    • 如果上下文满 → 保存状态 → 新会话继续             │
│                                                      │
│ 循环: 直到所有子任务完成                              │
└──────────────────────────────────────────────────────┘
```

### Progress File 示例

```json
{
  "task": "Implement user authentication",
  "created": "2026-03-23",
  "baseline_commit": "abc123",
  "subtasks": [
    {
      "id": 1,
      "description": "Create auth middleware",
      "status": "completed",
      "commit": "def456"
    },
    {
      "id": 2,
      "description": "Add JWT token validation",
      "status": "in_progress",
      "notes": "Token refresh logic pending"
    },
    {
      "id": 3,
      "description": "Write integration tests",
      "status": "pending"
    }
  ],
  "decisions": [
    "Using bcrypt for password hashing (compliance requirement)",
    "JWT tokens expire after 1 hour (security policy)"
  ]
}
```

## Artifact-Driven Continuity (基于制品的连续性)

```
Agent 的"记忆"不只存在于记忆系统中，还存在于：

Git Commits:
├── 每个有意义的变更都是一个 commit
├── Commit message 包含上下文 (why, not what)
└── git log 提供完整的工作历史

Progress Files:
├── JSON 格式的结构化进度
├── 新会话启动时首先读取
└── 提供快速上下文恢复

Feature Lists:
├── 待完成功能的清单
├── 勾选完成项
└── 优先级排序

Code Comments:
├── TODO: 标记待处理项
├── FIXME: 标记已知问题
└── 为后续会话留下线索
```

## 记忆的衰减与更新

```
记忆不是一劳永逸的——它们会过期:

问题:
├── 记忆中提到的文件可能已被删除
├── 记忆中的函数可能已被重命名
├── 项目决策可能已经改变
└── 人员变动可能使记忆过时

最佳实践:
├── 使用前验证记忆的当前有效性
├── 记忆中保存 "Why" 而非 "What"
│   (Why 比 What 更持久)
├── 将相对日期转换为绝对日期
├── 定期清理过时的记忆
└── 信任当前代码而非过去的记忆
```

## 下一步

→ 阅读 [07-safety-guardrails.md](./07-safety-guardrails.md) 了解安全与护栏设计
