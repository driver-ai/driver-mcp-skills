# driver-mcp-skills

Exemplar skills and guidance for getting the most out of [Driver MCP](https://driverai.com) in AI-assisted development workflows. Includes two ready-to-use skills (research and planning) and a guide for integrating Driver into your own skills.

## Quick Start

1. **Audit your own harness engineering approach** — use the [audit checklist](#how-to-audit-your-harness-engineering-approach) below
2. **Read the CLAUDE.md** — see how a project-level CLAUDE.md (or AGENTS.md) integrates Driver MCP
3. **(Optional) Try a skill** — copy `skills/research/` or `skills/planning/` into your project's `.claude/skills/` directory (or equivalent for your rig)

---

## How Driver MCP Works

Driver MCP gives your AI agents deep codebase understanding through a hierarchy of tools. Understanding this hierarchy is key to effective integration.

### Tool Hierarchy

```
request_task_context         ← PRIMARY: dispatch task-specific analysis
    │ returns request_id
    ▼
poll_task_context            ← Poll until COMPLETED, then retrieve context
    │
    └── Primitive Tools      ← Targeted follow-up (code map, file docs, source files)
```

### `request_task_context` + `poll_task_context` — The Primary Workflow

This request/poll pair is the primary workflow in Driver MCP. It should be your agent's default for any dynamic codebase context need.

> **Migrating from an earlier version?** The synchronous `gather_task_context` tool is deprecated. Replace each call with `request_task_context`, retain its `request_id`, and use `poll_task_context` to retrieve the result.

**What it actually does:** `request_task_context` spawns a specialized context agent on Driver's servers and returns a `request_id` immediately. This agent analyzes the codebase and synthesizes task-specific context tailored to your description. `poll_task_context` checks the request's status and returns that context once complete.

This is not a docs lookup. It's a server-side agent doing sophisticated codebase analysis.

**How to use it:** Provide a detailed task description and codebase names to `request_task_context`. Retain the returned `request_id`, then pass it to `poll_task_context` until the request reaches a terminal status. The richer your description, the better the context.

```
Good: "Researching how the notification system handles delivery retries.
      Need to understand: retry logic, failure modes, queue architecture,
      and how delivery status is tracked. Codebase: my-backend"

Bad:  "Tell me about notifications"
```

**Use `get_codebase_names` first** to verify exact codebase names. Invalid names can fail or misdirect a context request.

### Asynchronous Execution: 1-3 Minutes

The context agent typically takes 1-3+ minutes to finish. **This is expected and normal.** `request_task_context` returns immediately; the work continues server-side.

The context agent is doing significant work that your agent would otherwise have to do on its own through many iterations of file reading, searching, and synthesis — consuming far more tokens and taking just as long or longer.

Think of it as compressed expert-level codebase analysis. The wait is not wasted — it's the most efficient path to task-specific context.

Call `poll_task_context` for the first time after roughly 30-45 seconds, then every 20-30 seconds. Handle statuses explicitly:

- **`QUEUED` / `RUNNING`** — the agent is still working; do useful work and poll again
- **`COMPLETED`** — consume the returned synthesized context
- **`FAILED` / `CANCELLED`** — report the returned error; do not keep polling

### Primitive Tools

For targeted follow-up after broad context is gathered:

- **`get_code_map`** — navigate codebase directory structure with descriptions
- **`get_file_documentation`** — symbol-level docs for a specific file (function signatures, types, classes)
- **`get_source_file`** — read actual source code with line numbers

### Running Multiple Requests in Parallel

When you have multiple independent research angles, call `request_task_context` once per angle without waiting for earlier requests to finish. Driver runs the requests concurrently, so no native subagent wrapper is needed. Keep a mapping of each angle to its `request_id`, poll all outstanding IDs in turn, and collect every completed result before synthesizing.

---

## How to Integrate Driver into Your Skills

If you're building custom skills (SKILL.md files, CLAUDE.md instructions, or equivalent), follow these principles to ensure agents actually use Driver MCP correctly.

### 1. Name Tools Explicitly

The single most impactful thing you can do. Don't say "use Driver" — name the specific tool.

```markdown
❌ "Use Driver to understand the codebase"
❌ "Gather context about the code"
❌ "Research the codebase architecture"

✅ "Call `request_task_context` (Driver MCP) with a detailed task description
   and codebase names, then call `poll_task_context` with the returned
   `request_id` until the context is complete"
```

When a skill says "use Driver" generically, models don't know which tool to call. Name both `request_task_context` and `poll_task_context` so the model dispatches the work and retrieves the result.

### 2. Explain What the Tool Does

Models make tool choices based on their understanding of what each tool does. If your skill doesn't explain that `request_task_context` starts a server-side agent (not a docs lookup), the model may categorize it as a static documentation tool and bypass it when it thinks it needs "real" source access. If it doesn't explain the polling step, the model may never retrieve the result.

Include a brief description in your skill:

```markdown
`request_task_context` spawns a specialized context agent on Driver's servers
and immediately returns a `request_id`. The agent analyzes the codebase and
synthesizes task-specific context. Call `poll_task_context` with the ID until
it returns that context.
```

### 3. Explain Polling and Timing

Without explicit framing, models may treat a `QUEUED` or `RUNNING` response as a failure, poll too aggressively, or forget to retrieve the result. Include status handling and timing in your skill:

```markdown
The context agent takes 1-3+ minutes. This is expected. First call
`poll_task_context` after roughly 30-45 seconds, then every 20-30 seconds.
Continue while status is `QUEUED` or `RUNNING`; consume context on `COMPLETED`;
report the error on `FAILED` or `CANCELLED`.
```

### 4. Add Anti-Substitution Language

Explicitly tell the model what NOT to do:

```markdown
Do NOT use native Explore agents, subagents, or manual file-reading/grep
as a substitute for `request_task_context` + `poll_task_context`. Native tools
do not provide Driver's synthesized, task-specific analysis.
```

### 5. Use Native Async Parallelism

The asynchronous workflow already supports parallel context requests:

```markdown
For several independent questions, call `request_task_context` once per
question without waiting between calls. Retain every `request_id`, do useful
work while Driver runs them concurrently, then poll each ID in turn. Do not
spawn subagents merely to wrap Driver MCP calls.
```

---

## How to Audit Your Harness Engineering Approach

If you're using a third-party harness (like Superpowers, gstack, etc.), have skills that were written before Driver MCP, or are using a combination of techniques for harness engineering, here's how to find and fix issues.

### Audit Checklist

For each skill that involves codebase understanding:

- [ ] **Does it name `request_task_context` and `poll_task_context` explicitly?** — If it says "use Driver" or "gather context" without naming both tools, the model may not start the request or retrieve its result
- [ ] **Does it explain what the tools do?** — If the model thinks Driver is a "docs tool," it will bypass it for source-level tasks; if it misses the asynchronous contract, it may expect context from the request call
- [ ] **Does it explain statuses and polling cadence?** — Without framing, models may abandon a running request, poll too frequently, or forget to retrieve the result
- [ ] **Does it have anti-substitution language?** — Models default to native agents when the skill doesn't say otherwise
- [ ] **Does it use generic "subagent" language for codebase exploration?** — "Spawn a subagent to explore the codebase" causes models to use native agents instead of Driver
- [ ] **Does it have competing context-gathering patterns?** — Native file reading, grep-based exploration, or other tools that substitute for Driver's task-context workflow

### Scoring Your Skills

| Rating | Criteria |
|--------|----------|
| **Strong** | Names `request_task_context` and `poll_task_context`, explains the asynchronous contract and status handling, includes polling cadence and anti-substitution language |
| **Partial** | Names Driver tools but missing framing or anti-substitution language |
| **Weak** | Says "use Driver" without naming specific tools |
| **Ineffective** | No Driver mention, or generic "gather context" language that causes native agent substitution |

### Before/After Examples

**Before (weak):**
```markdown
## Research Phase
Use Driver to understand the codebase architecture. Spawn a subagent
to explore relevant code and gather context for the implementation.
```

**After (strong):**
```markdown
## Research Phase
Call `request_task_context` (Driver MCP) with a detailed task description
and codebase names. This starts a specialized context agent server-side and
returns a `request_id` immediately. Call `poll_task_context` with that ID after
roughly 30-45 seconds, then every 20-30 seconds while status is `QUEUED` or
`RUNNING`. On `COMPLETED`, use the returned synthesized context.

Do NOT use native Explore agents or subagents as a substitute for
the request/poll workflow. For targeted follow-up, use `get_code_map`,
`get_file_documentation`, or `get_source_file`.
```

---

## Common Anti-Patterns

These are real failure modes observed in production — not hypotheticals.

### 1. Deprioritization

**Symptom:** In environments with many MCP servers, skills, and tools loaded, the model simply never calls Driver MCP tools.

**Root cause:** No explicit instruction in skills or system prompt to use Driver. The model has to "discover" it on its own from tool descriptions, and in a crowded tool environment, it doesn't.

**Fix:** Add explicit Driver MCP instructions to your skills. Name `request_task_context` and `poll_task_context` directly. Don't rely on the model discovering them.

### 2. Native Subagent Substitution

**Symptom:** The model spawns native Explore agents or subagents to "research the codebase" instead of using `request_task_context` and `poll_task_context`.

**Root cause:** Skills that use generic "subagent" language for codebase exploration. The model sees it has a native Agent tool and defaults to what it knows.

**Fix:** Name the specific tool, explain that it starts a server-side context agent, and add anti-substitution language.

### 3. Incomplete or Impatient Polling

**Symptom:** The model calls `request_task_context` but never polls, abandons the request after a `QUEUED` or `RUNNING` response, or polls continuously without doing useful work between checks.

**Root cause:** The skill does not explain the asynchronous contract, terminal statuses, or appropriate polling cadence.

**Failure mode:** A model sees that the request returned no context, assumes the result is empty, and falls back to a simpler tool instead of using the returned `request_id` to poll.

**Fix:** State that `request_task_context` returns only a request ID, specify the first and subsequent poll timing, define all statuses, and require polling every request to a terminal status.

---

## Exemplar Skills

This repo includes two exemplar skills that demonstrate these integration patterns in practice.

### Research Skill (`skills/research/`)

A standalone skill for exploring technical topics against codebases. Demonstrates:
- `request_task_context` + `poll_task_context` as the primary workflow with explicit status and cadence guidance
- Anti-substitution language that survives tool-heavy environments
- Native asynchronous parallelism for concurrent task-context requests
- Conversational Q&A to clarify research intent before gathering context
- Organized output: overview document + numbered deep-dive research docs

### Planning Skill (`skills/planning/`)

A skill for creating implementation plans from research output. Demonstrates:
- Progressive deepening through the full Driver MCP tool hierarchy
- The request/poll workflow for broad architectural context
- Primitive tools (`get_code_map`, `get_file_documentation`, `get_source_file`) for code-level plan specificity
- TDD-first task ordering with concrete, implementable task specifications
- Self-review step that validates the plan against actual codebase state using Driver tools

**Typical flow:** Run the research skill first, then point the planning skill at the research output.

### How to Use Them

1. Copy the skill directory (e.g., `skills/research/`) into your project's skill location (`.claude/skills/` for Claude Code)
2. Ensure Driver MCP is configured in your environment (`.mcp.json` or equivalent)
3. Invoke the skill through your agent rig's skill mechanism

---

## Also See

- **CLAUDE.md** in this repo — a working example of Driver MCP integration in a project-level config
- **[Driver Documentation](https://driver.ai/docs)** — full Driver MCP documentation
