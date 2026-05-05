# deepclaude

This is a fork of https://github.com/aattaran/deepclaude

Run Claude Code's full autonomous agent loop with [openrouter.ai](https://openrouter.ai) **DeepSeek V4 Flash** - same tools, same UX, dramatically lower cost.

![Remote control running DeepSeek V4 Flash in the browser](screenshots/remote-control-deepseek.png)

## What it does

Claude Code is the best coding agent, but its $200/month plan has usage caps and costs $15/M output tokens. **deepclaude** keeps the Claude Code shell (file editing, bash, git, subagents, tool loops) and replaces only the AI model:

```
Your terminal
  └── Claude Code CLI  (file editing, bash, git, tool loops — unchanged)
        └── API calls → DeepSeek V4 Flash via OpenRouter  ($0.28/M, not $15/M)
```

DeepSeek V4 Flash scores 79.0% on SWE-Bench. For coding tasks it's 80% comparable to Claude Sonnet 4.6 (79.6%) and Claude Opus 4.6 (80.8%). You can switch back to Anthropic mid-session for the hard 20%.

## Quick start

### 1. Get an OpenRouter API key

Sign up at [openrouter.ai](https://openrouter.ai), add credit, copy your API key. OpenRouter is an AI API aggregator that gives you access to DeepSeek and dozens of other models through a single key, hosted on US infrastructure.

```bash
export OPENROUTER_API_KEY="sk-or-..."   # macOS/Linux
setx OPENROUTER_API_KEY "sk-or-..."     # Windows PowerShell
```

### 2. Install

```bash
chmod +x deepclaude.sh
sudo ln -s "$(pwd)/deepclaude.sh" /usr/local/bin/deepclaude
```

**Windows:** copy `deepclaude.ps1` to a directory in your PATH.

### 3. Launch

```bash
deepclaude                    # OpenRouter + DeepSeek V4 Flash (default)
deepclaude --status           # Show configured keys and active proxy
deepclaude --cost             # Pricing comparison across backends
deepclaude --benchmark        # Latency test across all providers
```

## How the model is selected

deepclaude sets Claude Code's model environment variables before launch so every request — including subagents — goes to DeepSeek:

| Variable | Set to |
|---|---|
| `ANTHROPIC_BASE_URL` | OpenRouter (or DeepSeek/Fireworks) endpoint |
| `ANTHROPIC_AUTH_TOKEN` | Your provider API key |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `deepseek/deepseek-v4-flash` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `deepseek/deepseek-v4-flash` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `deepseek/deepseek-v4-flash` |
| `CLAUDE_CODE_SUBAGENT_MODEL` | `deepseek/deepseek-v4-flash` |

These are set only for the current session — your shell environment is restored when you exit.

Claude Code sometimes sends Anthropic model names (e.g. `claude-opus-4-6`) regardless of these vars. The local proxy intercepts those requests and **remaps the model name** before forwarding to the active backend:

```
Claude Code  →  localhost:3200 (proxy)
                  ├── /v1/messages  →  remap model name  →  DeepSeek V4 Flash
                  └── everything else  →  Anthropic (passthrough)
```

## OpenRouter

[OpenRouter](https://openrouter.ai) is an API aggregator that provides a single OpenAI/Anthropic-compatible endpoint for dozens of models from different providers. Key properties:

- **Single key**: access DeepSeek, Llama, Mistral, and others through one API key
- **US infrastructure**: lower latency than hitting DeepSeek's servers directly (especially from Europe)
- **Provider routing**: deepclaude routes DeepSeek V4 Flash through AtlasCloud for consistent performance
- **Same price as direct**: $0.44/M input, $0.87/M output — no markup vs DeepSeek direct

OpenRouter is the default backend (`CHEAPCLAUDE_DEFAULT_BACKEND=or`). You can change the default by setting that environment variable.

## Backends

| Backend | Flag | Input/M | Output/M | Notes |
|---|---|---|---|---|
| **OpenRouter DeepSeek Pro** | `-b or` | $0.49 | $2.26 | US servers, best latency from US/EU |
| **OpenRouter DeepSeek Flash** (default) | `-b or` | $0.14 | $0.28 | US servers, best latency from US/EU |
| **DeepSeek** | `-b ds` | $0.44 | $0.87 | Direct to DeepSeek; auto context caching (120x cheaper on cache hits) |
| **Fireworks AI** | `-b fw` | $1.74 | $3.48 | Fastest inference, US servers |
| **Anthropic** | `-b anthropic` | $3.00 | $15.00 | Original Claude Opus — use for hard problems |

Switch backend at launch:

```bash
deepclaude -b ds          # DeepSeek direct
deepclaude -b fw          # Fireworks AI
deepclaude -b anthropic   # Normal Claude Code
```

### Setup per backend

**OpenRouter** (default):
```bash
export OPENROUTER_API_KEY="sk-or-..."
```

**DeepSeek direct** (optional):
```bash
export DEEPSEEK_API_KEY="sk-..."
```

**Fireworks AI** (optional):
```bash
export FIREWORKS_API_KEY="fw_..."
```

## Live switching (no restart)

Switch backends **mid-session** — from inside Claude Code itself, without restarting.

### Option 1: Slash commands (recommended)

Add these files to `~/.claude/commands/`:

**`openrouter.md`:**
```
Switch the model proxy to OpenRouter. Run this command silently and report the result:
curl -sX POST http://127.0.0.1:3200/_proxy/mode -d "backend=openrouter"
If successful, say: "Switched to OpenRouter."
```

**`deepseek.md`:**
```
Switch the model proxy to DeepSeek. Run this command silently and report the result:
curl -sX POST http://127.0.0.1:3200/_proxy/mode -d "backend=deepseek"
If successful, say: "Switched to DeepSeek."
```

**`anthropic.md`:**
```
Switch the model proxy back to Anthropic. Run this command silently and report the result:
curl -sX POST http://127.0.0.1:3200/_proxy/mode -d "backend=anthropic"
If successful, say: "Switched to Anthropic."
```

Type `/openrouter`, `/deepseek`, or `/anthropic` inside any Claude Code session to switch instantly.

**In Claude Code terminal:**

![/deepseek in Claude Code CLI](screenshots/terminal%20for%20terminal%20embed2.PNG)

**In Claude Code VS Code extension:**

![/deepseek in VS Code extension](screenshots/terminal%20for%20vscode%20embed2.PNG)

### Option 2: CLI flag

```bash
deepclaude --switch openrouter   # or: or, ds, fw, anthropic
deepclaude -s anthropic
```

### Option 3: VS Code keyboard shortcuts

Add to `.vscode/tasks.json`:
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Proxy: Switch to DeepSeek",
      "type": "shell",
      "command": "Invoke-RestMethod -Uri http://127.0.0.1:3200/_proxy/mode -Method Post -Body 'backend=deepseek'",
      "presentation": { "reveal": "always" },
      "problemMatcher": []
    },
    {
      "label": "Proxy: Switch to Anthropic",
      "type": "shell",
      "command": "Invoke-RestMethod -Uri http://127.0.0.1:3200/_proxy/mode -Method Post -Body 'backend=anthropic'",
      "presentation": { "reveal": "always" },
      "problemMatcher": []
    }
  ]
}
```

Bind in `keybindings.json`:
```json
{ "key": "ctrl+alt+d", "command": "workbench.action.tasks.runTask", "args": "Proxy: Switch to DeepSeek" },
{ "key": "ctrl+alt+a", "command": "workbench.action.tasks.runTask", "args": "Proxy: Switch to Anthropic" }
```

## Cost tracking

The proxy tracks token usage and calculates savings vs Anthropic:

```bash
curl -s http://127.0.0.1:3200/_proxy/cost
```

```json
{
  "backends": {
    "openrouter": {
      "input_tokens": 125000,
      "output_tokens": 45000,
      "requests": 12,
      "cost": 0.0941,
      "anthropic_equivalent": 1.05
    }
  },
  "total_cost": 0.0941,
  "anthropic_equivalent": 1.05,
  "savings": 0.9559
}
```

## Cost comparison

| Usage level | Anthropic Max | deepclaude (OpenRouter) | Savings |
|---|---|---|---|
| Light (10 days/mo) | $200/mo (capped) | ~$20/mo | 90% |
| Heavy (25 days/mo) | $200/mo (capped) | ~$50/mo | 75% |
| With auto loops | $200/mo (capped) | ~$80/mo | 60% |

DeepSeek direct (`-b ds`) has automatic context caching: after the first turn, the system prompt and file context are re-read at $0.004/M instead of $0.44/M — a 120x reduction for long sessions.

## What works and what doesn't

### Works
- File reading, writing, editing (Read/Write/Edit tools)
- Bash/shell execution
- Glob and Grep search
- Multi-step autonomous tool loops
- Subagent spawning
- Git operations
- Project initialization (`/init`)
- Thinking mode (enabled by default)

### Limitations

| Feature | Status |
|---|---|
| Image/vision input | Not supported — DeepSeek's Anthropic-compatible endpoint doesn't accept images |
| MCP server tools | Not supported through the compatibility layer |
| Prompt caching headers | DeepSeek/OpenRouter ignore Anthropic's `cache_control` — DeepSeek direct uses its own automatic caching instead |

## Remote control (`--remote`)

Open a Claude Code session in any browser with DeepSeek as the brain:

```bash
deepclaude --remote              # Remote control + OpenRouter (default)
deepclaude --remote -b ds        # Remote control + DeepSeek direct
deepclaude --remote -b anthropic # Remote control + Anthropic (normal)
```

This prints a `https://claude.ai/code/session_...` URL you can open on any device.

Remote control needs Anthropic's WebSocket bridge, but model calls are still intercepted by the local proxy:

```
claude remote-control
  ├── Bridge WebSocket  →  wss://bridge.claudeusercontent.com  (Anthropic, hardcoded)
  └── Model API calls   →  http://localhost:3200 (proxy)
                              ├── /v1/messages  →  DeepSeek V4 Flash
                              └── everything else  →  Anthropic (passthrough)
```

**Prerequisites:**
- `claude auth login` (needed for the bridge)
- A claude.ai subscription (Anthropic infrastructure)
- Node.js 18+

## VS Code / Cursor integration

Add a terminal profile so you can launch deepclaude from the IDE:

```json
{
  "terminal.integrated.profiles.osx": {
    "DeepSeek Agent": {
      "path": "/usr/local/bin/deepclaude"
    }
  }
}
```

Windows:
```json
{
  "terminal.integrated.profiles.windows": {
    "DeepSeek Agent": {
      "path": "powershell.exe",
      "args": ["-ExecutionPolicy", "Bypass", "-NoExit", "-File", "C:\\path\\to\\deepclaude.ps1"]
    }
  }
}
```

## License

MIT
