<div align="center">

# Free Claude Code

Run Claude Code through a local Anthropic-compatible proxy and route requests to hosted or local model providers.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)
[![Python 3.14](https://img.shields.io/badge/python-3.14-3776ab.svg?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/downloads/)
[![uv](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/uv/main/assets/badge/v0.json&style=for-the-badge)](https://github.com/astral-sh/uv)
[![Tested with Pytest](https://img.shields.io/badge/testing-Pytest-00c0ff.svg?style=for-the-badge)](https://github.com/Alishahryar1/free-claude-code/actions/workflows/tests.yml)
[![Type checking: Ty](https://img.shields.io/badge/type%20checking-ty-ffcc00.svg?style=for-the-badge)](https://pypi.org/project/ty/)
[![Code style: Ruff](https://img.shields.io/badge/code%20formatting-ruff-f5a623.svg?style=for-the-badge)](https://github.com/astral-sh/ruff)

[Quick start](#quick-start) · [How it works](#how-it-works) · [Providers](#providers) · [Clients](#connect-clients) · [Security](#security-notes) · [Roadmap](#development-roadmap)

</div>

<div align="center">
  <img src="assets/pic.png" alt="Free Claude Code in action" width="700">
</div>

## What This Project Does

Free Claude Code is a local FastAPI proxy that exposes a Claude/Anthropic-compatible API for Claude Code clients. It receives Claude Code requests at a local URL, chooses a configured model/provider, converts request and streaming response shapes when needed, and returns output in the form Claude Code expects.

Use it when you want to:

- Run Claude Code against non-Anthropic providers or local models.
- Route Opus, Sonnet, Haiku, and fallback requests to different providers.
- Use one local Admin UI to manage provider keys, model routing, messaging bots, voice transcription, and diagnostics.
- Run Claude Code sessions from the terminal, VS Code extension, JetBrains ACP, or optional Discord/Telegram bots.

It is **not** an Anthropic Max subscription replacement. It can approximate a high-throughput coding setup by multiplexing providers and model tiers, but quota, model quality, context limits, and tool behavior still depend on the upstream providers you configure.

## Quick Start

### 1. Install

The installers set up Claude Code if needed, install or update `uv`, install Python 3.14, and install Free Claude Code.

macOS/Linux:

```bash
curl -fsSL "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.sh?raw=1" | sh
```

Windows PowerShell:

```powershell
irm "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.ps1?raw=1" | iex
```

Review the scripts before running remote installers:

- [`scripts/install.sh`](scripts/install.sh)
- [`scripts/install.ps1`](scripts/install.ps1)

### 2. Start The Proxy

```bash
fcc-server
```

By default, the server listens on port `8082`. The code default host is `0.0.0.0`; the Admin UI itself is still protected to loopback clients only. See [Security Notes](#security-notes) before exposing the proxy on a network.

When healthy, the server opens or prints the Admin UI URL:

```text
http://127.0.0.1:8082/admin
```

### 3. Configure A Provider

Open the Admin UI, select **Providers**, set the API key or local base URL for your provider, and set `MODEL` to a provider-prefixed model reference:

```text
provider_id/model-id-from-that-provider
```

Examples of valid prefixes are `nvidia_nim/`, `open_router/`, `gemini/`, `deepseek/`, `mistral/`, `mistral_codestral/`, `opencode/`, `opencode_go/`, `wafer/`, `kimi/`, `cerebras/`, `groq/`, `fireworks/`, `zai/`, `lmstudio/`, `llamacpp/`, and `ollama/`.

Model IDs change frequently. Prefer the provider's model catalog or the Admin UI provider check over copying old examples from the internet.

### 4. Launch Claude Code Through The Proxy

```bash
fcc-claude
```

`fcc-claude` checks that the proxy is reachable, locates the real `claude` binary, removes stale `ANTHROPIC_*` variables from the child environment, then launches Claude Code with:

```text
ANTHROPIC_BASE_URL=http://127.0.0.1:8082  # or your configured local URL
ANTHROPIC_AUTH_TOKEN=<configured token, if non-empty>
CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1
CLAUDE_CODE_AUTO_COMPACT_WINDOW=190000
```

Keep `fcc-server` running while you use Claude Code.

## How It Works

<div align="center">
  <img src="assets/how-it-works.svg" alt="Free Claude Code request flow architecture" width="900">
</div>

Source: [`assets/how-it-works.mmd`](assets/how-it-works.mmd).

Runtime flow:

1. A Claude-compatible client sends an Anthropic Messages request to the local proxy.
2. The proxy authenticates the request when `ANTHROPIC_AUTH_TOKEN` is configured.
3. The model router maps the incoming Claude model name to `MODEL_OPUS`, `MODEL_SONNET`, `MODEL_HAIKU`, or fallback `MODEL`.
4. Fast-path optimizations answer selected Claude Code probes locally when enabled.
5. A provider adapter sends the request upstream using either OpenAI Chat Completions or Anthropic Messages transport.
6. Streaming, tool calls, thinking/reasoning blocks, token usage, and provider errors are normalized into Claude-compatible responses.

### Exposed API Routes

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/messages` | Accepts Anthropic Messages-style payloads and returns Claude-compatible message output. |
| `HEAD`, `OPTIONS` | `/v1/messages` | Compatibility probes. |
| `POST` | `/v1/messages/count_tokens` | Counts tokens for a Messages-style request. |
| `HEAD`, `OPTIONS` | `/v1/messages/count_tokens` | Compatibility probes. |
| `GET` | `/v1/models` | Lists configured and discovered gateway model IDs for clients that enable model discovery. |
| `GET` | `/` | Basic status with current fallback provider/model. Requires auth if token is configured. |
| `GET` | `/health` | Unauthenticated health check. |
| `POST` | `/stop` | Stops active CLI/messaging tasks. |
| `GET` | `/admin` | Local-only Admin UI. |

## Configuration

Free Claude Code loads settings from:

1. `.env` in the current working directory.
2. Managed user config at `~/.fcc/.env`.
3. Optional explicit file from `FCC_ENV_FILE`.
4. Process environment variables.

`fcc-init` creates `~/.fcc/.env` from `.env.example` if it does not exist. `fcc-server` also migrates legacy user env files from:

- `~/free-claude-code/.env`
- `~/.config/free-claude-code/.env`

The Admin UI writes supported managed settings back to `~/.fcc/.env`.

### Important Settings

| Setting | Purpose |
|---|---|
| `MODEL` | Fallback provider-prefixed model reference. |
| `MODEL_OPUS` | Optional route for incoming Claude Opus model names. |
| `MODEL_SONNET` | Optional route for incoming Claude Sonnet model names. |
| `MODEL_HAIKU` | Optional route for incoming Claude Haiku model names. |
| `ANTHROPIC_AUTH_TOKEN` | Optional client auth token. Empty means proxy API auth is disabled. |
| `PROVIDER_RATE_LIMIT` / `PROVIDER_RATE_WINDOW` | Provider request rate window. |
| `PROVIDER_MAX_CONCURRENCY` | Per-process upstream concurrency cap. |
| `ENABLE_MODEL_THINKING` | Fallback thinking/reasoning switch. |
| `ENABLE_OPUS_THINKING` / `ENABLE_SONNET_THINKING` / `ENABLE_HAIKU_THINKING` | Per-tier thinking overrides. |
| `ENABLE_WEB_SERVER_TOOLS` | Enables local handling for `web_search` / `web_fetch`; see security notes. |
| `WEB_FETCH_ALLOWED_SCHEMES` | Comma-separated allowed URL schemes for `web_fetch`. |
| `WEB_FETCH_ALLOW_PRIVATE_NETWORKS` | Allows private/loopback/link-local fetch targets when true. Use only in trusted labs. |

## Providers

Provider IDs are canonical. Use them as the prefix before the upstream model ID.

| Provider ID | Upstream type | Credential / config | Default base URL | Notes |
|---|---|---|---|---|
| `nvidia_nim` | OpenAI Chat | `NVIDIA_NIM_API_KEY` | `https://integrate.api.nvidia.com/v1` | Hosted NVIDIA NIM gateway. |
| `open_router` | Anthropic Messages | `OPENROUTER_API_KEY` | `https://openrouter.ai/api/v1` | OpenRouter gateway. |
| `gemini` | OpenAI Chat | `GEMINI_API_KEY` | `https://generativelanguage.googleapis.com/v1beta/openai/` | Google AI Studio OpenAI-compatible endpoint. |
| `deepseek` | Anthropic Messages | `DEEPSEEK_API_KEY` | `https://api.deepseek.com/anthropic` | DeepSeek Anthropic-compatible path. |
| `mistral` | OpenAI Chat | `MISTRAL_API_KEY` | `https://api.mistral.ai/v1` | Mistral La Plateforme. |
| `mistral_codestral` | OpenAI Chat | `CODESTRAL_API_KEY` | `https://codestral.mistral.ai/v1` | Codestral key is separate from La Plateforme. |
| `opencode` | OpenAI Chat | `OPENCODE_API_KEY` | `https://opencode.ai/zen/v1` | OpenCode Zen. |
| `opencode_go` | OpenAI Chat | `OPENCODE_API_KEY` | `https://opencode.ai/zen/go/v1` | Shares OpenCode key; different upstream path. |
| `wafer` | Anthropic Messages | `WAFER_API_KEY` | `https://pass.wafer.ai/v1` | Wafer Pass. |
| `kimi` | Anthropic Messages | `KIMI_API_KEY` | `https://api.moonshot.ai/anthropic/v1` | Moonshot/Kimi Anthropic-compatible path. |
| `cerebras` | OpenAI Chat | `CEREBRAS_API_KEY` | `https://api.cerebras.ai/v1` | Cerebras OpenAI-compatible endpoint. |
| `groq` | OpenAI Chat | `GROQ_API_KEY` | `https://api.groq.com/openai/v1` | Groq OpenAI-compatible endpoint. |
| `fireworks` | Anthropic Messages | `FIREWORKS_API_KEY` | `https://api.fireworks.ai/inference/v1` | Fireworks Messages endpoint. |
| `zai` | Anthropic Messages | `ZAI_API_KEY` | `https://api.z.ai/api/anthropic/v1` | Z.ai Anthropic-compatible path. |
| `lmstudio` | Anthropic Messages | `LM_STUDIO_BASE_URL` | `http://localhost:1234/v1` | Local provider; no API key required. |
| `llamacpp` | Anthropic Messages | `LLAMACPP_BASE_URL` | `http://localhost:8080/v1` | Local `llama-server`; requires compatible endpoint/features. |
| `ollama` | Anthropic Messages | `OLLAMA_BASE_URL` | `http://localhost:11434` | Local Ollama root URL; do not append `/v1`. |

Most adapters advertise chat, streaming, tool, and thinking support, but actual behavior depends on the upstream model. Local models in particular must support the requested context size and tool-use patterns for Claude Code workflows.

### Per-Tier Routing

Claude Code sends Claude model names. Free Claude Code classifies them by substring and resolves them as follows:

1. Names containing `opus` use `MODEL_OPUS` when set.
2. Names containing `haiku` use `MODEL_HAIKU` when set.
3. Names containing `sonnet` use `MODEL_SONNET` when set.
4. Everything else uses `MODEL`.

This lets you send expensive/high-reasoning work to one provider and routine work to another. Blank tier variables inherit `MODEL`.

## Connect Clients

### Claude Code CLI

Recommended:

```bash
fcc-claude
```

You can pass normal Claude Code arguments after it:

```bash
fcc-claude --help
```

### VS Code Claude Code Extension

Open VS Code settings JSON and set:

```json
"claudeCode.environmentVariables": [
  { "name": "ANTHROPIC_BASE_URL", "value": "http://localhost:8082" },
  { "name": "ANTHROPIC_AUTH_TOKEN", "value": "freecc" },
  { "name": "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY", "value": "1" },
  { "name": "CLAUDE_CODE_AUTO_COMPACT_WINDOW", "value": "190000" }
]
```

Use your configured port and token. Reload the extension after editing settings.

### JetBrains ACP

Edit the installed Claude ACP config:

- Windows: `C:\Users\%USERNAME%\AppData\Roaming\JetBrains\acp-agents\installed.json`
- Linux/macOS: `~/.jetbrains/acp.json`

Set the environment for `acp.registry.claude-acp`:

```json
"env": {
  "ANTHROPIC_BASE_URL": "http://localhost:8082",
  "ANTHROPIC_AUTH_TOKEN": "freecc",
  "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY": "1",
  "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "190000"
}
```

Restart the IDE.

### Claude Desktop / Other Claude Applications

This project is designed for clients that let you override Anthropic API base URL and auth token. Claude Code CLI, VS Code extension, JetBrains ACP, and the included messaging bot flow can do that.

Claude Desktop does not expose a documented, stable custom Anthropic API base URL mechanism in this repository. Do not assume it can be pointed at this proxy unless the specific Desktop build or wrapper you use supports those environment variables.

## Optional Integrations

### Discord And Telegram Bots

The messaging layer can run Claude Code sessions from Discord or Telegram, stream updates, support reply-based branches, and persist session trees.

Configure in the Admin UI:

1. Open `/admin` locally.
2. Go to **Messaging**.
3. Choose `discord`, `telegram`, or `none`.
4. Set the bot token and allowed channel/user IDs.
5. Set `ALLOWED_DIR` to the absolute workspace root the bot may use.
6. Validate and apply.

Supported slash commands in code:

- `/stop` — cancel tasks globally or for a replied branch.
- `/clear` — clear session state globally or for a replied branch.
- `/stats` — show session/task state.

Only configured Discord channels or the configured Telegram user should be allowed to interact with the bot.

### Voice Notes

Voice notes are supported for Discord/Telegram when optional dependencies are installed.

macOS/Linux:

```bash
# NVIDIA NIM transcription / Riva gRPC
curl -fsSL "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.sh?raw=1" | sh -s -- --voice-nim

# Local Whisper
curl -fsSL "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.sh?raw=1" | sh -s -- --voice-local

# Both backends
curl -fsSL "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.sh?raw=1" | sh -s -- --voice-all

# Local Whisper with CUDA wheels
curl -fsSL "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.sh?raw=1" | sh -s -- --voice-local --torch-backend cu130
```

Windows PowerShell:

```powershell
# NVIDIA NIM transcription / Riva gRPC
& ([scriptblock]::Create((irm "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.ps1?raw=1"))) -VoiceNim

# Local Whisper
& ([scriptblock]::Create((irm "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.ps1?raw=1"))) -VoiceLocal

# Both backends
& ([scriptblock]::Create((irm "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.ps1?raw=1"))) -VoiceAll

# Local Whisper with CUDA wheels
& ([scriptblock]::Create((irm "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.ps1?raw=1"))) -VoiceLocal -TorchBackend cu130
```

Then configure:

- `VOICE_NOTE_ENABLED`
- `WHISPER_DEVICE`: `cpu`, `cuda`, or `nvidia_nim`
- `WHISPER_MODEL`
- `HF_TOKEN` when needed for local model downloads
- `NVIDIA_NIM_API_KEY` when `WHISPER_DEVICE=nvidia_nim`

## Security Notes

Read this section before binding the proxy outside your machine.

- **Proxy host default:** code currently defaults to `0.0.0.0`, which can listen on all interfaces. Configure firewall rules and `ANTHROPIC_AUTH_TOKEN` before exposing it to a LAN.
- **Admin UI:** `/admin` and `/admin/api/*` require loopback client and local origin checks. It is intended for local use only.
- **API auth:** if `ANTHROPIC_AUTH_TOKEN` is empty, proxy API routes do not require a token. Set a strong token for any non-local use.
- **Web tools:** `ENABLE_WEB_SERVER_TOOLS` allows proxy-side `web_search` / `web_fetch` handling. `web_fetch` can perform outbound HTTP from the proxy host. Keep private-network blocking enabled unless you explicitly need lab access.
- **Logs:** raw payload and diagnostic flags can leak prompts, tool arguments, stderr, message text, and provider payloads. Keep `LOG_RAW_API_PAYLOADS`, `LOG_RAW_SSE_EVENTS`, `LOG_API_ERROR_TRACEBACKS`, `LOG_RAW_MESSAGING_CONTENT`, `LOG_RAW_CLI_DIAGNOSTICS`, and `LOG_MESSAGING_ERROR_DETAILS` disabled outside debugging sessions.
- **Messaging:** set narrow allowed Discord channel IDs or one Telegram user ID, and set `ALLOWED_DIR` to the smallest workspace root needed.

## Current Limitations

- Upstream providers vary in context window, tool-call fidelity, thinking/reasoning output, rate limits, and model availability.
- The proxy validates provider prefixes, not whether a specific upstream model ID exists.
- There is per-provider rate limiting and concurrency control, but no automatic multi-provider fallback chain or load balancer yet.
- Claude Desktop support is not guaranteed because this repository does not implement or verify a stable Desktop custom-base-url path.
- Some defaults currently differ between code and `.env.example`; see [`implementation-notes.md`](implementation-notes.md) for details.

## Development

### Project Structure

```text
free-claude-code/
├── server.py              # ASGI entry point for source/development runs
├── api/                   # FastAPI routes, Admin UI, services, runtime, optimizations
├── core/                  # Shared Anthropic protocol helpers, SSE, trace/rate utilities
├── providers/             # Provider transports, registry, error mapping, model listing
├── messaging/             # Discord/Telegram adapters, sessions, rendering, voice
├── cli/                   # fcc-server, fcc-init, fcc-claude entry points
├── config/                # Settings, provider catalog, paths, logging
├── scripts/               # Shell and PowerShell installers
├── smoke/                 # Live/interactive smoke tests
└── tests/                 # Unit, contract, API, CLI, provider, messaging tests
```

### Run From Source

```bash
git clone https://github.com/Alishahryar1/free-claude-code.git
cd free-claude-code
uv run uvicorn server:app --host 0.0.0.0 --port 8082
```

### Check Sequence

Run checks in this order before pushing:

```bash
uv run ruff format
uv run ruff check
uv run ty check
uv run pytest
```

### Package Scripts

`pyproject.toml` installs:

- `fcc-server` — starts the supervised FastAPI proxy and opens Admin UI when enabled.
- `free-claude-code` — compatibility alias for `fcc-server`.
- `fcc-init` — creates or migrates `~/.fcc/.env`.
- `fcc-claude` — launches Claude Code with proxy environment variables.

## Development Roadmap

These are feasible next phases based on the current architecture.

### Phase 0 — Documentation Accuracy

- Keep this README aligned with code, not marketing assumptions.
- Track decisions and discrepancies in [`implementation-notes.md`](implementation-notes.md).

### Phase 1 — Safer Defaults

- Reconcile defaults across `Settings`, `.env.example`, installers, Admin UI, and docs.
- Consider changing the default host from `0.0.0.0` to `127.0.0.1`.
- Add explicit LAN opt-in documentation or flag.

### Phase 2 — Resilience And Observability

- Add provider health metrics and Admin UI stats.
- Add circuit breakers and configurable fallback chains.
- Add manual model discovery refresh and cache visibility.

### Phase 3 — Cross-Project Distribution

- Package the proxy as a simpler cross-platform sidecar.
- Add documented recipes for using it from other developer tools that accept Anthropic-compatible base URLs.
- Explore a companion VS Code extension workflow that can start/stop the sidecar automatically.

## Contributing

- `.env.example` is a reference template; prefer the Admin UI for normal configuration.
- Keep changes small and covered by tests.
- Do not add `# type: ignore` or `# ty: ignore`; fix type issues directly.
- Run the full check sequence before opening a pull request.
- Report bugs and feature requests in [Issues](https://github.com/Alishahryar1/free-claude-code/issues).

## License

MIT License. See [LICENSE](LICENSE) for details.