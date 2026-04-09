# Mini Harness: Landing Knowledge Assistant — Design Spec

## Context

A minimal but complete Agent Harness for onboarding at a new company. Analyzes codebases, documentation (Feishu), and produces structured knowledge maps following Harness Engineering principles (Ryan Lopopolo, OpenAI).

## Architecture

```
mini-harness/
├── main.py              ← Entry: terminal interaction loop
├── harness/
│   ├── __init__.py
│   ├── agent.py         ← Agent Loop: think → act → observe → loop
│   ├── llm.py           ← LLM client (GLM via OpenAI-compatible SDK)
│   ├── tools.py         ← Tool registry + 6 built-in tools
│   ├── memory.py        ← Memory system (JSON file persistence)
│   ├── prompt.py        ← System Prompt dynamic assembly
│   └── safety.py        ← Safety layer (blacklist + loop detection)
├── memory/              ← Memory storage (runtime generated)
├── output/              ← Output directory (knowledge maps, HTML)
└── pyproject.toml       ← Dependency: openai SDK only
```

## Data Flow

```
User Input
  → prompt.py assembles full context (system + tools + memory + history)
  → llm.py calls GLM
  → agent.py parses response
  → tool_calls? → safety.py check → tools.py execute → memory.py record
  → text response? → display to user
  → loop until done or max iterations
```

## LLM

- Provider: ZhipuAI (GLM)
- Protocol: OpenAI-compatible (base_url + api_key)
- SDK: `openai` Python package
- Config: env vars `GLM_API_KEY`, `GLM_BASE_URL`, `GLM_MODEL`

## Tools (6)

| Tool | Purpose | Landing Use Case |
|------|---------|-----------------|
| read_file | Read file content (with line limit) | Read code, configs, docs |
| list_dir | List directory (with depth control) | Understand project skeleton |
| search_code | Search patterns in code | Find key classes, call chains |
| run_command | Execute shell commands | git, gh, glab, feishu-cli, etc. |
| write_file | Write files | Produce knowledge maps, HTML |
| memorize | Store to memory | Remember key findings across sessions |

CLI tools accessed via run_command: gh, glab, feishu-cli, git, rg, tree, etc.

## Memory

- Storage: `memory/{session-id}.json`
- Structure: list of `{fact, source, timestamp}` objects
- Loaded into system prompt on each turn (with token budget)
- Cross-session: facts persist, loaded on next session start

## Safety

- Command blacklist: `rm -rf`, `sudo`, `shutdown`, destructive git ops
- Loop detection: counter for identical tool calls (threshold: 3)
- Max iterations per turn: 15

## Output Structure (Harness Engineering Principles)

```
output/{project-name}/
├── INDEX.md                ← Entry map (~100 lines, pointers only)
├── architecture.md         ← Layered architecture + module relations
├── modules/                ← One file per core module
│   ├── {module-a}.md
│   └── ...
├── people.md               ← Key people/teams + domains
├── risks.md                ← Tech debt, known issues
├── glossary.md             ← Business terminology
├── onboarding-checklist.md ← Auto-generated landing TODOs
└── explorer.html           ← Interactive knowledge browser (single-file)
```

INDEX.md follows Ryan Lopopolo's principle: ~100 lines, pointers not inline content, progressive disclosure.

## System Prompt Design

The system prompt tells the agent:
1. Its role: "Landing knowledge assistant"
2. Available tools and CLI tools
3. Output structure template (INDEX.md format)
4. Memory recall (injected facts)
5. Constraints: read-heavy, ask before destructive ops

## Interaction

- Terminal-first (stdin/stdout)
- Colored output (tool calls, agent thinking, results)
- Special commands: `/memory` (show memory), `/output` (show output tree), `/quit`
