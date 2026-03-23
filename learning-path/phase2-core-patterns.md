# Phase 2: 核心模式

## 学习目标

动手构建 Agent 的核心组件，掌握关键设计模式。

## Week 1: 核心组件

### Day 1-2: 工具集成

**学什么**: 如何让 Agent 连接到外部世界

```
核心知识点:
├── Tool Registry 设计模式
├── JSON Schema 工具定义
├── MCP 协议基础
└── 工具安全设计
```

**实践项目 A: 自定义工具注册**

```python
# 使用 Claude Agent SDK 注册自定义工具

from anthropic import Anthropic

# 定义一个文件搜索工具
file_search_tool = {
    "name": "search_files",
    "description": "在指定目录中搜索包含关键字的文件",
    "input_schema": {
        "type": "object",
        "properties": {
            "directory": {
                "type": "string",
                "description": "搜索目录的绝对路径"
            },
            "keyword": {
                "type": "string",
                "description": "搜索关键字"
            }
        },
        "required": ["directory", "keyword"]
    }
}

# 思考:
# 1. 工具描述如何影响模型的使用方式?
# 2. 参数 description 为什么这么重要?
# 3. 如何防止目录遍历攻击?
```

**实践项目 B: 配置 MCP Server**

```
步骤:
1. 安装一个 MCP Server (如 GitHub MCP)
2. 在 Claude Code 中配置 .mcp.json
3. 使用 MCP 工具完成一个任务
4. 观察: 延迟加载是如何工作的?
```

### Day 3-4: 上下文工程

**学什么**: 如何精心构建和管理提供给模型的信息

```
核心知识点:
├── System Prompt 动态组装
├── 优先级分层
├── 上下文压缩策略
└── Prompt Cache 原理
```

**实践项目: 构建动态 System Prompt**

```python
# 伪代码: 动态 System Prompt 组装器

class SystemPromptBuilder:
    def __init__(self):
        self.sections = []

    def add(self, priority: int, content: str):
        self.sections.append((priority, content))

    def build(self, max_tokens: int) -> str:
        # 按优先级排序
        sorted_sections = sorted(self.sections, key=lambda x: x[0])
        # 组装，直到达到 token 预算
        prompt = ""
        for priority, content in sorted_sections:
            if count_tokens(prompt + content) < max_tokens:
                prompt += content + "\n\n"
            else:
                break  # 超出预算，停止添加低优先级内容
        return prompt

# 使用:
builder = SystemPromptBuilder()
builder.add(10, "You are a coding assistant...")       # 身份
builder.add(20, "NEVER delete files without asking")    # 安全
builder.add(40, tool_schemas_text)                     # 工具
builder.add(60, claude_md_content)                     # 项目标准
builder.add(90, f"Working directory: {cwd}")           # 环境

system_prompt = builder.build(max_tokens=4000)
```

### Day 5-7: 记忆系统

**学什么**: 如何让 Agent 跨步骤、跨会话保持连续性

```
核心知识点:
├── 三层记忆架构
├── Initializer-Executor 模式
├── Progress File 设计
└── JSON vs Markdown 选择
```

**实践项目: 实现 Initializer-Executor 模式**

```
设计一个跨会话的任务管理系统:

1. Initializer:
   - 分析任务描述
   - 拆解为子任务
   - 创建 progress.json
   - 记录基线信息

2. Executor:
   - 读取 progress.json
   - 找到下一个未完成子任务
   - 执行子任务
   - 更新 progress.json
   - 如果完成 → 结束; 否则 → 继续

3. Recovery:
   - 如果中断 → 读取 progress.json → 从上次位置继续
```

## Week 2: 设计模式

### Day 1-2: 安全与护栏

**实践项目: 实现一个 PreToolUse Hook**

```bash
# hook.sh - 在工具调用前检查安全性
#!/bin/bash

# 读取工具名和参数
TOOL_NAME="$1"
TOOL_INPUT="$2"

# 检查危险操作
if [[ "$TOOL_NAME" == "Bash" ]]; then
    if echo "$TOOL_INPUT" | grep -qE "rm -rf|git push --force|dd if="; then
        echo "BLOCKED: Dangerous command detected"
        exit 1
    fi
fi

# 检查敏感文件
if [[ "$TOOL_NAME" == "Write" ]]; then
    if echo "$TOOL_INPUT" | grep -qE "\.env|credentials|secret"; then
        echo "BLOCKED: Attempting to write sensitive file"
        exit 1
    fi
fi

exit 0
```

### Day 3-4: 设计模式实践

**实践项目 A: Orchestrator-Worker**

```
设计一个代码分析 Agent:

Orchestrator:
├── 接收: "分析这个项目的代码质量"
├── 拆解:
│   ├── Worker 1: 搜索 TODO/FIXME
│   ├── Worker 2: 检查测试覆盖率
│   └── Worker 3: 分析代码复杂度
├── 等待所有 Worker 完成
└── 汇总: 生成综合报告

每个 Worker:
├── 只有需要的工具 (Schema 隔离)
├── 独立执行
└── 返回结构化结果
```

**实践项目 B: Evaluator-Optimizer**

```
设计一个自动改进代码的循环:

Generator Agent:
├── 生成代码

Evaluator Agent:
├── 运行测试
├── 检查代码质量
├── 生成改进建议

循环:
Generator → Evaluator → 反馈 → Generator → Evaluator → 通过 → 输出
```

### Day 5-7: 综合项目

**构建一个 Code Review Agent**

```
需求:
├── 输入: Git diff 或文件路径
├── 输出: 代码审查报告

组件:
├── Agent Loop: 基本的 Think → Act → Output
├── 工具:
│   ├── Read (读取文件)
│   ├── Bash("git diff") (获取变更)
│   └── Grep (搜索模式)
├── 安全: 只读权限 (Schema 隔离)
├── 上下文: 动态加载相关文件
└── 输出: Markdown 格式的审查报告

构建步骤:
1. 定义工具 Schema
2. 构建 System Prompt
3. 实现 Agent Loop
4. 添加输出格式化
5. 测试不同场景
```

## Phase 2 自检清单

```
□ 能独立注册和定义自定义工具
□ 理解 JSON Schema 对模型行为的影响
□ 配置过至少一个 MCP Server
□ 能构建动态 System Prompt
□ 理解上下文压缩的策略
□ 能设计 Progress File 格式
□ 实现了 Initializer-Executor 模式
□ 实现了至少一个安全 Hook
□ 实践了 Orchestrator-Worker 模式
□ 构建了一个简单但完整的 Agent
```

## 进入 Phase 3 的信号

当你能构建一个有工具、有安全、有基本记忆的 Agent，并理解每个组件的作用时，你就可以进入 Phase 3 了。
