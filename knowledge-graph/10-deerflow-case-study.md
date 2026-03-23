# DeerFlow 2.0 深度案例研究

## 一句话定位

> **DeerFlow 2.0** (Deep Exploration and Efficient Research Flow) 是字节跳动开源的 **SuperAgent Harness**，不是一个 Agent Framework，而是 Agent 的**运行时基础设施**。

## 基本信息

| 维度 | 信息 |
|------|------|
| **发布者** | 字节跳动 (ByteDance) |
| **发布时间** | 2026 年 2 月 27 日 |
| **GitHub** | github.com/bytedance/deer-flow |
| **Stars** | ~37,500 (2026.3) |
| **License** | MIT |
| **定位** | Open-source SuperAgent Harness |
| **技术栈** | Python 3.12+ / Node.js 22+ / LangGraph / FastAPI / Next.js |

发布 24 小时内登顶 **GitHub Trending #1**。

## 为什么它是 Harness 而不是 Framework？

```
关键类比:

LangChain (Framework)          DeerFlow (Harness)
═══════════════════            ═══════════════════
提供抽象给"推理"               提供基础设施给"执行"
路由、模板、链                  沙箱、记忆、编排
"怎么想"                       "怎么跑"
类似: Web 框架                 类似: Docker + Kubernetes

DeerFlow 不关心你用什么模型思考,
它关心你的 Agent 在什么环境中安全地执行、
如何跨会话记忆、如何编排子任务。
```

---

## 架构总览

### 微服务拓扑

```
                    用户
                     │
                     ▼
              ┌─────────────┐
              │   Nginx      │ :2026  (统一入口)
              │  反向代理     │
              └──────┬──────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Frontend │ │ Gateway  │ │ LangGraph│
  │ Next.js  │ │ FastAPI  │ │ Server   │
  │ :3000    │ │ :8001    │ │ :2024    │
  │          │ │          │ │          │
  │ Web UI   │ │ 配置/记忆 │ │ Agent    │
  │ SSE 流   │ │ /模型/技能│ │ 执行引擎  │
  └──────────┘ └──────────┘ └────┬─────┘
                                  │
                           ┌──────▼──────┐
                           │ Provisioner │ :8002 (可选)
                           │ K8s 沙箱管理 │
                           └─────────────┘
```

### 严格的双层分离 (Harness/App Boundary)

```
┌─────────────────────────────────────────┐
│  Harness Layer (deerflow.*)             │ ← 可独立 pip install 的核心
│  ├── Agent Loop (LangGraph StateGraph)  │
│  ├── Middleware Pipeline (14 阶段)       │
│  ├── Tool System (5 来源动态组装)        │
│  ├── Memory System (置信度事实)          │
│  ├── Skills System (渐进式加载)          │
│  └── Sandbox Management (3 种模式)      │
├─────────────────────────────────────────┤
│  App Layer (app.*)                      │ ← 部署特定代码
│  ├── Gateway API (FastAPI)              │
│  ├── IM 集成 (Telegram/Slack/飞书)      │
│  └── Web UI (Next.js)                  │
└─────────────────────────────────────────┘

强制边界: tests/test_harness_boundary.py
• 用 Python ast.parse() 扫描所有 .py 文件
• 断言 Harness 层没有任何 "from app.*" / "import app.*"
• CI 自动执行,违反即失败
• 意义: deerflow-harness 可独立发布为 pip 包
```

---

## 14 阶段中间件管道 (核心创新)

源码位置: `deerflow/agents/lead_agent/agent.py` → `_build_middlewares()`

每一层解决一个**生产环境中真实遇到的问题**：

```
请求进入
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│                14-Stage Middleware Pipeline                    │
│              (条件性包含,按严格顺序执行)                        │
│                                                              │
│  ┌─ before_agent hooks ──────────────────────────────────┐   │
│  │                                                        │   │
│  │  ① ThreadData         线程目录初始化                    │   │
│  │     创建 threads/{id}/user-data/{workspace,uploads,    │   │
│  │     outputs} 目录。必须第一个,后续中间件依赖 thread_id  │   │
│  │                                                        │   │
│  │  ② Uploads            文件上传注入                      │   │
│  │     扫描 uploads 目录,注入 <uploaded_files> 块          │   │
│  │     到对话消息中                                        │   │
│  │                                                        │   │
│  │  ③ Sandbox            沙箱获取/释放                     │   │
│  │     before: acquire() → 获取 sandbox_id                │   │
│  │     after:  release() → 释放沙箱资源                    │   │
│  │                                                        │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─ before_model hooks ─────────────────────────────────┐    │
│  │                                                       │    │
│  │  ④ DanglingToolCall   悬挂调用修复                     │    │
│  │     问题: 用户中断后,历史中有 tool_call 但无 result    │    │
│  │     方案: 注入占位 ToolMessage,防止模型崩溃            │    │
│  │                                                       │    │
│  │  ⑤ Guardrail          工具调用授权                     │    │
│  │     三种 Provider: Allowlist / OAP / 自定义            │    │
│  │     拒绝时返回错误 ToolMessage                         │    │
│  │                                                       │    │
│  │  ⑥ Summarization      上下文压缩                      │    │
│  │     触发条件: token 数 / 消息数 / 占比 (OR 逻辑)       │    │
│  │     保留最近 N 条,旧消息 LLM 摘要                      │    │
│  │                                                       │    │
│  │  ⑦ ViewImage          图片处理                         │    │
│  │     条件: 模型 supports_vision=true                    │    │
│  │     将图片转 base64 注入 ViewedImage                   │    │
│  │                                                       │    │
│  │  ⑧ DeferredToolFilter  延迟工具过滤                    │    │
│  │     条件: tool_search.enabled=true                     │    │
│  │     隐藏 MCP 工具 Schema,通过 tool_search 按需发现     │    │
│  │                                                       │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─ System Prompt 注入 ─────────────────────────────────┐    │
│  │                                                       │    │
│  │  ⑨ Todo              任务列表管理                      │    │
│  │     条件: is_plan_mode=True                            │    │
│  │     注入 write_todos 工具 + 任务跟踪系统提示           │    │
│  │     强制: 同一时间只有一个任务 in_progress              │    │
│  │                                                       │    │
│  │  ⑩ Memory            记忆注入                          │    │
│  │     注入 Top 15 高置信度事实 + 上下文摘要              │    │
│  │     通过 <memory> 标签加入 System Prompt               │    │
│  │                                                       │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                              │
│                    LLM 推理                                   │
│                                                              │
│  ┌─ after_model hooks ──────────────────────────────────┐    │
│  │                                                       │    │
│  │  ⑪ SubagentLimit     子 Agent 并发控制                 │    │
│  │     条件: subagent_enabled=true                        │    │
│  │     计数 task 调用,超过 max_concurrent(默认3) 则截断   │    │
│  │     范围限制: [2, 4]                                   │    │
│  │                                                       │    │
│  │  ⑫ LoopDetection     死循环检测 (P0 安全)             │    │
│  │     MD5 哈希每组 tool_calls (名称+参数,顺序无关)       │    │
│  │     滑动窗口: 20 (LRU 淘汰, 最大 100 线程)            │    │
│  │     3 次相同 → 注入警告 SystemMessage                  │    │
│  │     5 次相同 → 强制停止,剥离所有 tool_calls            │    │
│  │                                                       │    │
│  │  ⑬ Clarification     澄清问题拦截 (必须最后!)         │    │
│  │     拦截 ask_clarification 工具调用                    │    │
│  │     Command(goto=END) 中断工作流,返回控制给用户        │    │
│  │                                                       │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─ after_agent hooks ──────────────────────────────────┐    │
│  │                                                       │    │
│  │  ⑭ Title             会话标题生成                      │    │
│  │     首次完整对话后自动生成                              │    │
│  │     限制: 最多 6 词 / 60 字符                          │    │
│  │                                                       │    │
│  └───────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**关键设计决策**：中间件列表是**动态构建**的。`_build_middlewares()` 根据运行时配置条件性地包含各层。这意味着：
- 不支持视觉的模型不会加载 ViewImage
- 不开启 plan_mode 不会加载 Todo
- 不开启子 Agent 不会加载 SubagentLimit

---

## Agent Loop 执行引擎

### 入口点

```python
# langgraph.json 注册
make_lead_agent(config: RunnableConfig)

# 构建流程:
1. _resolve_model_name()     → 模型选择 (请求覆盖 > agent 配置 > 全局默认)
2. 检查模型能力              → supports_thinking, supports_vision
3. get_available_tools()      → 5 来源动态组装工具
4. _build_middlewares()       → 14 阶段中间件链
5. apply_prompt_template()    → 技能、记忆、子Agent 指令
6. create_agent()             → 编译为 LangGraph StateGraph
```

### 状态定义 (ThreadState)

```python
class ThreadState(AgentState):
    messages: list[BaseMessage]    # 核心对话
    sandbox: SandboxState          # 沙箱 ID 和引用
    thread_data: ThreadDataState   # workspace/uploads/outputs 路径
    title: str | None              # 自动生成的标题
    artifacts: list[str]           # 生成的文件路径 (去重)
    todos: list[dict]              # Plan Mode 任务列表
    uploaded_files: list[str]      # 用户文件引用
    viewed_images: dict            # 视觉模型预处理的图片
```

### 执行循环

```
用户消息
    │
    ▼
[before_agent hooks: ThreadData → Uploads → Sandbox]
    │
    ▼
┌────────────────────────────────────────────┐
│           LLM 调用循环                      │
│                                            │
│  [before_model hooks]                      │
│  ④ DanglingToolCall → ⑤ Guardrail →       │
│  ⑥ Summarization → ⑦ ViewImage →          │
│  ⑧ DeferredToolFilter                     │
│      │                                     │
│      ▼                                     │
│  LLM 推理 (+ 技能/记忆/工具 Schema)        │
│      │                                     │
│      ▼                                     │
│  [after_model hooks]                       │
│  ⑪ SubagentLimit → ⑫ LoopDetection →     │
│  ⑬ Clarification                          │
│      │                                     │
│      ├── 有 tool_calls → 执行工具 → 循环   │
│      │                    回到 before_model │
│      │                                     │
│      └── 无 tool_calls → 退出循环          │
│                                            │
└────────────────────────────────────────────┘
    │
    ▼
[after_agent hooks: ⑭ Title + ⑩ Memory(队列)]
    │
    ▼
SSE 流式返回给前端
```

---

## Sub-Agent 系统详解

### 双线程池架构

```python
# 源码: deerflow/subagents/executor.py

_scheduler_pool = ThreadPoolExecutor(
    max_workers=3,
    thread_name_prefix="subagent-scheduler-"
)
_execution_pool = ThreadPoolExecutor(
    max_workers=3,
    thread_name_prefix="subagent-exec-"
)
```

**为什么需要两个线程池？**

```
问题: 如何在不阻塞执行的情况下强制超时?

方案:
┌─────────────┐         ┌─────────────┐
│ Scheduler   │ 提交    │ Execution   │
│ Pool (3)    │────────►│ Pool (3)    │
│             │         │             │
│ 负责:       │         │ 负责:       │
│ • 生命周期   │         │ • 实际运行   │
│ • 超时控制   │         │ • Agent Loop │
│ • 状态管理   │         │ • 工具调用   │
│             │         │             │
│ future.result│         │             │
│ (timeout=   │         │             │
│  900s)      │         │             │
└─────────────┘         └─────────────┘

Scheduler 线程用 future.result(timeout) 等待
如果超时 → 可以取消执行线程,不影响其他任务
如果完成 → 收集结果返回
```

### 后端轮询 vs 传统方式

```
传统方式 (浪费 token):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Lead Agent → LLM: "调 task()"        消耗 token ✓
Lead Agent → LLM: "调 task_status()" 消耗 token ✗ 浪费!
Lead Agent → LLM: "调 task_status()" 消耗 token ✗ 浪费!
Lead Agent → LLM: "调 task_status()" 消耗 token ✗ 浪费!
Lead Agent → LLM: "结果来了,继续"     消耗 token ✓

DeerFlow 方式 (高效):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Lead Agent → LLM: "调 task()"        消耗 token ✓
  task() 内部:
    ├── 提交到 scheduler_pool
    ├── 每 5 秒轮询 get_background_task_result()  ← 后端, 不消耗 token
    ├── 发送 SSE 进度事件给前端
    └── 完成后返回结果
Lead Agent → LLM: "结果来了,继续"     消耗 token ✓

节省: N 次 LLM 调用 (N = 等待秒数 / 轮询间隔)
```

### Sub-Agent 配置

```yaml
# config.yaml
subagents:
  timeout_seconds: 900          # 默认 15 分钟
  agents:
    general-purpose:
      timeout_seconds: 1800     # 可按类型覆盖
      # 工具: 所有工具 EXCEPT task (不能递归委派)
    bash:
      # Shell 专家
```

```
Sub-Agent 继承关系:
├── 继承: 父 Agent 的沙箱 (同一文件系统)
├── 继承: 父 Agent 的线程数据
├── 不继承: 完整工具集 (按 allowed/disallowed 过滤)
├── 不继承: 父 Agent 的上下文历史
├── 特殊: model="inherit" 使用父模型,否则用指定模型
└── 限制: 子 Agent 的 thinking 被禁用
```

---

## 记忆系统实现

### 数据结构 (memory.json)

```json
{
  "version": "1.0",
  "lastUpdated": "2026-03-23T...",
  "user": {
    "workContext":     {"summary": "...", "updatedAt": "..."},
    "personalContext": {"summary": "...", "updatedAt": "..."},
    "topOfMind":       {"summary": "...", "updatedAt": "..."}
  },
  "history": {
    "recentMonths":      {"summary": "...", "updatedAt": "..."},
    "earlierContext":     {"summary": "...", "updatedAt": "..."},
    "longTermBackground": {"summary": "...", "updatedAt": "..."}
  },
  "facts": [
    {
      "id": "fact_a1b2c3d4",
      "content": "User works with LangGraph and Python 3.12",
      "category": "knowledge",
      "confidence": 0.9,
      "createdAt": "2026-03-20T...",
      "source": "thread_abc123"
    }
  ]
}
```

### 完整的记忆更新流水线

```
对话完成
    │
    ▼
① MemoryMiddleware (after_agent hook)
    │ 过滤消息:
    │ ├── 保留: 用户消息 + 最终 AI 回复
    │ ├── 删除: 工具调用、工具结果、<uploaded_files>
    │ └── 跳过: 纯上传消息 (无文本内容)
    │
    ▼
② MemoryUpdateQueue (单例)
    │ ├── 按 thread_id 去重 (新的覆盖旧的)
    │ └── threading.Timer 防抖 (默认 30 秒)
    │     每次新增重置计时器
    │
    ▼ (30 秒后触发)
③ MemoryUpdater.update_memory()
    │ ├── 加载当前 memory.json
    │ ├── format_conversation_for_update()
    │ │   └── 截断 >1000 字符的消息
    │ ├── 构建 MEMORY_UPDATE_PROMPT
    │ └── 调用 LLM (thinking 禁用)
    │
    ▼
④ LLM 返回 JSON:
    {
      "user": {
        "workContext": {"summary": "...", "shouldUpdate": true/false}
      },
      "newFacts": [
        {"content": "...", "category": "...", "confidence": 0.85}
      ],
      "factsToRemove": ["fact_id_1", "fact_id_2"]
    }
    │
    ▼
⑤ _apply_updates()
    ├── shouldUpdate=true 的部分才更新
    ├── 删除 factsToRemove 中的事实
    ├── 新事实过滤:
    │   ├── confidence >= 0.7 (阈值可配)
    │   ├── 内容去重 (normalize + set 比较)
    │   └── ID 生成: fact_{uuid4.hex[:8]}
    ├── 总数超 max_facts(100) → 按置信度排序,保留 Top N
    └── _strip_upload_mentions_from_memory()
        └── 正则删除所有关于文件上传的句子
    │
    ▼
⑥ 原子写入
    ├── 写入 .tmp 文件
    └── Path.replace() (大多数系统上原子)
```

### 置信度评分标准

```
LLM 按以下标准赋分:

0.9-1.0  显式陈述  "我的角色是 X" / "我在用 Y 框架"
0.7-0.8  强推断    从行为和讨论中明确推断
0.5-0.6  模式猜测  仅限非常清晰的模式 (谨慎使用)
< 0.5    不保存    低于阈值直接丢弃

分类: preference / knowledge / context / behavior / goal
```

### 记忆注入 (下次对话时)

```python
# format_memory_for_injection(memory_data, max_tokens=2000)

注入内容:
├── 用户上下文摘要 (bullet points)
├── 历史上下文摘要
└── 事实列表 (按置信度降序)
    每条: "- [category | 0.90] content text"

Token 控制:
├── 使用 tiktoken (cl100k_base) 精确计数
├── 渐进添加,直到耗尽 2000 token 预算
└── 作为 <memory> 标签注入 System Prompt

缓存: 基于 memory.json 的文件 mtime
如果文件未变 → 返回缓存,不重新读取
```

**vs Claude Code 的记忆:**

```
Claude Code:
├── 格式: Markdown 文件 (MEMORY.md + 独立 .md)
├── 索引: MEMORY.md 手动维护
├── 提取: Agent 自己决定何时保存
├── 注入: 启动时加载 MEMORY.md
└── 无置信度评分

DeerFlow:
├── 格式: JSON (memory.json)
├── 索引: 无需 (结构化数据)
├── 提取: 专用 LLM 异步提取 + 防抖
├── 注入: Top 15 高置信度 + token 预算控制
└── 有置信度评分 + 自动清理
```

---

## 技能系统

### 目录结构

```
skills/
├── public/              # 内置技能 (git 跟踪)
│   ├── pdf-processing/
│   │   └── SKILL.md
│   ├── frontend-design/
│   │   └── SKILL.md
│   ├── research/
│   │   └── SKILL.md
│   └── slides/
│       └── SKILL.md
└── custom/              # 用户安装 (.gitignore)
    └── my-workflow/
        └── SKILL.md
```

### SKILL.md 格式

```yaml
---
name: PDF Processing
description: Handle PDF documents efficiently
license: MIT
allowed-tools:
  - read_file
  - write_file
  - bash
---

# 技能指令
详细的工作流指令，在 Agent 使用时注入到 System Prompt...
```

### 渐进式加载 (两级)

```
Level 1: System Prompt 中 (始终)
    只有名称 + 描述 + 容器路径
    例: "PDF Processing - Handle PDF documents - /mnt/skills/public/pdf-processing/"

Level 2: 按需加载 (Agent 决定使用时)
    Agent 用 read_file 读取 /mnt/skills/.../SKILL.md
    获取完整指令

优势:
├── 不浪费上下文窗口 (可能有几十个技能)
├── 对 token 敏感的小模型特别友好
└── 类似 Claude Code 的 ToolSearch / Deferred Tools
```

### 安装

```
POST /api/skills/install
├── 接受 .skill ZIP 包
├── 解压到 skills/custom/
├── 验证 frontmatter 字段
└── 运行时自动发现
```

---

## 工具系统: 5 来源动态组装

```python
# get_available_tools(model_name, groups, subagent_enabled)
# 返回: loaded_tools + builtin_tools + mcp_tools

来源 1: Config 定义的工具 (loaded_tools)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
config.yaml tools[] 每项有 use 路径
例: "deerflow.community.tavily.tools:web_search_tool"
通过 importlib 动态加载
按 tool_groups 过滤

来源 2: 内置工具 (builtin_tools)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
├── present_file_tool   展示输出文件 (限 /mnt/user-data/outputs)
├── ask_clarification   请求用户澄清 (被 ClarificationMiddleware 拦截)
├── view_image_tool     查看图片 (条件: supports_vision)
└── task_tool           子 Agent 委派 (条件: subagent_enabled)

来源 3: 沙箱工具
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
├── bash          Shell 执行 (路径翻译 + 错误处理)
├── ls            目录列表 (tree 格式, 最多 2 层)
├── read_file     读文件 (可选行范围)
├── write_file    写文件 (自动创建目录)
└── str_replace   字符串替换 (单次或全部)

来源 4: MCP 工具 (mcp_tools)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
从 extensions_config.json 加载
支持 OAuth (client_credentials + refresh_token)
可选延迟加载: tool_search.enabled=true 时
  → 注册 DeferredToolRegistry
  → 添加 tool_search 元工具
  → 按需发现,不全部暴露

来源 5: 用户自定义工具
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
通过 config.yaml 的 tools[] 扩展
任何 LangChain Tool 兼容的 Python 对象
```

### MCP OAuth 实现

```
OAuthTokenManager (每 server 独立):
├── 支持: client_credentials / refresh_token
├── Token 缓存 + 过期追踪 (_OAuthToken.expires_at)
├── 自动刷新: 过期前 refresh_skew_seconds 内刷新
├── 并发安全: asyncio.Lock per server (双重检查锁定)
└── 启动预获取: get_initial_oauth_headers()

传输类型:
├── stdio  → 本地命令 (如 npx @mcp/server-github)
├── SSE    → Server-Sent Events
└── HTTP   → 标准 HTTP (支持 OAuth)
```

---

## 沙箱架构

### 三种模式

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌─── Local (开发) ────────────────────────────────────┐    │
│  │ LocalSandboxProvider (单例)                          │    │
│  │ sandbox_id = "local"                                │    │
│  │ 直接在宿主文件系统执行                                │    │
│  │ 路径翻译: 虚拟路径 → 物理路径                         │    │
│  │   replace_virtual_path()                            │    │
│  │   replace_virtual_paths_in_command()                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─── Docker (单机生产) ───────────────────────────────┐    │
│  │ AioSandboxProvider                                   │    │
│  │ 每个任务一个容器                                      │    │
│  │ LRU 淘汰 (达到 replicas 上限时)                      │    │
│  │ 配置: image, port, container_prefix, mounts, env    │    │
│  │ 虚拟路径直接 bind-mount 到容器中 (无需翻译)           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─── K8s (多租户) ───────────────────────────────────┐    │
│  │ AioSandboxProvider + provisioner_url                 │    │
│  │ 每个任务一个 Pod                                     │    │
│  │ 通过 Provisioner 服务 (port 8002) HTTP 管理          │    │
│  │ RemoteSandboxBackend 处理通信                        │    │
│  │ 完整的 Pod 级隔离                                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 统一虚拟文件系统

```
Agent 看到的 (所有模式统一):

/mnt/user-data/
├── workspace/     ← 工作区 (代码、项目文件)
├── uploads/       ← 用户上传的文件
└── outputs/       ← Agent 生成的产物

/mnt/skills/       ← 技能文件

物理映射 (Local 模式):
/mnt/user-data/workspace → backend/.deer-flow/threads/{id}/user-data/workspace
/mnt/user-data/uploads   → backend/.deer-flow/threads/{id}/user-data/uploads
/mnt/user-data/outputs   → backend/.deer-flow/threads/{id}/user-data/outputs

Docker/K8s 模式: 直接 bind-mount, 容器内路径就是虚拟路径
```

---

## 真实执行示例

**用户输入**: "写一份关于量子计算的研究报告"

```
① 请求到达 POST /api/langgraph/threads/{id}/runs/stream
   Nginx → LangGraph Server (port 2024)

② make_lead_agent(config) 构建 Agent
   模型: claude-3-5-sonnet
   工具: web_search + bash + read/write_file + task + present_file
   中间件: 14 阶段链
   System Prompt: 技能列表 + 记忆 + 子Agent 指令

③ before_agent hooks
   ThreadData: 创建 threads/{id}/user-data/{workspace,uploads,outputs}
   Uploads: 无文件
   Sandbox: acquire() → sandbox_id="local"

④ before_model hooks
   DanglingToolCall: 首条消息,无悬挂
   Summarization: token 少,不触发
   Memory: 注入 Top 15 事实 + 上下文摘要

⑤ LLM 推理: 决定委派 3 个子任务
   tool_calls:
   ├── task("研究量子计算基础原理", subagent_type="general-purpose")
   ├── task("研究 2025-2026 量子计算突破", subagent_type="general-purpose")
   └── task("研究量子计算产业应用", subagent_type="general-purpose")

⑥ after_model hooks
   SubagentLimit: 3 个 task,未超限 (max=3),通过
   LoopDetection: 首次出现,记录哈希

⑦ 工具执行: 3 个子 Agent 并行启动
   ┌─────────────────────────────────────────────────┐
   │ Scheduler Pool                                   │
   │ ├── Thread 1: Sub-Agent A (基础原理)             │
   │ │   → web_search → web_fetch → 整理 → 返回      │
   │ ├── Thread 2: Sub-Agent B (最新突破)             │
   │ │   → web_search → web_fetch → 整理 → 返回      │
   │ └── Thread 3: Sub-Agent C (产业应用)             │
   │     → web_search → web_fetch → 整理 → 返回      │
   └─────────────────────────────────────────────────┘

   task() 内部每 5 秒轮询,发送 SSE 进度事件

⑧ Lead Agent 综合: 收到 3 个子 Agent 结果
   → write_file("/mnt/user-data/outputs/quantum_report.md", 报告内容)
   → present_files("/mnt/user-data/outputs/quantum_report.md")

⑨ after_agent hooks
   Title: 生成 "量子计算研究报告"
   Memory: 过滤消息 → 队列 (30 秒后处理)

⑩ SSE 流式返回报告 + 文件链接

⑪ 30 秒后, 异步记忆更新:
   提取事实: "用户对量子计算研究感兴趣" (confidence: 0.8, category: context)
   原子写入 memory.json
```

---

## 配置系统 (config.yaml)

```yaml
config_version: 3

# 模型定义 (支持任意 OpenAI 兼容 API)
models:
  - name: default
    use: langchain_openai:ChatOpenAI
    model: gpt-4o
    api_key: $OPENAI_API_KEY
    supports_thinking: false
    supports_vision: true

# 沙箱
sandbox:
  use: deerflow.sandbox.local:LocalSandboxProvider  # 开发
  # use: deerflow.community.aio_sandbox:AioSandboxProvider  # 生产

# 子 Agent
subagents:
  timeout_seconds: 900
  agents:
    general-purpose:
      timeout_seconds: 1800

# 记忆
memory:
  enabled: true
  debounce_seconds: 30
  max_facts: 100
  fact_confidence_threshold: 0.7
  max_injection_tokens: 2000

# 上下文压缩
summarization:
  enabled: true
  trigger:
    - type: tokens
      value: 100000
  keep:
    type: messages
    value: 10

# 技能
skills:
  path: ./skills
  container_path: /mnt/skills

# 工具
tools:
  - name: web_search
    group: web
    use: deerflow.community.tavily.tools:web_search_tool

# MCP 延迟加载
tool_search:
  enabled: true

# 护栏
guardrails:
  enabled: true
  provider:
    use: deerflow.guardrails:AllowlistProvider

# 配置加载优先级:
# 1. config_path 参数
# 2. DEER_FLOW_CONFIG_PATH 环境变量
# 3. ./config.yaml
# 4. ../config.yaml
#
# 缓存 + 自动重载: 检查文件 mtime,修改后自动刷新
# 环境变量: 以 $ 开头的值自动从 os.getenv() 解析
```

---

## 关键启示

```
1. 中间件数量反映生产复杂度
   14 层不是过度工程，每层对应一个你在生产环境中
   一定会遇到的问题。如果你觉得用不上，说明你的
   Agent 还没到生产级别。

2. 后端轮询是经济学选择
   LLM 调用按 token 计费。每次 status 检查都是钱。
   把等待移到后端 = 直接省钱。

3. Harness 和 Framework 共存
   DeerFlow 底层用了 LangGraph (Framework)
   它不是要替代 LangChain，而是在其上构建 Harness 层。

4. 记忆系统的异步防抖是关键
   不能每次对话结束都同步更新记忆 (太慢)
   不能每条消息都更新 (太频繁)
   30 秒防抖 + 异步 = 最佳平衡

5. 架构边界需要测试来守护
   仅靠文档和规范无法阻止 import 违规
   test_harness_boundary.py 用 ast.parse() 自动检测
   = 架构即代码

6. 虚拟文件系统是沙箱透明化的关键
   Agent 始终看到 /mnt/user-data/...
   Local/Docker/K8s 的差异对 Agent 完全透明
```

---

## 学习资源

| 资源 | 链接 |
|------|------|
| GitHub 仓库 | github.com/bytedance/deer-flow |
| 架构深度解析 (DeepWiki) | deepwiki.com/bytedance/deer-flow |
| 多 Agent 工作流 | deepwiki.com/bytedance/deer-flow/2.1-multi-agent-workflow |
| 配置架构 | deepwiki.com/bytedance/deer-flow/7.1-configuration-architecture |
| 深度分析书 | github.com/coolclaws/deerflow-book |
| 介绍文章 | kiledjian.com/2026/03/06/deerflow-bytedances-opensource-ai-agent.html |
| MarkTechPost 报道 | marktechpost.com (搜索 "DeerFlow 2.0") |
| 记忆系统解析 | docs.bswen.com/blog/2026-03-16-deerflow-memory-system/ |
