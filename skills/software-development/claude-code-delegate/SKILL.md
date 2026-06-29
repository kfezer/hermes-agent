---
name: claude-code-delegate
description: "Hermes implements, Claude Code CLI reviews the final diff. Falls back to Hermes self-review on any failure."
version: 1.0.0
author: kfezer
license: MIT
platforms: [macos]
metadata:
  hermes:
    tags: [coding, review, delegation, claude-code, fallback, local-model]
    related_skills: [requesting-code-review, simplify-code, systematic-debugging, test-driven-development]
---

# Claude Code Delegate

Use this skill when implementing non-trivial code changes. Hermes does all the
implementation work; Claude Code CLI acts as an independent final reviewer with
a hard spend cap. If Claude Code is unavailable or hits its budget, a fresh
Hermes subagent reviews instead — task completion is never blocked.

Designed for deployments where Hermes runs a local model (Gemma 4, Qwen, etc.)
that may miss subtle bugs a cloud model would catch. The review pass uses
the Claude API, not the local model.

## When to Use

Trigger when the user asks for:
- Editing or writing code files (MCP servers, plugins, scripts, services)
- Multi-step implementation tasks across multiple files
- "Have Claude review this" or "do a final pass" after coding

**Do NOT use for:**
- Single-line fixes or config-only changes (direct editing is faster)
- Tasks in repos with no git history (diff will be empty)

## The Process

### Phase 1 — Implement (Hermes)

Use your normal tools (`read_file`, `patch`, `terminal`) to implement the task.
Stage changes when done:

```bash
git add -A
```

### Phase 2 — Get the diff

```bash
git diff HEAD      # staged + unstaged vs last commit
# or
git diff --staged  # if already staged
```

If the diff is empty, check `git status` and warn the user.
If the diff is >15,000 characters, scope it to the most-changed file to stay
within Claude Code's input budget.

### Phase 3 — Claude Code review pass

Run via terminal with a hard spend cap:

```bash
claude --print \
  --permission-mode acceptEdits \
  --max-budget-usd 0.10 \
  --output-format text \
  -p "You are a code reviewer. Review this diff for bugs, logic errors, and
security issues. Be concise — list issues with file:line references and
severity (critical/important/minor). Do not rewrite the code.

<diff>
$(git diff HEAD)
</diff>"
```

Capture the exit code immediately after running:
- **Exit 0**: review output is valid — read it, proceed to Phase 4a
- **Non-zero** (any reason): log the error message, proceed to Phase 4b

### Phase 4a — Apply Claude Code findings

Parse Claude's output. Triage:
- **Critical** (security vulnerability, data loss, crash): fix before finishing
- **Important** (logic error, missed edge case): fix if straightforward; flag for user if complex
- **Minor** (style, naming): mention in summary, do not apply

If any critical or important fixes were applied, re-run tests:

```bash
python -m pytest -q 2>/dev/null || npm test 2>/dev/null || true
```

### Phase 4b — Fallback: Hermes self-review

If Claude Code exited non-zero for **any** reason, dispatch a fresh Hermes subagent
for an independent review:

```
delegate_task(
  goal="Review the following diff for bugs, security issues, and logic errors.
        Report issues with file:line references and severity (critical/important/minor).
        Do not fix anything — report only.",
  context="DIFF:\n<paste diff here>\n\nRepo: <absolute path>",
  toolsets=["file", "terminal", "web"]
)
```

Apply the same triage as Phase 4a.

### Phase 5 — Report to user

Summarize in one response:
- What was implemented
- Which reviewer ran (Claude Code or Hermes fallback) and why
- Issues found, fixes applied, items flagged for user decision

---

## Cost control

`--max-budget-usd 0.10` caps each review call at ~$0.10, covering roughly 200–400
lines of diff reviewed by Claude Sonnet.

| Scenario | Suggested cap |
|---|---|
| Routine fixes, <200 lines | `0.05` |
| Standard feature work | `0.10` (default) |
| Large refactor, >400 lines | `0.25` |
| Always use fallback | set `0.00` to force fallback every time |

If the cap fires frequently, mention it in the Phase 5 summary and suggest the
user adjust it or switch to fallback-only mode.

## Fallback triggers

Any non-zero exit from `claude --print` falls back — no exceptions:

- `--max-budget-usd` cap hit
- Rate limit exceeded (429)
- Auth error (API key missing or expired)
- Timeout or network error
- Any other failure

**Never block task completion on Claude Code availability.** The fallback always works.

## Prerequisites

Claude Code CLI must be installed on the host machine:

```bash
# Verify
claude --version

# Install if missing (requires npm)
npm install -g @anthropic-ai/claude-code
```

The `ANTHROPIC_API_KEY` environment variable must be set, or the user must be
logged in via `claude login`.
