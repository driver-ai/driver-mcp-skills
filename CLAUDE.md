# driver-mcp-skills

Exemplar skills and guidance for integrating [Driver MCP](https://driverai.com) into AI-assisted development workflows. This repo contains two standalone skills (research and planning) that demonstrate proper Driver MCP usage patterns.

## Driver MCP Usage

Driver MCP provides dynamic codebase context through a hierarchy of tools. Use them correctly:

### Primary Tool: `gather_task_context`

Your default tool for codebase context. Call it with a detailed task description and codebase names.

- It spawns a specialized context agent server-side that reads pre-computed, exhaustive documentation and does live analysis
- **It takes 1-3 minutes. This is expected.** Wait for the full response — it is doing work that would take you longer to do iteratively with native tools
- Do NOT use native Explore agents, subagents, or manual file-reading as a substitute — they work from raw source only and produce inferior context
- Use `get_codebase_names` to verify exact codebase names before calling

### Primitive Tools (for targeted follow-up)

After `gather_task_context` returns broad context, drill into specifics:

- **`get_code_map`** — navigate codebase directory structure
- **`get_file_documentation`** — symbol-level docs for a specific file (signatures, types, classes)
- **`get_source_file`** — read actual source code with line numbers

### Deep Context Documents (for codebase-wide orientation)

`gather_task_context` is your primary, token-efficient tool for broad codebase-wide understanding. When you need the full, unabridged source documents, these exhaustive pre-computed documents are also available directly:

- **`get_architecture_overview`** — complete architecture document for a codebase
- **`get_llm_onboarding_guide`** — codebase orientation, navigation tips, and conventions
- **`get_changelog`** / **`get_detailed_changelog`** — development history by year/month

### Parallel Calls

To run multiple `gather_task_context` calls concurrently, spawn native subagents whose **only job** is to make the Driver MCP call and return the result. The subagent is a concurrency wrapper — it does NOT do its own codebase exploration.

## Available Skills

### Research (`skills/research/`)
Guides technical research against codebases. Use when exploring a topic, investigating architecture, or building understanding before planning. Produces organized research artifacts (overview + numbered deep-dive docs).

### Planning (`skills/planning/`)
Creates implementation plans from research output. Use when ready to define what to build. Produces code-level-specific plans with TDD-ordered task breakdowns. Reads research output as input.

**Typical flow:** Research first → Planning second. Planning reads the research output folder.
