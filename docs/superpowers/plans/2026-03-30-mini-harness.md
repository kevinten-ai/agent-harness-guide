# Mini Harness: Landing Knowledge Assistant — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a minimal but complete Agent Harness that helps a new employee analyze codebases and docs, producing structured knowledge maps.

**Architecture:** 6-module Python harness (agent loop, LLM client, tool registry, memory, prompt builder, safety layer) with terminal interaction. Uses GLM via OpenAI-compatible SDK. Follows Harness Engineering principles for output structure.

**Tech Stack:** Python 3.11+, openai SDK, GLM API (ZhipuAI)

**Spec:** `docs/superpowers/specs/2026-03-30-mini-harness-design.md`

---

## File Structure

```
mini-harness/
├── main.py                  ← Terminal interaction loop + special commands
├── harness/
│   ├── __init__.py          ← Re-exports for convenience
│   ├── llm.py               ← OpenAI-compatible client, chat completion wrapper
│   ├── tools.py             ← @tool decorator, registry, 6 built-in tools, JSON Schema gen
│   ├── safety.py            ← Command blacklist, loop detection counter
│   ├── memory.py            ← JSON file store, fact CRUD, token-budgeted recall
│   ├── prompt.py            ← System prompt assembly (role + tools + memory + output template)
│   └── agent.py             ← Agent loop: call LLM → parse → safety check → execute → loop
├── memory/                  ← Runtime: per-session JSON files
├── output/                  ← Runtime: generated knowledge maps
├── pyproject.toml           ← Project config, single dependency: openai
└── .env.example             ← Template for GLM_API_KEY, GLM_BASE_URL, GLM_MODEL
```

---

## Chunk 1: Foundation (llm + tools + safety)

### Task 1: Project scaffold and LLM client

**Files:**
- Create: `mini-harness/pyproject.toml`
- Create: `mini-harness/.env.example`
- Create: `mini-harness/harness/__init__.py`
- Create: `mini-harness/harness/llm.py`

- [ ] **Step 1: Create project scaffold**

```toml
# pyproject.toml
[project]
name = "mini-harness"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["openai>=1.0.0"]
```

```bash
# .env.example
GLM_API_KEY=your-api-key-here
GLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4
GLM_MODEL=glm-4-flash
```

- [ ] **Step 2: Implement LLM client**

`harness/llm.py` — wraps OpenAI SDK with GLM config:
- `LLMClient` class: init from env vars, `chat()` method
- `chat(messages, tools=None)` → returns response message
- Handles streaming=False (simplest first)
- Raises clear errors on auth/network failure

- [ ] **Step 3: Manual smoke test**

```bash
cd mini-harness
GLM_API_KEY=xxx python -c "from harness.llm import LLMClient; c = LLMClient(); print(c.chat([{'role':'user','content':'hello'}]))"
```

Expected: GLM responds with text.

- [ ] **Step 4: Commit**

```bash
git add mini-harness/
git commit -m "feat: project scaffold + LLM client (GLM via OpenAI SDK)"
```

---

### Task 2: Tool registry and built-in tools

**Files:**
- Create: `mini-harness/harness/tools.py`

- [ ] **Step 1: Implement @tool decorator and ToolRegistry**

`harness/tools.py`:
- `@tool(description=...)` decorator: captures function name, docstring, type hints
- `ToolRegistry` class: `register()`, `get_schemas()` → OpenAI function-calling JSON format, `execute(name, args)` → result string
- Auto-generates JSON Schema from Python type hints (str, int, bool, Optional)

- [ ] **Step 2: Implement 6 built-in tools**

All registered via `@tool`:

```python
@tool(description="读取文件内容")
def read_file(path: str, limit: int = 200) -> str: ...

@tool(description="列出目录结构")
def list_dir(path: str, depth: int = 2) -> str: ...

@tool(description="在代码中搜索模式")
def search_code(pattern: str, path: str = ".", file_type: str = "") -> str: ...

@tool(description="执行 shell 命令 (可调用 git/gh/glab/feishu-cli 等)")
def run_command(command: str) -> str: ...

@tool(description="写入文件")
def write_file(path: str, content: str) -> str: ...

@tool(description="记住一个重要发现")
def memorize(fact: str, source: str = "") -> str: ...
```

- `read_file`: reads with line numbers, respects limit
- `list_dir`: uses `os.walk` with depth control, tree-like output
- `search_code`: subprocess calls `grep -rn` (or `rg` if available)
- `run_command`: `subprocess.run` with timeout (30s), stderr capture
- `write_file`: creates parent dirs, writes content, returns confirmation
- `memorize`: placeholder that returns "memorized" (wired to memory.py in Task 4)

- [ ] **Step 3: Manual test**

```python
from harness.tools import registry
# Check schemas generate correctly
print(registry.get_schemas())
# Check execution
print(registry.execute("list_dir", {"path": ".", "depth": 1}))
```

- [ ] **Step 4: Commit**

```bash
git add mini-harness/harness/tools.py
git commit -m "feat: tool registry with 6 built-in tools"
```

---

### Task 3: Safety layer

**Files:**
- Create: `mini-harness/harness/safety.py`

- [ ] **Step 1: Implement safety checks**

`harness/safety.py`:
- `SafetyGuard` class:
  - `check_command(cmd: str) -> tuple[bool, str]`: checks against blacklist patterns
  - `check_loop(tool_name: str, args: dict) -> tuple[bool, str]`: tracks identical calls, threshold=3
  - `reset_loop()`: called when user sends new message
- Blacklist patterns: `rm -rf`, `sudo`, `mkfs`, `dd if=`, `> /dev/`, `shutdown`, `reboot`, `git push --force`, `git reset --hard`
- Returns `(allowed: bool, reason: str)`

- [ ] **Step 2: Manual test**

```python
from harness.safety import SafetyGuard
g = SafetyGuard()
print(g.check_command("ls -la"))           # (True, "")
print(g.check_command("rm -rf /"))         # (False, "blocked: rm -rf")
print(g.check_loop("read_file", {"path": "a.py"}))  # (True, "")
# Call same thing 3 times
print(g.check_loop("read_file", {"path": "a.py"}))  # (True, "")
print(g.check_loop("read_file", {"path": "a.py"}))  # (False, "loop detected: ...")
```

- [ ] **Step 3: Commit**

```bash
git add mini-harness/harness/safety.py
git commit -m "feat: safety layer with command blacklist and loop detection"
```

---

## Chunk 2: Memory + Prompt + Agent Loop

### Task 4: Memory system

**Files:**
- Create: `mini-harness/harness/memory.py`

- [ ] **Step 1: Implement memory store**

`harness/memory.py`:
- `MemoryStore` class:
  - `__init__(memory_dir="memory")`: loads existing session or creates new
  - `add(fact: str, source: str)`: appends `{fact, source, timestamp}`
  - `get_all() -> list[dict]`: returns all facts
  - `recall(token_budget: int = 2000) -> str`: formats facts as string within budget (estimate ~4 chars/token)
  - `save()`: writes to `memory/{session-id}.json`
  - `load(session_id: str)`: loads existing session
  - `list_sessions() -> list[str]`: lists available sessions
- Session ID: `YYYY-MM-DD-HHMMSS` format
- Wire `memorize` tool in tools.py to call `memory_store.add()`

- [ ] **Step 2: Manual test**

```python
from harness.memory import MemoryStore
m = MemoryStore()
m.add("order-service is the core module", "reading src/order/")
m.add("@zhangsan owns order-service", "git shortlog")
print(m.recall(500))
m.save()
```

Expected: JSON file created in `memory/`, recall returns formatted string.

- [ ] **Step 3: Commit**

```bash
git add mini-harness/harness/memory.py
git commit -m "feat: memory system with JSON persistence and token-budgeted recall"
```

---

### Task 5: System prompt assembly

**Files:**
- Create: `mini-harness/harness/prompt.py`

- [ ] **Step 2: Implement prompt builder**

`harness/prompt.py`:
- `build_system_prompt(tools_schema: list, memory_recall: str, project_path: str = "") -> str`
- Assembles from sections:
  1. **Role**: Landing knowledge assistant identity
  2. **Available tools**: summarized from schema
  3. **CLI tools**: gh, glab, feishu-cli, git, rg usage hints
  4. **Output structure**: INDEX.md template, directory convention
  5. **Memory**: injected recalled facts
  6. **Constraints**: read-heavy, don't modify source code, ask before destructive ops
- Total prompt target: ~1500 tokens (leaves room for conversation)

- [ ] **Step 2: Manual test**

```python
from harness.prompt import build_system_prompt
print(build_system_prompt([], "", "/some/project"))
```

Expected: well-structured system prompt with all sections.

- [ ] **Step 3: Commit**

```bash
git add mini-harness/harness/prompt.py
git commit -m "feat: system prompt builder with role, tools, output template, memory"
```

---

### Task 6: Agent loop

**Files:**
- Create: `mini-harness/harness/agent.py`

- [ ] **Step 1: Implement agent loop**

`harness/agent.py`:
- `Agent` class:
  - `__init__(llm, tools, memory, safety)`: takes all components
  - `run(user_message: str) -> str`: the core loop:

```
def run(user_message):
    safety.reset_loop()
    messages = [system_prompt] + history + [user_message]

    for i in range(MAX_ITERATIONS=15):
        response = llm.chat(messages, tools=tool_schemas)

        if response has text only (no tool_calls):
            return response.content  # done

        if response has tool_calls:
            for call in tool_calls:
                # safety check
                if call is run_command:
                    allowed, reason = safety.check_command(call.args["command"])
                    if not allowed: append error message, continue

                allowed, reason = safety.check_loop(call.name, call.args)
                if not allowed: append error message, continue

                # execute
                result = tools.execute(call.name, call.args)

                # append tool result to messages
                messages.append(tool_call_message)
                messages.append(tool_result_message)

            continue  # loop back to LLM with tool results

    return "Reached max iterations."
```

  - `history`: maintained across calls within session
  - Prints tool calls and results to terminal with color

- [ ] **Step 2: Manual test**

```python
from harness.agent import Agent
from harness.llm import LLMClient
from harness.tools import registry
from harness.memory import MemoryStore
from harness.safety import SafetyGuard

agent = Agent(LLMClient(), registry, MemoryStore(), SafetyGuard())
print(agent.run("列出当前目录的文件"))
```

Expected: Agent calls `list_dir`, returns formatted result.

- [ ] **Step 3: Commit**

```bash
git add mini-harness/harness/agent.py
git commit -m "feat: agent loop with tool dispatch, safety checks, and iteration control"
```

---

## Chunk 3: Terminal UI + Integration

### Task 7: Main entry point and terminal UI

**Files:**
- Create: `mini-harness/main.py`
- Update: `mini-harness/harness/__init__.py`

- [ ] **Step 1: Implement terminal interaction loop**

`main.py`:
- Loads `.env` (or reads env vars directly)
- Initializes all components: LLMClient, ToolRegistry, MemoryStore, SafetyGuard, Agent
- Optional: `--project /path/to/project` argument to set working context
- Optional: `--session <id>` to resume a session
- REPL loop:
  - Prompt: `> `
  - Special commands:
    - `/memory` → print current memory facts
    - `/output` → tree of output/ directory
    - `/session` → show session info
    - `/quit` → save memory and exit
  - Otherwise: pass to `agent.run()`, print response
- Colored output:
  - Tool calls: dim/gray
  - Tool results: cyan (truncated to 500 chars in display)
  - Agent response: normal/white
  - Errors: red

- [ ] **Step 2: Update __init__.py**

```python
from harness.agent import Agent
from harness.llm import LLMClient
from harness.tools import registry
from harness.memory import MemoryStore
from harness.safety import SafetyGuard
from harness.prompt import build_system_prompt
```

- [ ] **Step 3: End-to-end test**

```bash
cd mini-harness
GLM_API_KEY=xxx python main.py --project /path/to/some/repo
> 帮我了解这个项目的整体结构
# Should: call list_dir, read_file on key files, produce summary
> /memory
# Should: show any memorized facts
> /quit
```

- [ ] **Step 4: Commit**

```bash
git add mini-harness/main.py mini-harness/harness/__init__.py
git commit -m "feat: terminal UI with REPL, special commands, colored output"
```

---

### Task 8: Final integration and README

**Files:**
- Create: `mini-harness/README.md`

- [ ] **Step 1: Write README**

Brief README with:
- What this is (one paragraph)
- Quick start (3 commands: install, set env, run)
- Architecture diagram (ASCII, from spec)
- Available tools table
- Output structure description

- [ ] **Step 2: Full end-to-end validation**

Run the harness against the `harness/` repo itself:

```bash
cd mini-harness
python main.py --project /Users/kevinten/projects/harness
> 分析这个项目的结构，生成一份知识地图到 output/harness/
```

Verify:
- Agent explores the project (list_dir, read_file)
- Agent produces INDEX.md following the template
- Memory persists facts
- No safety violations
- Output directory has correct structure

- [ ] **Step 3: Commit**

```bash
git add mini-harness/
git commit -m "feat: mini-harness v0.1 — Landing Knowledge Assistant"
```
