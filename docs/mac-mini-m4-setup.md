# Hermes on Mac Mini M4 (24 GB) — Local Inference Setup

This guide documents running Hermes with a fully local inference backend on a Mac Mini M4 with 24 GB unified memory. No cloud API keys required.

**Target hardware:** Mac Mini M4 (Mac16,10), 24 GB unified memory, macOS Sequoia or later

---

## Overview

The stack:
- **Hermes** — AI gateway, Discord + CLI interface
- **Rapid-MLX** — Apple Silicon inference server (MLX-based, OpenAI-compatible)
- **Model** — Gemma 4 E4B 4-bit (`mlx-community/gemma-4-e4b-it-4bit`)
- **launchd** — keeps rapid-mlx running as a background service

Hermes talks to rapid-mlx at `http://127.0.0.1:8001/v1` as a custom OpenAI-compatible provider.

---

## 1. Install Hermes

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

Follow prompts. Hermes installs to `~/.hermes/hermes-agent/` and registers a launchd gateway service.

---

## 2. Install Rapid-MLX

```bash
brew install rapid-mlx
```

Verify:
```bash
rapid-mlx --version
```

### Install the mlx-vlm dependency

Gemma 4 requires `mlx-vlm` even when running in text-only mode. Install it into rapid-mlx's Homebrew Python environment:

```bash
/opt/homebrew/Cellar/rapid-mlx/$(rapid-mlx --version | awk '{print $1}')/libexec/bin/pip \
  install 'mlx-vlm>=0.4.4' --ignore-installed fsspec
```

The `--ignore-installed fsspec` flag is required because Homebrew pins fsspec to a version that conflicts with mlx-vlm's requirement. Without this, rapid-mlx will fail with `ImportError: Vision/multimodal models require the optional mlx-vlm dependency` when loading any Gemma 4 model.

> **If you upgrade rapid-mlx via Homebrew**, re-run this command — the Homebrew upgrade creates a fresh libexec environment.

---

## 3. Download the model

```bash
huggingface-cli download mlx-community/gemma-4-e4b-it-4bit
```

This downloads ~5.2 GB to `~/.cache/huggingface/hub/`. The model's working set at runtime is ~4.5 GB, leaving ~19 GB headroom for the OS and Hermes.

### Why E4B and not a larger model?

Hermes sends a system prompt with ~42 tool definitions on every request (~18,000–20,000 prompt tokens). Larger models require proportionally larger KV caches:

| Model | Disk | Working set | Notes |
|---|---|---|---|
| **gemma-4-e4b-it-4bit** | 5.2 GB | ~4.5 GB | **Recommended — ample headroom** |
| gemma-4-12B-it-4bit | 6.8 GB | ~6.0 GB | Slowest of the three (12.6 tok/s) |
| gemma-4-26b-a4b-it-4bit | 15.6 GB | ~22 GB | Crashes under Hermes's full prompt load |

The 26B model's KV cache for an 18K-token prompt exhausts Metal GPU memory and causes `SIGABRT` from `mlx::core::gpu::check_error`. E4B handles the same prompt easily.

---

## 4. Create the launchd plist

Create `~/Library/LaunchAgents/com.rapid-mlx.server.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.rapid-mlx.server</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/rapid-mlx</string>
        <string>serve</string>
        <string>mlx-community/gemma-4-e4b-it-4bit</string>
        <string>--port</string>
        <string>8001</string>
        <string>--no-mllm</string>
        <string>--max-tokens</string>
        <string>65536</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/.hermes/logs/rapid-mlx.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/.hermes/logs/rapid-mlx.log</string>
</dict>
</plist>
```

Replace `YOUR_USERNAME` with your macOS username (`whoami`).

**Critical flags:**
- `--no-mllm` — Routes Gemma 4 through the mlx-lm text path instead of mlx-vlm. Required even for text-only serving: without it, rapid-mlx attempts the VLM architecture path and raises `ValueError: Received 126 parameters not in model` due to a version mismatch between the mlx-community weights and mlx-vlm 0.6.3.
- `--max-tokens 65536` — Hermes sends `max_tokens=65536` on every request; without this cap, rapid-mlx can attempt to allocate a KV cache sized for the full 262K context window.
- `--port 8001` — Keeps port 8000 free for other services.

Load the service:
```bash
launchctl load ~/Library/LaunchAgents/com.rapid-mlx.server.plist
```

Verify it's running:
```bash
curl http://127.0.0.1:8001/v1/models
```

---

## 5. Configure Hermes

Edit `~/.hermes/config.yaml`:

```yaml
model:
  api_key: none
  base_url: http://127.0.0.1:8001/v1
  default: mlx-community/gemma-4-e4b-it-4bit
  provider: custom

agent:
  disabled_toolsets:
    - image_gen
    - video_gen
    - computer_use
    - vision
    - tts
  environment_hint: "User is in Seattle, WA — default for weather, events, time. On routine tasks (cron, jobs, traceability, scheduling): act on sensible defaults, ask only when ambiguity changes which tool to call."

auxiliary:
  compression:
    provider: custom
    base_url: http://127.0.0.1:8001/v1
    model: mlx-community/gemma-4-e4b-it-4bit
    api_key: none

web:
  backend: ddgs
  search_backend: ddgs
  extract_backend: ddgs

toolsets:
  - hermes-cli
  - web
  - browser
```

**Key config notes:**
- `provider: custom` with `base_url` pointing directly to rapid-mlx on port 8001. Do not use `provider: ollama` for the auxiliary compression client — it doesn't recognize that provider name.
- Disabled toolsets skip image generation, vision, TTS — not useful for a text-only local model. `code_execution` is left **enabled** (it's useful and Gemma 4 handles it fine).
- `environment_hint` injects operator context into every system prompt — use it to pre-answer recurring questions (location, preferred defaults) so the model doesn't ask.
- No `context_length` override needed — Gemma 4 natively reports 262144 tokens, well above Hermes's 64K minimum.
- No `reasoning_effort` override needed — let Hermes manage it via the gemma4 reasoning parser.

Restart Hermes to pick up config changes:
```bash
hermes gateway restart
```

---

## 6. Verify everything

```bash
# rapid-mlx is up and serving E4B
curl -s http://127.0.0.1:8001/v1/models | python3 -c \
  "import sys,json; m=json.load(sys.stdin); print(m['data'][0]['id'], m['data'][0]['modality'])"
# → mlx-community/gemma-4-e4b-it-4bit text

# Quick inference test
curl -s http://127.0.0.1:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"mlx-community/gemma-4-e4b-it-4bit","messages":[{"role":"user","content":"Say hi."}],"max_tokens":10}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['choices'][0]['message']['content'])"

# Hermes is connected
hermes gateway status
```

---

## Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| `ImportError: Vision/multimodal models require the optional mlx-vlm dependency` | mlx-vlm not installed in rapid-mlx's env | Run the pip install command from Step 2 |
| `ValueError: Received 126 parameters not in model` | mlx-vlm version mismatch with model weights | Add `--no-mllm` to plist ProgramArguments |
| All output in `reasoning_content`, TTFT 8–10s | Gemma 4 chain-of-thought consuming all tokens | Hermes's gemma4 reasoning parser handles this; or add `--no-thinking` to disable CoT entirely |
| `SIGABRT` / Metal GPU crash | KV cache for large prompt exhausts GPU memory | Switch to E4B model (smaller working set) |
| `Provider returned an empty stream with no finish_reason` | Proxy/connection dropped mid-response | Point `base_url` directly to port 8001, skip any proxy |
| Hermes `context window below minimum 64K` | Model reports a small context | Add `context_length: 65536` under `model:` in config |

---

## Updating rapid-mlx

```bash
brew upgrade rapid-mlx
# Re-install mlx-vlm into the new libexec env (version number changes)
/opt/homebrew/Cellar/rapid-mlx/<NEW_VERSION>/libexec/bin/pip \
  install 'mlx-vlm>=0.4.4' --ignore-installed fsspec
launchctl unload ~/Library/LaunchAgents/com.rapid-mlx.server.plist
launchctl load ~/Library/LaunchAgents/com.rapid-mlx.server.plist
```

## MCP Servers

### Vestaboard

Repo: https://github.com/kfezer/vestaboard-mcp

```bash
cd ~/hermes/vestaboard-mcp
uv venv && uv pip install -r requirements.txt
```

Add to `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  vestaboard:
    command: /Users/YOUR_USERNAME/hermes/vestaboard-mcp/.venv/bin/python3
    args:
      - /Users/YOUR_USERNAME/hermes/vestaboard-mcp/vestaboard_mcp.py
    env:
      VESTABOARD_HOST: http://<board-ip>:7000
      VESTABOARD_API_KEY: <your-key>
    enabled: true
```

Use the board's IP address directly — mDNS can be unreliable. To get the API key, POST to `/local-api/enablement` with the enablement token shown on the board.

**Tools:**

| Tool | Description |
|---|---|
| `vestaboard_weather_forecast` | Fetches live weather. Standard mode: icon for 4s then full 3-day data. `compact=True`: icon + condensed today view side by side |
| `vestaboard_icon_text` | Generic icon + text layout: 6×8 icon on the left, 13 chars of free-form text on the right |
| `vestaboard_send_text` | Send centered text (up to 6 rows × 22 chars) |
| `vestaboard_send_long_text` | Auto-splits long text across multiple screens, cycles in background |
| `vestaboard_show_icon` | Display a full-board weather icon (6×22): sunny, cloudy, rainy, snowy, stormy, foggy, windy, hot |
| `vestaboard_read` | Read current board contents |
| `vestaboard_clear` | Blank the display |
| `vestaboard_send_raw` | Send a raw 6×22 color-code grid |

---

## Tuning for local models

Local models running as `provider: custom` need a few extra nudges that cloud models get automatically.

### Reduce clarifying questions (act-don't-ask patch)

By default, the `<act_dont_ask>` guidance — which tells Hermes to act on obvious defaults instead of asking — is only injected for OpenAI/Grok model families. Gemma and other local models miss it, causing repeated clarifying questions for routine tasks.

**Fix:** Edit `~/.hermes/hermes-agent/agent/system_prompt.py` line 255:

```python
# before
if "gpt" in _model_lower or "codex" in _model_lower or "grok" in _model_lower:
# after
if "gpt" in _model_lower or "codex" in _model_lower or "grok" in _model_lower or "gemma" in _model_lower:
```

Then restart: `hermes gateway restart`

This applies to `mlx-community/gemma-4-e4b-it-4bit` and any other Gemma variant.

### Pre-populate user profile

Hermes loads `~/.hermes/memories/USER.md` into every system prompt. Pre-filling it with known facts means the model never asks about them. Create the file if it doesn't exist:

```
User is in Seattle, WA. Use Seattle as default for weather, events, time, and location queries.
§
For routine tasks (cron, scheduling, traceability): act on sensible defaults, don't ask.
§
All Python work uses uv: `uv venv` and `uv pip install`.
```

Entries are separated by `§` (section sign). The store is limited to 1,375 chars — keep entries short.

---

## Claude Code Delegate skill

This skill lets Hermes implement code and then hand the final diff to Claude Code CLI for an independent review — useful because local models (Gemma 4, Qwen, etc.) may miss subtle bugs that a cloud model catches. If Claude Code is unavailable or hits its budget, a fresh Hermes subagent reviews instead.

The skill is bundled in the hermes-agent repo at `skills/software-development/claude-code-delegate/` and loads automatically.

### Prerequisites

Claude Code CLI must be installed on the Mac Mini:

```bash
# Install
npm install -g @anthropic-ai/claude-code

# Verify
claude --version

# Authenticate (stores key in keychain)
claude login
```

The skill uses `--max-budget-usd 0.10` per review call by default (~$0.10 covers 200–400 lines via Claude Sonnet). Adjust the cap in the skill file if needed.

### How to trigger

Hermes auto-triggers it for multi-step coding tasks. You can also invoke it explicitly:

- "Use the claude-code-delegate skill to implement X"
- "Implement X, then have Claude Code review the final diff"
- "Do a final review pass on what you just wrote"

Single-line fixes and config-only changes skip it — direct editing is faster.

### How it works

1. **Hermes implements** using its own tools (read_file, patch, terminal)
2. **Gets the diff** (`git diff HEAD`)
3. **Runs** `claude --print --max-budget-usd 0.10 -p "Review this diff..."` with a hard spend cap
4. **Applies findings**: critical issues fixed immediately; important issues fixed or flagged; minor issues noted in summary only
5. **Fallback**: if `claude` exits non-zero for any reason (rate limit, budget hit, auth error), a fresh Hermes subagent does the review instead — task completion is never blocked

---

## Dream — weekly self-reflection cron

Dream is a weekly cron job that reviews recent sessions and updates memory. The primary goal is **memory hygiene and distillation**, not accumulation — each run should leave memory the same size or smaller.

Create the job:

```bash
hermes cron create "0 3 * * 0" \
  "You are running a weekly Dream pass — a memory hygiene and distillation job.

CONTEXT: This deployment runs a small local model (Gemma 4 E4B) with hard memory limits: 2,200 chars for MEMORY, 1,375 chars for USER. Your primary job is to keep memory lean, high-signal, and below 60% capacity. Adding knowledge is secondary to compression and consolidation.

Step 1 — Check current state (ALWAYS DO THIS FIRST)
Call memory(action='read', target='memory') and memory(action='read', target='user'). The read output shows current usage as 'X/2,200 chars' and 'X/1,375 chars'. Record both percentages before proceeding.

Step 2 — Review recent sessions
Call session_search(sort='newest', limit=10) to browse session titles and previews. For 2-4 substantive sessions, call session_search(session_id='...') to read them. Note patterns: recurring corrections, consistent workflows, repeated questions.

Step 3 — Decide mode based on usage
- MEMORY < 60% AND USER < 60%: consolidation pass + up to 2 new entries if novel
- Either store 60-80%: consolidation-only, no new entries
- Either store > 80%: emergency compression, target dropping below 60%

Step 4 — Update memory with atomic operations
Use memory(operations=[...]) for all changes in a single atomic batch. The batch checks the NET final state — removing 3 entries + adding 1 is valid even if the add alone would overflow.

Consolidation rules: merge entries on the same topic; remove superseded or session-specific entries; rewrite verbose entries as terse rules under 80 chars; after the batch, total chars must be <= Step 1.

Addition rules (only when below 60%): max 2 new entries per run; only if the pattern appeared in >= 2 sessions and is not already captured; prefer behavioral rules over one-off facts.

Step 5 — End with [SILENT]" \
  --name "Dream" --deliver local
```

After creating, restrict toolsets to keep the job's own context lean:

```bash
# Get the job ID
hermes cron list

# Edit jobs.json to add enabled_toolsets (CLI doesn't expose this yet)
python3 -c "
import json
path = '$HOME/.hermes/cron/jobs.json'
with open(path) as f: data = json.load(f)
for j in data['jobs']:
    if j['name'] == 'Dream':
        j['enabled_toolsets'] = ['session_search', 'memory']
        print('Set toolsets on:', j['id'])
with open(path, 'w') as f: json.dump(data, f, indent=2)
"
```

**Test a manual run:**
```bash
hermes cron run <job-id>
# Output lands in ~/.hermes/cron/output/<job-id>/
```

**Memory tiering explained:**
- < 60%: consolidate existing entries, then add up to 2 genuinely new ones
- 60–80%: consolidation only — merge/remove, no net adds
- > 80%: emergency compression — aggressively consolidate, target dropping below 60%

The memory tool hard-refuses adds that would exceed limits, returning all current entries and "consolidate now." The `operations` batch evaluates net final state, so removing two entries and adding one condensed replacement is valid in a single atomic call.

---

## Performance reference (Mac Mini M4, 24 GB)

Benchmarked with 3-run average, 300-token response, Hermes system prompt (~18K tokens):

| Model | TTFT | tok/s |
|---|---|---|
| gemma-4-e4b-it-4bit | ~0.22s | ~28 |
| gemma-4-12B-it-4bit | ~0.44s | ~12 |
| qwen3-8b-4bit (previous) | ~0.18s | ~22 |
