# 工具集成层

## 工具是 Agent 的"手脚"

如果 LLM 是大脑，工具就是 Agent 与外部世界交互的接口。没有工具的 Agent 只能"说"，有了工具的 Agent 才能"做"。

## 工具的分类体系

```
┌──────────────────────────────────────────────────────┐
│                   Tool Taxonomy                       │
│                                                      │
│  ┌─── 文件操作 ────┐  ┌─── 代码分析 ────┐            │
│  │ Read  (读取)     │  │ Grep  (内容搜索) │            │
│  │ Write (写入)     │  │ Glob  (文件搜索) │            │
│  │ Edit  (编辑)     │  │ LSP   (语义分析) │            │
│  │ LS    (列目录)   │  │ AST   (语法树)   │            │
│  └─────────────────┘  └─────────────────┘            │
│                                                      │
│  ┌─── 执行环境 ────┐  ┌─── 网络交互 ────┐            │
│  │ Bash  (Shell)    │  │ WebFetch (抓取)  │            │
│  │ 沙箱隔离         │  │ WebSearch(搜索)  │            │
│  │ 超时保护         │  │ API 调用        │            │
│  └─────────────────┘  └─────────────────┘            │
│                                                      │
│  ┌─── 人机交互 ────┐  ┌─── 编排控制 ────┐            │
│  │ AskQuestion      │  │ Agent (子Agent)  │            │
│  │ PlanMode         │  │ Task (任务管理)  │            │
│  │ TodoWrite        │  │ Skill (技能调用) │            │
│  └─────────────────┘  └─────────────────┘            │
│                                                      │
│  ┌─── 外部工具 (MCP) ───────────────────┐            │
│  │ GitHub, Supabase, Vercel, Slack...    │            │
│  │ 按需发现、延迟加载                     │            │
│  └───────────────────────────────────────┘            │
└──────────────────────────────────────────────────────┘
```

## Tool Registry 设计模式

```
class ToolRegistry:
    tools: Dict[str, ToolHandler]

    def register(name, handler, schema):
        """注册工具: 名称 → 处理器 + JSON Schema"""

    def dispatch(tool_call):
        """调度: 接收 LLM 的 tool_call → 路由到处理器 → 返回结果"""

    def get_schemas(context):
        """获取当前可用工具的 Schema 列表"""
        """可根据上下文动态调整可用工具"""
```

### Schema 设计原则

工具的 JSON Schema 直接影响模型使用工具的质量：

```json
{
  "name": "Edit",
  "description": "对文件执行精确字符串替换...",
  "parameters": {
    "file_path": {
      "type": "string",
      "description": "文件的绝对路径 (不是相对路径)"
    },
    "old_string": {
      "type": "string",
      "description": "要替换的文本 (必须在文件中唯一)"
    },
    "new_string": {
      "type": "string",
      "description": "替换后的文本 (必须与 old_string 不同)"
    }
  }
}
```

**Anthropic 的 SWE-bench 团队花在优化工具定义上的时间比优化整体 prompt 还多。**

关键原则：
- 使用绝对路径而非相对路径
- 描述要精确，包含约束条件
- 提供失败时的处理建议
- 应用 Poka-yoke (防错) 原则

## MCP: Model Context Protocol

### 什么是 MCP

MCP 是 Anthropic 发起的开放标准，类似工具连接的 "USB-C 接口"：

```
传统方式:                    MCP 方式:
每个工具需要自定义集成       统一协议，即插即用

Agent ──custom──> GitHub     Agent ──MCP──> GitHub
Agent ──custom──> DB         Agent ──MCP──> DB
Agent ──custom──> Slack      Agent ──MCP──> Slack
(N 个集成 = N 种代码)        (N 个集成 = 1 种协议)
```

### MCP 传输类型

| 类型 | 适用场景 |
|------|----------|
| **stdio** | 本地进程通信 (最常用) |
| **SSE** | 服务端推送事件 |
| **HTTP** | RESTful API |
| **WebSocket** | 双向实时通信 |

### 延迟加载策略

MCP 工具不是全部预加载，而是按需发现：

```
1. Harness 启动时注册 MCP Server 连接
2. 模型需要工具时，通过关键字评分找到匹配的 MCP 工具
3. 首次使用时才真正加载工具 Schema
4. 后续调用复用已加载的连接

优势:
• 避免系统膨胀 (可能有数百个 MCP 工具)
• 减少 prompt 中的工具 Schema 数量
• 降低初始化时间
```

## 工具执行安全

### 沙箱化执行

```
Shell 命令 (Bash 工具):
┌────────────────────────────────┐
│ 检查层:                        │
│ 1. 危险命令黑名单              │
│    rm -rf /, dd if=...         │
│ 2. 工作目录限制                │
│ 3. 超时保护 (默认 2min)       │
│ 4. 输出大小限制                │
│ 5. 用户审批 (敏感操作)         │
└────────────────────────────────┘
```

### 文件操作安全

```
Edit 工具的安全设计:
├── 必须先 Read 才能 Edit (防止盲改)
├── old_string 必须唯一 (精确替换)
├── 不允许 old_string == new_string
├── 检测过时读取 (stale-read detection)
└── 模糊匹配容错 (fuzzy matching)
```

## 工具设计最佳实践

1. **职责单一**: 每个工具做一件事，做好
2. **描述精确**: Schema description 是给模型看的"文档"
3. **防错设计**: 使参数约束清晰，减少误用
4. **容错能力**: LLM 输出是概率性的，需要模糊匹配
5. **结果简洁**: 返回模型需要的最少信息
6. **可组合**: 简单工具可组合完成复杂任务
7. **可观测**: 每次工具调用都应该是可追踪的

## 下一步

→ 阅读 [05-context-engineering.md](./05-context-engineering.md) 了解上下文工程
