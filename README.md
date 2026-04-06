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
gather_task_context          ← PRIMARY: start here, always
    │
    ├── Deep Context Docs    ← Full pre-computed documents (architecture, onboarding, changelog)
    │
    └── Primitive Tools      ← Targeted follow-up (code map, file docs, source files)
```

### `gather_task_context` — The Primary Tool

This is the single most important tool in Driver MCP. It should be your agent's default for any dynamic codebase context need.

**What it actually does:** It spawns a specialized context agent on Driver's servers. This agent reads pre-computed, exhaustive codebase documentation — architecture overviews, code maps, symbol-level file documentation, development changelogs — and performs live runtime analysis. It then synthesizes everything into task-specific context tailored to your description.

This is not a docs lookup. It's a server-side agent doing sophisticated codebase analysis.

**How to call it:** Provide a detailed task description and codebase names. The richer your description, the better the context.

```
Good: "Researching how the notification system handles delivery retries.
      Need to understand: retry logic, failure modes, queue architecture,
      and how delivery status is tracked. Codebase: my-backend"

Bad:  "Tell me about notifications"
```

**Use `get_codebase_names` first** to verify exact codebase names. Typos cause empty results.

### Execution Time: 1-3 Minutes

`gather_task_context` typically takes 1-3+ minutes to return. **This is expected and normal.**

The tool is doing significant work that your agent would otherwise have to do on its own through many iterations of file reading, searching, and synthesis — consuming far more tokens and taking just as long or longer. It produces much higher-quality context because it leverages pre-computed, exhaustive guiding documentation that native tools don't have access to.

Think of it as compressed expert-level codebase analysis. The wait is not wasted — it's the most efficient path to deep context.

### Deep Context Documents

For cases where you want the full, unabridged codebase-wide documents, these are available directly:

- **`get_architecture_overview`** — complete architecture document
- **`get_llm_onboarding_guide`** — codebase orientation, navigation, conventions
- **`get_changelog`** / **`get_detailed_changelog`** — development history

These are large documents. `gather_task_context` reads them server-side and returns only what's relevant. Use these directly only when you need the complete source document.

### Primitive Tools

For targeted follow-up after broad context is gathered:

- **`get_code_map`** — navigate codebase directory structure with descriptions
- **`get_file_documentation`** — symbol-level docs for a specific file (function signatures, types, classes)
- **`get_source_file`** — read actual source code with line numbers

### Running Multiple Calls in Parallel

`gather_task_context` is a synchronous MCP tool call. When you have multiple research angles, you can parallelize by spawning native subagents whose **only job** is to call `gather_task_context` and return the result.

The subagent is a concurrency wrapper — it does NOT do its own codebase exploration. This is a critical distinction (see [Anti-Patterns](#common-anti-patterns) below).

---

## How to Integrate Driver into Your Skills

If you're building custom skills (SKILL.md files, CLAUDE.md instructions, or equivalent), follow these principles to ensure agents actually use Driver MCP correctly.

### 1. Name Tools Explicitly

The single most impactful thing you can do. Don't say "use Driver" — name the specific tool.

```markdown
❌ "Use Driver to understand the codebase"
❌ "Gather context about the code"
❌ "Research the codebase architecture"

✅ "Call `gather_task_context` (Driver MCP) with a detailed task description
   and codebase names"
```

When a skill says "use Driver" generically, models don't know which tool to call. When it names `gather_task_context` explicitly, they call it.

### 2. Explain What the Tool Does

Models make tool choices based on their understanding of what each tool does. If your skill doesn't explain that `gather_task_context` is a server-side agent (not a docs lookup), the model may categorize it as a static documentation tool and bypass it when it thinks it needs "real" source access.

Include a brief description in your skill:

```markdown
`gather_task_context` spawns a specialized context agent on Driver's servers
that reads pre-computed, exhaustive codebase documentation and does live
analysis. It returns synthesized, task-specific context.
```

### 3. Address the Wait Time

Without explicit framing, models may interpret the 1-3 minute execution time as a failure signal and abandon the call. Include wait-time framing in your skill:

```markdown
`gather_task_context` takes 1-3+ minutes. This is expected. Wait for the full
response — it is doing work that would take you longer to do iteratively.
```

### 4. Add Anti-Substitution Language

Explicitly tell the model what NOT to do:

```markdown
Do NOT use native Explore agents, subagents, or manual file-reading/grep
as a substitute for `gather_task_context`. Native tools work from raw source
only. Driver has access to pre-computed, exhaustive codebase documentation
that native tools cannot replicate.
```

### 5. Distinguish Substitution from Parallelism

If your skill uses native subagents for any purpose, be explicit about the distinction:

```markdown
Native subagents may be used as concurrency wrappers to run multiple
`gather_task_context` calls in parallel. The subagent's ONLY job is to
make the Driver MCP call and return the result — it does NOT do its own
codebase exploration.
```

---

## How to Audit Your Harness Engineering Approach

If you're using a third-party harness (like Superpowers, gstack, etc.), have skills that were written before Driver MCP, or are using a combination of techniques for harness engineering, here's how to find and fix issues.

### Audit Checklist

For each skill that involves codebase understanding:

- [ ] **Does it name `gather_task_context` explicitly?** — If it says "use Driver" or "gather context" without naming the tool, the model may not call it
- [ ] **Does it explain what the tool does?** — If the model thinks Driver is a "docs tool," it will bypass it for source-level tasks
- [ ] **Does it address the wait time?** — Without framing, models may abandon the 1-3 minute call
- [ ] **Does it have anti-substitution language?** — Models default to native agents when the skill doesn't say otherwise
- [ ] **Does it use generic "subagent" language for codebase exploration?** — "Spawn a subagent to explore the codebase" causes models to use native agents instead of Driver
- [ ] **Does it have competing context-gathering patterns?** — Native file reading, grep-based exploration, or other tools that do what `gather_task_context` does (worse)

### Scoring Your Skills

| Rating | Criteria |
|--------|----------|
| **Strong** | Names `gather_task_context` explicitly, explains what it does, includes wait-time framing and anti-substitution language |
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
Call `gather_task_context` (Driver MCP) with a detailed task description
and codebase names. This spawns a specialized context agent server-side
and takes 1-3+ minutes — wait for the full response. It returns synthesized,
task-specific context that would take longer to gather manually.

Do NOT use native Explore agents or subagents as a substitute for
`gather_task_context`. For targeted follow-up, use `get_code_map`,
`get_file_documentation`, or `get_source_file`.
```

---

## Common Anti-Patterns

These are real failure modes observed in production — not hypotheticals.

### 1. Deprioritization

**Symptom:** In environments with many MCP servers, skills, and tools loaded, the model simply never calls Driver MCP tools.

**Root cause:** No explicit instruction in skills or system prompt to use Driver. The model has to "discover" it on its own from tool descriptions, and in a crowded tool environment, it doesn't.

**Fix:** Add explicit Driver MCP instructions to your skills. Name `gather_task_context` directly. Don't rely on the model discovering it.

### 2. Native Subagent Substitution

**Symptom:** The model spawns native Explore agents or subagents to "research the codebase" instead of calling `gather_task_context`.

**Root cause:** Skills that use generic "subagent" language for codebase exploration. The model sees it has a native Agent tool and defaults to what it knows.

**Real-world example:** A model explained its choice: *"Driver is for pre-computed documentation. I needed actual source, so I used an Explore agent."* This reasoning is wrong — `gather_task_context` spawns a live agent that reads source — but reveals how models categorize Driver when skills don't explain what it does.

**Fix:** Name the specific tool, explain what it does (server-side agent, not docs lookup), and add anti-substitution language.

### 3. Impatience / Abandonment

**Symptom:** The model calls `gather_task_context`, but doesn't wait for the response. It either proceeds with parallel work and ignores the results, or abandons the call and falls back to simpler tools.

**Root cause:** The 1-3 minute execution time doesn't match the model's expectation of sub-second tool responses. Without framing, the model interprets the delay as failure.

**Real-world example:** A model called `gather_task_context`, received an empty intermediate response, and reasoned: *"The empty result might mean it's processing. But that doesn't make sense for a synchronous tool call. Let me try getting the architecture overview instead."* It talked itself out of waiting.

**Fix:** Include explicit wait-time framing in your skills. Explain that the time is expected and that the tool is doing work the agent would otherwise have to do itself.

---

## Exemplar Skills

This repo includes two exemplar skills that demonstrate these integration patterns in practice.

### Research Skill (`skills/research/`)

A standalone skill for exploring technical topics against codebases. Demonstrates:
- `gather_task_context` as the primary tool with proper wait-time framing
- Anti-substitution language that survives tool-heavy environments
- Parallel subagent-wrapper pattern for concurrent `gather_task_context` calls
- Conversational Q&A to clarify research intent before gathering context
- Organized output: overview document + numbered deep-dive research docs

### Planning Skill (`skills/planning/`)

A skill for creating implementation plans from research output. Demonstrates:
- Progressive deepening through the full Driver MCP tool hierarchy
- `gather_task_context` for broad architectural context
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
- **[Driver Documentation](https://driverai.com)** — full Driver MCP documentation
