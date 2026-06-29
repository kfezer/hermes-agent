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

The Dream job must:
- Use `enabled_toolsets: ["session_search", "memory"]` to minimize its own context footprint
- Produce short, high-signal entries (target: under 100 chars each)
- Consolidate and remove entries, not just append — keep total memory lean
- End with `[SILENT]` to suppress delivery (internal maintenance only)

### Implementation — existing mechanisms only

Everything needed already exists:
- `session_search(sort="newest", limit=10)` — browse recent sessions (cron sessions visible)
- `session_search(session_id="...")` — read full session content
- `memory(action="read")` — required at job start since `skip_memory=True` in cron context
- `memory(operations=[...])` — atomic batch of add/replace/remove in one call

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

### Dream prompt

```
You are running a weekly self-reflection pass. Your goal: improve future sessions
by synthesizing patterns from recent interactions into memory, while keeping
memory lean for a context-limited deployment (hard limits: 2,200 chars MEMORY,
1,375 chars USER).

Step 1 — Load current state
Call memory(action="read", target="memory") and memory(action="read", target="user").
Note existing entries — do not duplicate them.

Step 2 — Review recent sessions
Call session_search(sort="newest", limit=10) for a session list.
For the 3-5 most substantive sessions, call session_search(session_id="...") to read them.

Step 3 — Identify improvements
Look for:
- Recurring corrections or clarifications the user made → encode as behavioral rules
- Workflows or tools that worked well → record the pattern, not the instance
- Stale or redundant memory entries that can be consolidated or removed
- Preferences that appeared consistently but weren't captured
- Tasks that recur on a schedule → candidate cron jobs to suggest

Step 4 — Update memory atomically
Use memory(operations=[...]) with a mix of add/replace/remove.
Rules:
- Max 3 new entries per run — quality beats quantity
- Each entry under 100 chars — compress ruthlessly
- Always consolidate two entries into one where possible
- Prefer behavioral rules over one-off facts

Step 5 — End with [SILENT]
This is an internal maintenance run. No delivery.
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
