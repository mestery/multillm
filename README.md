# MultiLLM

A multi-user LLM stack built from open-source parts:

- **Backend** — one or more [MTPLX](https://github.com/youssofal/MTPLX) and/or
  [LM Studio](https://lmstudio.ai) servers, running natively on the host
  (Apple Silicon).
- **Router** — [LLMRouter](https://github.com/llm-router/llm-router), an
  OpenAI-compatible proxy that aggregates the backends, balances load between
  them, and gates access with API keys. Runs in Docker.
- **UI** — [Open WebUI](https://open-webui.com/), a multi-user chat interface
  with accounts, permissions, and per-user settings. Runs in Docker.

Both containerized services are brought up with `docker compose`.

## Architecture

```
                        host (Apple Silicon)
+-------------------------+-------------------------+
|  MTPLX   :8000   |  LM Studio :1234   |  (more…)  |   native
+-------------------------+-------------------------+
        ^ OpenAI-compatible /v1
        |
+-------+---------+  docker
| LLMRouter :9000 |  (container port 8000)
+-------+---------+
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
  - **MTPLX**: `mtplx serve --port 8000` (OpenAI-compatible, base URL `http://<host>:8000/v1`)
  - **LM Studio**: Developer tab → *Start Server* (default port `1234`, base URL `http://<host>:1234/v1`)
- Port 9000 and 3000 free on the host (the defaults used below).

## Quickstart

```bash
cp .env.example .env        # review ports and the API key
docker compose up -d

# Open WebUI:  http://localhost:3000   (create the first admin account)
# LLMRouter:   http://localhost:9000   (manage API keys and model groups)
```

Open WebUI is pre-wired to talk to LLMRouter (`http://llmrouter:8000/v1`)
using the API key from `llm-router/config.yaml`. After the first admin
sign-up, the models served by whichever backends are up appear in the UI.

## Configuration

### Ports and key (`.env`)

| Variable               | Default | Meaning                                                    |
| ---------------------- | ------- | ---------------------------------------------------------- |
| `LLMROUTER_HOST_PORT`  | `9000`  | Host port for LLMRouter (container always uses 8000)       |
| `OPENWEBUI_HOST_PORT`  | `3000`  | Host port for Open WebUI (container always uses 8080)      |
| `LLMROUTER_API_KEY`    | —       | Key Open WebUI uses against LLMRouter; must match `llm-router/config.yaml` |
| `WEBUI_AUTH`           | `true`  | `false` disables user accounts (single-user mode)          |

The LLMRouter host port defaults to **9000** because MTPLX serves on host
port 8000 by default, which is also LLMRouter's in-container port.

### Backends (`llm-router/config.yaml`)

Each backend is an *endpoint* inside a *model group*. The initial config
ships with one group per supported backend:

- `mtplx`    → `http://host.docker.internal:8000/v1`
- `lmstudio` → `http://host.docker.internal:1234/v1`

Add more servers by appending an endpoint (or a whole group) following the
same shape. `model_names: []` means "pass through whatever models the
backend reports via `/v1/models`". `host.docker.internal` resolves to the
Docker host on both macOS and Linux (wired via `host-gateway` in the
compose file).

You can also manage keys and groups at runtime in the LLMRouter web UI at
`http://localhost:9000`; changes are written back to `config.yaml` in the
mounted `./llm-router/config` directory.

## Multi-user setup

- **Accounts**: Open WebUI provides user registration, login, and
  admin-controlled roles. With `WEBUI_AUTH=true` (default), every user gets
  their own account and chat history.
- **API keys**: LLMRouter's API keys are the gate to the backends. The
  shipped key in `llm-router/config.yaml` is the default for all users.
  For per-user billing or limits, create one LLMRouter key per user
  (LLMRouter UI → *API Keys*) and set it on that user in Open WebUI
  (Admin → Users → user → *OpenAI API Key*).

> Rotate the initial `sk-…` key in `llm-router/config.yaml` (and `.env`)
> before exposing the stack beyond a trusted LAN.

## Notes

- Backends run **natively**, not in Docker: LM Studio is a GUI app and MTPLX
  is optimized for Apple Silicon, so the containers reach them via
  `host.docker.internal`.
- State: Open WebUI data lives in the `open-webui-data` named volume;
  LLMRouter config is bind-mounted from `./llm-router/config`.
- To stop: `docker compose down` (add `-v` only if you want to erase
  Open WebUI data).

## License

Distributed under the [Apache License, Version 2.0](LICENSE).
