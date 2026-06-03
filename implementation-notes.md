# README Implementation Notes

This file records decisions made while rewriting `README.md`, especially where the previous README, `.env.example`, and current code did not fully agree.

## Scope

- Rewrote `README.md` from verified project code rather than treating the old README as authoritative.
- Did not change Python code, installers, tests, `.env.example`, `summary.html`, or assets.
- Used Markdown for this log because it is easier to diff and review than HTML.

## Sources Inspected

- `README.md` and `summary.html`
- `.env.example`
- `pyproject.toml`
- `api/routes.py`, `api/dependencies.py`, `api/admin_routes.py`, `api/app.py`, `api/optimization_handlers.py`
- `cli/entrypoints.py`
- `config/settings.py`, `config/provider_catalog.py`, `config/provider_ids.py`, `config/paths.py`
- `assets/how-it-works.mmd`
- Broad subagent surveys of `providers/`, `core/anthropic/`, `messaging/`, `scripts/`, `tests/`, and `smoke/`

## Major Decisions

### README should describe code reality, not provider marketing

The old README included many concrete provider model slugs. The code validates only provider prefixes, not specific upstream model IDs. Because model catalogs change frequently and several old examples could not be verified in code, the rewritten README avoids listing speculative model IDs and instructs users to consult provider catalogs/Admin UI checks.

Tradeoff: the README is less copy-paste friendly for first-time users, but it is less likely to become inaccurate or direct users to nonexistent models.

### Provider count is 17

`config/provider_catalog.py` contains 17 canonical provider entries:

1. `nvidia_nim`
2. `open_router`
3. `gemini`
4. `deepseek`
5. `mistral`
6. `mistral_codestral`
7. `opencode`
8. `opencode_go`
9. `wafer`
10. `kimi`
11. `cerebras`
12. `groq`
13. `fireworks`
14. `zai`
15. `lmstudio`
16. `llamacpp`
17. `ollama`

The rewritten README treats `opencode` and `opencode_go` as separate provider IDs because the catalog exposes them separately even though they share the same API key.

### Claude Desktop support is intentionally not promised

The code and previous README show clear paths for Claude Code CLI, VS Code extension, JetBrains ACP, and the built-in messaging bot flow. I did not find a verified, stable Claude Desktop custom base URL path in this repository. The README now says Desktop support is not guaranteed unless a specific wrapper/build supports equivalent environment variables.

Tradeoff: this is conservative, but avoids implying support for a client this codebase does not verify.

### "Tier Max / Max 20x" is framed as an analogy, not a claim

Anthropic Max-style tiers are subscription/quota products. This project can multiplex providers and route model tiers, but it does not grant Anthropic quota or reproduce Anthropic's service guarantees. The README now says it can approximate high-throughput coding workflows but depends on upstream providers.

### Security warnings were promoted

The code default host is `0.0.0.0` in `config/settings.py`, while Admin UI itself is loopback-restricted in `api/admin_routes.py`. Since the API can be reachable on a LAN when the host binds all interfaces, the README now explicitly warns users to configure `ANTHROPIC_AUTH_TOKEN`, firewall rules, and web-tool settings before non-local use.

## Discrepancies Found

### Default model mismatch

- `config/settings.py`: `model = "nvidia_nim/z-ai/glm4.7"`
- `.env.example`: `MODEL="nvidia_nim/nvidia/nemotron-3-super-120b-a12b"`
- Old README: described the `.env.example` default.

README handling: avoid claiming a single canonical default model. It explains model reference format and provider prefixes instead.

Recommended future fix: choose one canonical default and sync `Settings`, `.env.example`, Admin UI defaults, tests, and docs.

### Web server tools default mismatch

- `config/settings.py`: `ENABLE_WEB_SERVER_TOOLS` defaults to `False`.
- `.env.example`: `ENABLE_WEB_SERVER_TOOLS=true`.
- Old README said local `web_search` / `web_fetch` handling is on by default.

README handling: document the setting and its SSRF/security implications without promising it is always on.

Recommended future fix: decide whether the secure default is off or whether generated env should explicitly enable it; document the rationale.

### Voice defaults mismatch

- `config/settings.py`: `voice_note_enabled=True`, `whisper_device="cpu"`, `whisper_model="base"`.
- `.env.example`: `VOICE_NOTE_ENABLED=false`, `WHISPER_DEVICE="nvidia_nim"`, `WHISPER_MODEL="openai/whisper-large-v3"`.

README handling: describe both backends and required settings, without claiming a single default path.

Recommended future fix: align code defaults and env template based on safest expected installation experience.

### Provider rate-limit defaults mismatch

- `config/settings.py`: `PROVIDER_RATE_LIMIT=40`, `PROVIDER_RATE_WINDOW=60`, `PROVIDER_MAX_CONCURRENCY=5`.
- `.env.example`: `PROVIDER_RATE_LIMIT=1`, `PROVIDER_RATE_WINDOW=3`, `PROVIDER_MAX_CONCURRENCY=5`.

README handling: document the settings generically, not the default values.

Recommended future fix: define rate-limit defaults by provider or expose clear Admin UI labels.

### Host default and local-only wording

- Code binds to `0.0.0.0` by default.
- Admin UI is loopback-only.
- Old README emphasized local Admin UI but did not clearly warn that proxy API can bind broadly.

README handling: explicitly distinguish server bind behavior from Admin UI loopback checks.

Recommended future fix: consider defaulting `host` to `127.0.0.1` and adding explicit LAN opt-in.

## Items Removed From README

- Star history section: useful for repository marketing, not necessary for implementation/user guidance.
- Long lists of unverified model examples: high maintenance, could mislead users.
- Claims that imply Claude Desktop support without a verified path.
- Over-specific provider model recommendations where code does not enforce or verify availability.

## Items Added To README

- Explicit explanation of what the project is and is not.
- API route table.
- Settings load order and managed config path.
- Canonical 17-provider table from `config/provider_catalog.py`.
- Security section covering bind address, auth, web tools, logs, and messaging.
- Current limitations section.
- Development roadmap split into phases.

## Residual Risks

- README may still become stale if provider catalog entries or upstream base URLs change.
- README references VS Code extension environment variable behavior based on existing README/code launcher assumptions; the external extension may change its config key in the future.
- Since this task intentionally did not modify code, default mismatches remain present.
- Install script behavior was summarized based on survey results; README does not document every installer edge case.

## Proposed Future Phases

### Phase 1 — Safer Defaults

- Align `config/settings.py`, `.env.example`, Admin UI defaults, tests, and README.
- Consider `127.0.0.1` as the default bind host.
- Add explicit LAN opt-in and stronger startup warnings when auth is disabled on non-loopback bind.

### Phase 2 — Better Provider Resilience

- Add health-aware fallback chains.
- Add circuit breakers for repeated provider failures.
- Expose provider latency/error/rate-limit status in the Admin UI.

### Phase 3 — Cross-Project Sidecar

- Package proxy as a cross-platform sidecar for other developer tools.
- Document a stable recipe for any client that supports `ANTHROPIC_BASE_URL` and `ANTHROPIC_AUTH_TOKEN`.
- Explore a VS Code companion extension that can install, configure, start, and stop the sidecar.

### Phase 4 — Smoke Coverage Expansion

- Add live smoke targets for each provider category.
- Add messaging bot end-to-end tests with fake platform adapters.
- Add security smoke tests for loopback admin restrictions and private-network web fetch blocking.