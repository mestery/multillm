# MultiLLM

A multi-user LLM stack built from open-source parts:

- **Backend** — one or more [MTPLX](https://github.com/youssofal/MTPLX),
  [LM Studio](https://lmstudio.ai), and/or [oMLX](https://github.com/jundot/omlx)
  servers, running natively on the host (Apple Silicon).
- **Router** — [LiteLLM](https://github.com/BerriAI/litellm), an
  OpenAI-compatible proxy that aggregates the backends, balances load between
  them, and gates access with API keys. Runs in Docker.
- **UI** — [Open WebUI](https://open-webui.com/), a multi-user chat interface
  with accounts, permissions, and per-user settings. Runs in Docker.

The router and UI are brought up with `docker compose`; the LiteLLM config is
rendered from your `.env` on every start via `scripts/render-config.sh`.

## Architecture

```
                          host (Apple Silicon)
+-------------------------+---------------------------+
|  MTPLX :8000  |  LM Studio :1234  |  oMLX :8000  …   |  native
+-------------------------+---------------------------+
        ^ OpenAI-compatible /v1
        |
+-------+---------+  docker
| LiteLLM :4000   |  (container port 4000)
+-----------------+
        ^ OpenAI-compatible /v1
        |
+-------+---------+  docker
| Open WebUI :3000|  (container port 8080)
+-----------------+
        ^
        |
   browsers / users
```

## Prerequisites

- Docker + Docker Compose (v2 plugin) on the host.
- At least one backend running natively on the host:
  - **MTPLX**: `mtplx serve --port 8000` (base URL `http://<host>:8000/v1`)
  - **LM Studio**: Developer tab → *Start Server* (default port `1234`, base URL `http://<host>:1234/v1`)
  - **oMLX**: `omlx serve --model-dir ~/models` (default port `8000`, no API key unless started with `--api-key`)
- Port 9001 and 3000 free on the host (the defaults used below).

## Quickstart

```bash
cp .env.example .env        # review ports, router key, and backends
docker compose up -d

# Open WebUI:  http://localhost:3000   (create the first admin account)
# LiteLLM:     http://localhost:9001   (health check at /health)
```

On start, a one-shot `llmrouter-init` container renders
`llm-router/config.yaml` from the `BACKEND_*` variables in `.env`, then the
router comes up. Open WebUI is pre-wired to talk to the router
(`http://llmrouter:4000/v1`) using `ROUTER_API_KEY` from the same file.
After the first admin sign-up, the models served by whichever backends are
up appear in the UI.

## Configuration

All configuration lives in `.env` (copy from `.env.example`; it is
gitignored — change `.env`, not `.env.example`).

### Ports and router key

| Variable              | Default | Meaning                                                    |
| --------------------- | ------- | ---------------------------------------------------------- |
| `LLMROUTER_HOST_PORT` | `9001`  | Host port for LiteLLM (container always uses 4000)         |
| `OPENWEBUI_HOST_PORT` | `3000`  | Host port for Open WebUI (container always uses 8080)      |
| `ROUTER_API_KEY`      | —       | Key Open WebUI uses against LiteLLM (required)             |
| `WEBUI_AUTH`          | `true`  | `false` disables user accounts (single-user mode)          |

The LiteLLM host port defaults to **9001** because MTPLX and oMLX both
serve on host port 8000 by default, which is also LiteLLM's in-container
port.

### Backends

Declare one or more backends as `BACKEND_<N>_*` (N = 1..8, contiguous
starting at 1). Each becomes a model group in the router:

| Variable                      | Required | Default                                  |
| ----------------------------- | -------- | ---------------------------------------- |
| `BACKEND_<N>_TYPE`            | yes      | — (`mtplx` \| `lmstudio` \| `omlx`)      |
| `BACKEND_<N>_HOST`            | no       | `host.docker.internal`                   |
| `BACKEND_<N>_PORT`            | no       | `8000` (mtplx/omlx), `1234` (lmstudio)   |
| `BACKEND_<N>_NAME`            | no       | `<type>-<port>`                          |
| `BACKEND_<N>_API_KEY`         | no       | placeholder (set a real key if the server is started with auth) |

Examples:

```bash
# A mixed fleet, each server on its own port:
BACKEND_1_TYPE=mtplx                     # port 8000 (default)
BACKEND_2_TYPE=lmstudio                  # port 1234 (default)
BACKEND_3_TYPE=omlx
BACKEND_3_PORT=8001                      # oMLX also defaults to 8000

# Two MTPLX instances for load balancing:
BACKEND_1_TYPE=mtplx
BACKEND_2_TYPE=mtplx
BACKEND_2_PORT=8100
BACKEND_2_NAME=mtplx-fast
```

Rendering is done by `scripts/render-config.sh` (plain POSIX sh); you can
run it on the host for a quick check:

```bash
set -a; . ./.env; set +a
sh scripts/render-config.sh
```

`llm-router/config.yaml` is generated and gitignored. The LiteLLM web UI
can still edit it at runtime, but `.env` is the source of truth — the file
is re-rendered on every `docker compose up`, overwriting runtime edits.

## Multi-user setup

- **Accounts**: Open WebUI provides user registration, login, and
  admin-controlled roles. With `WEBUI_AUTH=true` (default), every user gets
  their own account and chat history.
- **API keys**: LiteLLM's API keys are the gate to the backends.
  `ROUTER_API_KEY` is the default for all users. For per-user billing or
  limits, create one LiteLLM key per user (LiteLLM UI → *API Keys*) and
  set it on that user in Open WebUI (Admin → Users → user → *OpenAI API Key*).

> **Key hygiene.** Rotate `ROUTER_API_KEY` in `.env` before exposing the stack
> beyond a trusted LAN. Never commit real keys: `.env` is gitignored and only
> `.env.example` (placeholders) is tracked. If a real key ever leaks into git
> history, treat it as compromised — rotate it in `.env`, then purge the old
> value from every commit with [`git filter-repo`](https://github.com/newren/git-filter-repo)
> (e.g. `git filter-repo --replace-text <(echo 'sk-LEAKED_KEY==>REDACTED')`)
> and force-push.

## Notes

- Backends run **natively**, not in Docker: LM Studio is a GUI app and
  MTPLX/oMLX are optimized for Apple Silicon, so the containers reach them
  via `host.docker.internal`.
- State: Open WebUI data lives in the `open-webui-data` named volume;
  LiteLLM config is bind-mounted from `./llm-router/config.yaml` (generated).
- To stop: `docker compose down` (add `-v` only if you want to erase
  Open WebUI data).

## License

Distributed under the [Apache License, Version 2.0](LICENSE).
