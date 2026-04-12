# PLANNING.md

Generic planning protocol for all AI agents working in this repository.
Applies to Claude, Copilot, Cursor, Codex, and any other AI assistant.

> This protocol exists to keep the main context window lean. Discovery work is expensive and pollutes synthesis. Delegate it.
>
> **Why this is mathematical, not just stylistic:** Naive agent loops accumulate O(N²) tokens — the API rebills the full conversation history on every turn. A 10-step loop with 8K tokens/step costs ~43× more than a single-pass call. Sub-agent isolation cuts that by ~40–54% by keeping each agent's trace invisible to all others. Context rot compounds this: model reasoning quality degrades measurably before the window is full (effective limit is well below the advertised maximum). Keeping the main context lean is cost control and quality control simultaneously.

---

## When to Apply This Protocol (Auto-Trigger)

Activate the planning protocol when **any** of these are true:

| Signal | Example |
|--------|---------|
| Task touches ≥ 2 apps or packages | API change + client-app query update |
| New feature or domain (not a known bug in a known file) | Adding a new NestJS module |
| Schema or migration change | New entity, new column, relation change |
| Refactor spanning > 3 files | Renaming a service, moving a pattern |
| Architectural decision | New integration, new shared pattern |
| Ambiguous scope ("update X to do Y" with no file specified) | Any task where target files are not obvious |

**Skip the protocol when:**
- Single-file bug fix with an identified line
- Pure text/copy change
- Test-only change with known file
- Plan already exists in the current context
- Simple config or env var change
- Task is ≤ 5 clearly scoped steps with all target files already known — fan-out overhead exceeds its value at this scale; sub-agents cost ~15× more tokens per turn than a single agent

---

## Planning Protocol

### Step 1 — Decompose into Context Dimensions

Map the task to **at most 5** of these standard dimensions. Pick only what is relevant — unused dimensions waste tokens.

| Dimension | What to discover |
|-----------|-----------------|
| **Schema & Data Model** | Entities, DB columns, TypeORM relations, GraphQL types involved |
| **Existing Patterns** | How similar things are implemented in the target app (naming, file structure, conventions) |
| **Consumers & Dependencies** | What imports / calls the code being changed; downstream impact |
| **Test Requirements** | Existing test patterns in the area; what project rules require for this type of change |
| **Cross-Cutting Concerns** | Auth guards, validation, env vars, migrations, type generation, shared packages |

### Step 2 — Spawn Parallel Explore Agents

For each selected dimension, spawn one `Explore` sub-agent. Rules:

- **Maximum 5 agents in parallel.** Never exceed this.
- Each agent gets **ONE question only.** No compound tasks.
- Agent type must be `Explore` — read-only tools (Glob, Grep, Read). No Edit, Write, or further Agent spawns.
- Provide an **explicit search scope** (file paths, glob patterns, or directory). No open-ended wandering.
- Include the output contract in every prompt (see template below).

#### Sub-Agent Prompt Template

```
You are a context-gathering agent. Read-only. Do NOT edit files or spawn sub-agents.

QUESTION: [one specific question]

SCOPE: [explicit paths / glob patterns to search — nothing outside this]

Return ONLY this format, hard cap 300 words / ~400 tokens total:

FACTS:
- [bullet: what exists, what is relevant]

GOTCHAS:
- [bullet: edge cases, non-obvious behaviors, traps]

CONSTRAINTS:
- [bullet: things that limit or shape the solution]

If a section has nothing to report, write "- none".
```

**Token discipline for sub-agent prompts:**
- Keep the prompt itself ≤ 150 words.
- Prefer `Grep` over `Read` for pattern discovery — it returns only matching lines.
- Use `Read` with `offset`/`limit` for targeted file reads — never full-file reads of large files.
- Do not repeat context the sub-agent can derive from the scope itself.

### Step 3 — Synthesize

After all sub-agents return, the main agent synthesizes. **Do not read additional files at this step** — use only what the sub-agents returned.

Produce this output before writing any code:

```markdown
## Implementation Plan
1. [concrete step — what file, what change, why]
2. ...

## Edge Cases
- [each gotcha or constraint that must be handled in the implementation]

## Out of Scope
- [explicit list of things this task does NOT include — prevents scope creep]
```

The plan must be specific enough that a different agent could execute it without re-reading the codebase.

### Step 4 — Execute

Implement against the plan. If a new discovery during execution would change the plan, **update the plan first**, then continue.

**Execution phase context discipline:**

- **Context trim trigger** — after every 10–15 tool calls, pause and compress: summarize what has been done, drop redundant tool outputs, keep only the active plan + last 5 tool results. This saves ~22% tokens over unmanaged execution.
- **Note-taking for long tasks** — for tasks spanning many files or phases, maintain a lightweight scratchpad (a short markdown block in context, not a file) tracking: completed steps, open decisions, discovered constraints. Retrieve this at each phase transition instead of re-reading prior tool outputs.
- **Context utilization alert** — if context utilization exceeds ~85% of the model's window, stop and compress before continuing. Do not push to the limit; model reasoning degrades well before the hard cutoff.
- **Main agent cost share** — the orchestrating agent should consume ≤ 10–15% of total tokens across the whole task. If it grows beyond this, it is accumulating sub-agent history it should not hold.

---

## Token Discipline Summary

| Rule | Why |
|------|-----|
| Use `Explore` agents for discovery | No Edit/Write tool overhead in sub-agent context |
| 5-agent cap | Token cost grows linearly; most tasks need ≤ 3 |
| 300-word / ~400-token output cap per agent | Forces compression; prevents context flood while leaving room for nuanced GOTCHAS |
| Sub-agent prompt ≤ 150 words | Lean prompt = lean context |
| No file reads during synthesis | Sub-agent outputs already contain the signal |
| Explicit scope per sub-agent | Prevents wide glob matches returning noise |
| Skip unused dimensions | Irrelevant agents = pure waste |

---

## Anti-Patterns

- **Main agent reads files to "double-check" sub-agent output** — trust the output or fix the sub-agent prompt.
- **One sub-agent asked to cover multiple dimensions** — split into separate agents.
- **Sub-agent given open scope ("search the whole repo")** — always constrain to relevant paths.
- **Planning skipped because "the task seems small"** — use the trigger table, not gut feel.
- **Plan written in prose without numbered steps** — the plan must be executable, not descriptive.
- **Out of Scope section omitted** — always write it; ambiguous scope causes over-implementation.
- **Main agent accumulates sub-agent full histories** — receive only the `FACTS / GOTCHAS / CONSTRAINTS` block; never pull a sub-agent's full tool trace into the main context. Sub-agents' internal exploration stays isolated.
- **No context trimming during long execution** — unmanaged execution phases accumulate O(N²) tokens just like planning does. Apply the 10–15 tool call trim trigger without exception.
