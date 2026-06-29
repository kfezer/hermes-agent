# Fork Ideas

Tracked here because GitHub issues are disabled on this fork.
Open these on the upstream repo when ready to propose.

---

## Idea 1: Reduce clarifying questions for custom/local model providers

### Problem

`OPENAI_MODEL_EXECUTION_GUIDANCE` (which contains the `<act_dont_ask>` block) is only injected for `gpt`, `codex`, and `grok` model name prefixes (`system_prompt.py:255`). Custom/local providers — Gemma, Mistral, Qwen, and any `provider: custom` deployment — get `TOOL_USE_ENFORCEMENT_GUIDANCE` and `GOOGLE_MODEL_OPERATIONAL_GUIDANCE` but **not** the act-don't-ask block.

In practice this means local models ask repeated clarifying questions for things they should handle with sensible defaults: cron schedules, traceability setups, job configurations, location-based queries.

### Root cause

```python
# system_prompt.py:255 — only gpt/codex/grok get act_dont_ask
if "gpt" in _model_lower or "codex" in _model_lower or "grok" in _model_lower:
    stable_parts.append(OPENAI_MODEL_EXECUTION_GUIDANCE)
```

The existing comment at line 356 already acknowledges: *"The body is family-agnostic; the OPENAI_ prefix reflects origin, not exclusivity."*

### Immediate fix (in this fork, branch `fix/act-dont-ask-gemma`)

Extend the condition to include `gemma`:

```python
if "gpt" in _model_lower or "codex" in _model_lower or "grok" in _model_lower or "gemma" in _model_lower:
```

### Broader fix (upstream proposal)

Add a config option `agent.act_dont_ask: auto | true | false` mirroring the existing `tool_use_enforcement` knob — so operators can explicitly enable or disable the guidance for any provider without relying on model name substring matching. Especially important for `provider: custom` deployments.

### Companion improvements

- Pre-populating `USER.md` with known user defaults (location, preferred tools, recurring workflows) reduces repeated questions at the data layer — no code change needed
- `agent.environment_hint` in `config.yaml` for operator-level context injection

---

## Idea 2: Dream — weekly self-reflection cron for continuous improvement

### Problem

Hermes improves only when the agent reactively calls the `memory` tool during a session. Repeated corrections, learned workflows, and user preferences don't propagate automatically. Over weeks of use, the same clarifications recur and the same mistakes repeat.

### Concept

A weekly **Dream** cron job: while idle (3 AM Sunday), Hermes reviews its recent interaction history and synthesizes improvements into memory. Not passive fact-adding — active pattern synthesis, memory hygiene, and entry consolidation. The result is a progressively better agent that needs less hand-holding over time.

### Why this is different from existing memory writes

The `memory` tool is reactive — the agent writes what it notices *during* a session. Dream is retrospective — it reads *across* sessions and identifies patterns the agent wouldn't notice in isolation: recurring user corrections, consistently used workflows, stale entries that no longer reflect reality.

### Design constraints (local/small model deployment)

Reference: Gemma 4 E4B (4-bit, ~28 tok/s), 18K-token system prompt. Memory limits: 2,200 chars `MEMORY.md`, 1,375 chars `USER.md`.

**Critical design principle:** The memory tool hard-refuses adds that would exceed limits, returning the full entry list and "consolidate now." The `operations` batch evaluates the *net final state* — so removing 2 + adding 1 is valid in one atomic call even if the add alone would overflow.

The Dream job must:
- Use `enabled_toolsets: ["session_search", "memory"]` to minimize its own context footprint
- Treat **consolidation as the primary function**, adding as secondary
- Leave memory the **same size or smaller** after every run — never larger
- Gate adds on available headroom (< 60% usage)
- End with `[SILENT]` to suppress delivery (internal maintenance only)

### Memory usage tiers

| Usage | Behavior |
|---|---|
| < 60% | Consolidation pass first, then up to 2 new entries if genuinely novel |
| 60–80% | Consolidation-only — merge/remove, no net adds |
| > 80% | Emergency compression — aggressively consolidate, target dropping below 60% |

### Implementation — existing mechanisms only

Everything needed already exists:
- `session_search(sort="newest", limit=10)` — browse recent sessions (cron sessions visible)
- `session_search(session_id="...")` — read full session content
- `memory(action="read")` — required at job start since `skip_memory=True` in cron context; also shows current usage as `"X/2,200 chars"`
- `memory(operations=[...])` — atomic batch; checks net final state, enabling swap patterns

### Cron job definition

```json
{
  "name": "Dream",
  "schedule": "0 3 * * 0",
  "enabled_toolsets": ["session_search", "memory"],
  "deliver": "local",
  "prompt": "..."
}
```

### Dream prompt (size-first design)

```
You are running a weekly Dream pass — a memory hygiene and distillation job.

CONTEXT: This deployment runs a small local model (Gemma 4 E4B) with hard memory
limits: 2,200 chars for MEMORY, 1,375 chars for USER. Your primary job is to keep
memory lean, high-signal, and below 60% capacity. Adding knowledge is secondary
to compression and consolidation.

Step 1 — Check current state (ALWAYS DO THIS FIRST)
Call memory(action="read", target="memory") and memory(action="read", target="user").
The read output shows current usage as "X/2,200 chars" and "X/1,375 chars".
Record both percentages before proceeding.

Step 2 — Review recent sessions
Call session_search(sort="newest", limit=10) to browse session titles and previews.
For 2-4 substantive sessions, call session_search(session_id="...") to read them.
Note patterns: recurring corrections, consistent workflows, repeated questions.

Step 3 — Decide mode based on usage
- MEMORY < 60% AND USER < 60%: consolidation pass + up to 2 new entries if novel
- Either store 60-80%: consolidation-only, no new entries
- Either store > 80%: emergency compression, target dropping below 60%

Step 4 — Update memory with atomic operations
Use memory(operations=[...]) for all changes in a single atomic batch.
The batch checks the NET final state — removing 3 entries + adding 1 is valid
even if the add alone would overflow.

Consolidation rules:
- Merge any two entries that cover the same topic into one shorter entry
- Remove entries that are superseded, no longer true, or too specific to one session
- Rephrase verbose entries as terse rules (prefer "Use uv for Python" over sentences)
- Each entry must be under 80 chars — rewrite anything longer
- After the batch, total chars must be <= what they were at Step 1

Addition rules (only when below 60% capacity):
- Max 2 new entries per run total across both stores
- Only add if the pattern appeared in >= 2 sessions and is not already captured
- Prefer a behavioral rule ("Always X when Y") over a one-off fact

Step 5 — End with [SILENT]
Internal maintenance only — no delivery needed.
```

### Future: two-phase design for larger deployments

For deployments where one LLM run can't afford both the review and synthesis passes:
1. **Gather job** — session_search, writes a structured summary to output
2. **Synthesize job** — reads gather output via `context_from`, writes to memory

Hermes's `context_from` field in job definitions already supports this pattern.

### Upstream proposal

This could ship as:
1. A new `hermes cron blueprint dream` entry in `blueprint_catalog.py`
2. Documentation in `docs/` on setting up continuous self-improvement
3. Optionally: a `hermes dream` CLI alias for `hermes cron run <dream-job-id>`
