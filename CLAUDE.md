# driver-mcp-skills

Exemplar skills and guidance for integrating [Driver MCP](https://driverai.com) into AI-assisted development workflows. This repo contains two standalone skills (research and planning) that demonstrate proper Driver MCP usage patterns.

## Driver MCP Usage

Driver MCP provides dynamic codebase context through a hierarchy of tools. Use them correctly:

### Primary Workflow: `request_task_context` + `poll_task_context`

Your default workflow for codebase context:

- Call `request_task_context` with a detailed task description and codebase names. It spawns a specialized context agent server-side and immediately returns a `request_id`.
- Call `poll_task_context` with that `request_id` to check progress and retrieve the result. A `QUEUED` or `RUNNING` status means the request is still working; `COMPLETED` includes the synthesized context; `FAILED` or `CANCELLED` includes an error.
- **The context agent takes on the order of several minutes. This is expected.** First poll after roughly 30-45 seconds, then every 20-30 seconds until it reaches a terminal status or the stall threshold. Do useful work between polls. If it is still pending well past roughly 10 minutes, treat it as stalled: report it and resubmit rather than polling forever.
- Do NOT use native Explore agents, subagents, or manual file-reading as a substitute — they work from raw source only and produce inferior context
- Use `get_codebase_names` to verify exact codebase names before requesting context

### Primitive Tools (for targeted follow-up)

After `poll_task_context` returns the completed broad context, drill into specifics:

- **`get_code_map`** — navigate codebase directory structure
- **`get_file_documentation`** — symbol-level docs for a specific file (signatures, types, classes)
- **`get_source_file`** — read actual source code with line numbers

### Deep Context Documents (for codebase-wide orientation)

The request/poll workflow is your primary, token-efficient path to broad codebase-wide understanding. When you need the full, unabridged source documents, these exhaustive pre-computed documents are also available directly:

- **`get_architecture_overview`** — complete architecture document for a codebase
- **`get_llm_onboarding_guide`** — codebase orientation, navigation tips, and conventions
- **`get_changelog`** / **`get_detailed_changelog`** — development history by year/month

### Parallel Requests

For multiple distinct questions, call `request_task_context` once per question without waiting for earlier requests to complete. Driver runs independent requests concurrently; no subagent wrapper is needed. Keep each returned `request_id`, then poll every outstanding request in turn and collect completed results.

## Available Skills

### Research (`skills/research/`)
Guides technical research against codebases. Use when exploring a topic, investigating architecture, or building understanding before planning. Produces organized research artifacts (overview + numbered deep-dive docs).

### Planning (`skills/planning/`)
Creates implementation plans from research output. Use when ready to define what to build. Produces code-level-specific plans with TDD-ordered task breakdowns. Reads research output as input.

**Typical flow:** Research first → Planning second. Planning reads the research output folder.
