# openbridge

Package `openbridge` v1.1.0. Telegram → OpenCode API relay bot.

## Entrypoints

- `openbridge` CLI: `src/openbridge/app.py` → `openbridge.app:main`
- Bridge module: `src/openbridge.opencode_bridge:main` (also reachable as `openbridge-opencode-bridge`)
- `python -m openbridge` → `src/openbridge/__main__.py` → `app:main`

## Architecture

6 modules layered under `src/openbridge/`:

| Module | Role |
|---|---|
| `opencode_bridge.py` | Composition root: dispatch, orchestration, stats, Telegram handlers |
| `opencode_api_client.py` | Stateless HTTP client for OpenCode API (sessions, polling, adaptive backoff) |
| `llm_service.py` | Optional input enhancement + output decoration via LLM |
| `bridge_presentation.py` | Telegram MarkdownV2 escaping, chunking, redaction, rendering |
| `workflow_management.py` | Workflow drafting, validation, persistence, `/workflow` command |
| `workflows.py` | Workflow engine: scheduler, cron, http_fetch, RSS normalization, state |
| `app.py` | CLI (argparse, 9+ commands), systemd lifecycle, setup wizard |

`bridge_presentation.py` and `workflow_management.py` were extracted from `opencode_bridge.py` in Phase 2 — keep boundaries clean.

## Commands

```bash
# Canonical test
python -m pytest -q

# Legacy unittest (still works)
.venv/bin/python -m unittest discover -s tests -p 'test_*.py'

# Full preflight: mypy → config drift → test → build
bash scripts/preflight.sh        # validate only
bash scripts/preflight.sh --install  # install deps first

# Type check (only src/openbridge/workflows.py!)
python -m mypy --config-file mypy.ini src/openbridge/workflows.py

# Package build
python -m build --sdist --wheel

# Nuitka binary build
./scripts/build_nuitka.sh                  # no implicit downloads
./scripts/build_nuitka.sh --allow-downloads # opt-in to downloads
```

## Testing quirks

- Tests use `sys.path.append` hack at top of each file (`sys.path.append(os.path.abspath(...))`) then import `from src.openbridge...`
- Mix of `unittest.TestCase` and pytest-style (both runners work)
- `test_backoff.py` and `test_narrow_exceptions.py` are pytest-style (use `pytest.raises`)
- Tests mock `urlopen` directly, no docker/services needed
- `test_opencode_bridge.py` creates `BridgeConfig` with inline mock values — never reads real `.env`

## Config & env

- **Runtime config**: `~/.config/openbridge/bridge.env` (shell `KEY=value` format, mode 0600)
- **Two files, least privilege**: `bridge.env` has Telegram + LLM secrets; `opencode.env` (for `opencode.service`) has only server auth — never leaks Telegram/LLM tokens to the OpenCode process
- **CLI `openbridge setup`** generates both files interactively
- **Allowlist**: `TELEGRAM_ALLOWED_CHAT_IDS` (comma-separated numeric IDs). Empty set + `TELEGRAM_ALLOW_ALL_CHATS=0` = all denied. `allow_all_chats` only works when explicitly `1`.
- `.env` at repo root is gitignored (real secrets present for local dev)
- Legacy `TELEWATCH_*` env vars still accepted as aliases for `OPENBRIDGE_*` (see `opencode_bridge.py:76-77`)

## Workflows

- JSON file at `~/.config/openbridge/workflows.json` (mode 0600)
- State file at `~/.config/openbridge/workflows-state.json` (mode 0600)
- Schedule formats: `daily@HH:MM`, `every:<seconds>`, `cron:<expression>`
- Step types: `opencode_prompt`, `telegram_send`, `http_fetch`, `transform_python`
- Max 10 steps per workflow (enforced in `_validate_workflow_safety`)
- `http_fetch` URLs blocked from private networks (`_is_safe_fetch_url` checks DNS + IP ranges)
- Overflow: prompts > `OPENBRIDGE_WORKFLOW_PROMPT_MAX_CHARS` (default 12000) are rejected or truncated based on `OPENBRIDGE_WORKFLOW_PROMPT_OVERFLOW_MODE`

## Security

- Token-like strings redacted from logs (`_redact_sensitive_text` in `bridge_presentation.py`)
- `/health` and `/stats` respect chat allowlist
- Workflow fetch blocked from `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, `::1/128`, `fc00::/7`, `fe80::/10`
- `subprocess.run` only in CLI management commands (systemd), never in prompt execution path

## Runtime

- One in-memory OpenCode session per Telegram chat; lost on bridge restart
- Per-chat bounded task queue (`OPENBRIDGE_CHAT_QUEUE_MAX_PENDING`, default 5)
- Overflow modes: `reject` (default) or `drop_oldest` (keeps latest prompt)
- No integration test dependencies (pinned `.venv`, no Docker)
