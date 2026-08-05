# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

**compose-for-agents** is a collection of independent, self-contained demo projects showing how to build
and run agentic AI applications with **Docker Compose**. Each top-level directory is a standalone demo that
pairs a different agent framework (A2A, ADK, Agno, CrewAI, LangGraph, Spring AI, langchaingo, Embabel,
Vercel AI SDK) with two Docker-native building blocks:

+ **Docker Model Runner** — runs open LLMs locally (or via Docker Offload on a remote GPU), declared as a
  top-level `models:` entry in `compose.yaml` and injected into a service via `endpoint_var`/`model_var`.
+ **Docker MCP Gateway** (`docker/mcp-gateway:latest`) — a service that securely exposes one or more MCP
  servers (DuckDuckGo, GitHub, Brave, MongoDB, etc.) to agents over SSE/streaming, with secrets injected
  from a mounted `.mcp.env` file rather than plaintext environment variables.

There is **no shared code, build system, or dependency graph** between demos — each has its own
`compose.yaml`, Dockerfile(s), language toolchain, and README. Treat each top-level directory as its own
project root.

## Demos

| Demo | Agent framework | Language | Default inference | MCP servers |
| ---- | ---- | ---- | ---- | ---- |
| `a2a` | [A2A](https://github.com/a2aproject/a2a-python) (Auditor/Critic/Reviser) | Python | OpenAI | duckduckgo |
| `adk` | [Google ADK](https://github.com/google/adk-python) | Python | gemma3 (local) | duckduckgo |
| `adk-cerebras` | ADK + [Cerebras](https://www.cerebras.ai/) | Python | local DMR + Cerebras remote | — |
| `adk-sock-shop` | ADK multi-agent sock-shop demo | Python | qwen3 (local) | brave, mongodb |
| `agno` | [Agno](https://github.com/agno-agi/agno) (agent + Next.js UI) | Python + TS | qwen3 (local) | github-official |
| `crew-ai` | [CrewAI](https://github.com/crewAIInc/crewAI) marketing strategy agent | Python | qwen3 (local) | duckduckgo |
| `langgraph` | [LangGraph](https://github.com/langchain-ai/langgraph) SQL agent | Python | qwen3 (local) | postgres |
| `langchaingo` | [langchaingo](https://github.com/tmc/langchaingo) | Go | gemma3 (local) | duckduckgo |
| `spring-ai` | [Spring AI](https://spring.io/projects/spring-ai) | Java (Maven) | local DMR | duckduckgo |
| `embabel` | [Embabel](https://github.com/embabel/embabel-agent) travel agent | — (README only; compose lives in the [tripper](https://github.com/embabel/tripper) upstream repo) | multiple | brave, google-maps, airbnb |
| `vercel` | Vercel AI SDK chat UI | — (README only; compose lives in the [scira-mcp-chat](https://github.com/slimslenderslacks/scira-mcp-chat) upstream repo) | qwen3 + llama3.2 (local) | brave, wikipedia-mcp |

`embabel` and `vercel` only contain a README in this repo — their actual `compose.yaml` and source live in
separate upstream GitHub repos that must be cloned before running.

## Running a demo

Every demo follows the same three-step pattern from its own directory:

```sh
cd <demo>
# create .mcp.env from mcp.env.example (only needed if the demo uses MCP secrets)
docker compose up --build
```

Prerequisites: Docker Desktop 4.43.0+ (or Docker Engine + Compose 2.38.1+), and a GPU for local model
inference — or [Docker Offload](https://www.docker.com/products/docker-offload/) if no GPU is available.

### Compose file variants (per demo, not all present in every demo)

+ `compose.yaml` — base file; the default inference backend varies per demo (see table above).
+ `compose.openai.yaml` — switch inference to OpenAI. Requires a `secret.openai-api-key` file (containing
  a bare `sk-...` key) at the demo root:
  ```sh
  docker compose -f compose.yaml -f compose.openai.yaml up
  ```
+ `compose.dmr.yaml` — explicit Docker Model Runner (local) config, used where the base file defaults to a
  remote provider (e.g. `a2a`).
+ `compose.offload.yaml` — override that pulls a larger model for use with Docker Offload's remote GPU:
  ```sh
  docker compose -f compose.dmr.yaml -f compose.offload.yaml up --build
  ```
+ `compose.gcloudrun.yaml` / `compose.gcloud.yaml` — Google Cloud Run deployment target (`adk`,
  `adk-sock-shop`).

### MCP secrets

MCP servers that need credentials read them from a `.mcp.env` file (copied/edited from that demo's
`.mcp.env.example`) mounted into the `mcp-gateway` service via a Compose `secrets:` entry named
`mcp_secret`:

```yaml
secrets:
  mcp_secret:
    file: ./.mcp.env
```

`.mcp.env` lines are `server.key=value` pairs (e.g. `github.personal_access_token=ghp_...`,
`brave.api_key=<fill-me>`). They can also be generated with the Docker MCP CLI, e.g.
`docker mcp secret set 'brave.api_key=...' && docker mcp secret export brave > .mcp.env`. Never commit a
real `.mcp.env` or `secret.openai-api-key` file — only the `.example` variants belong in git.

### Cleanup

```sh
docker compose down -v
```

## Architecture pattern common to every demo

```
Browser / CLI
     ↓
Agent service(s)  ──depends_on──▶  mcp-gateway (docker/mcp-gateway:latest)
     │                                    │
     │ models: {name: {endpoint_var, model_var}}   --servers=<mcp-server-list>
     ▼                                    ▼
Docker Model Runner (local LLM)      MCP server(s) (duckduckgo, github-official, brave, postgres, mongodb…)
```

+ Agent services declare which model they need via a service-level `models:` map
  (`endpoint_var: MODEL_RUNNER_URL`, `model_var: MODEL_RUNNER_MODEL`), and the actual model image/context
  size is declared once at the top-level `models:` key in `compose.yaml` (e.g.
  `models: {gemma3: {model: ai/gemma3:4B-Q4_0, context_size: 10000}}`).
+ The `mcp-gateway` service runs with `use_api_socket: true` and a `--servers=<comma-separated list>`
  command flag; it can also chain `--interceptor` commands to reshape MCP tool output (e.g. JSON→CSV) before
  it reaches the agent.
+ Multi-agent demos (`a2a`, `adk`, `adk-sock-shop`) have several agent services in the same compose file
  that talk to each other over HTTP and share one `mcp-gateway`.
+ `adk-sock-shop/CLAUDE.md` contains machine-readable instructions for **generating new `compose.yaml`
  model/gateway entries** in that specific demo — consult it (not this file) when adding models or MCP
  servers under `adk-sock-shop/`.

## Linting / CI

+ **Markdown**: `task lint:markdown` (or `task lint:markdown:fix`) runs
  `davidanson/markdownlint-cli2:v0.18.1` via Docker over the whole repo, using the rules in
  `.markdownlint.yaml` (line length 120, MD001/MD025 disabled, etc.). CI (`.github/workflows/markdownlint.yaml`)
  runs this on any push/PR touching `**/*.md`. Requires the [Task](https://taskfile.dev) CLI.
+ **Per-demo CI** (`.github/workflows/{a2a,adk,adk-cerebras,agno,crew-ai,langgraph}.yaml`, path-filtered to
  their own directory) each run a `check` job and a `build` job:
  - Python demos (`a2a`, `adk`, `adk-cerebras`, `agno/agent`, `langgraph`) use `uv`:
    ```sh
    uv run ruff format --check
    uv run ruff check
    uv run pyright   # skipped for adk-cerebras
    ```
  - `crew-ai` uses Poetry instead of uv:
    ```sh
    poetry install --with dev
    poetry run ruff format --check
    poetry run ruff check
    poetry run pyright
    ```
  - The `build` job runs `docker compose build` from the demo's own directory (via the
    `.github/actions/setup-compose` composite action).
  - `adk-sock-shop`, `embabel`, `langchaingo`, `spring-ai`, and `vercel` currently have **no** CI workflow.
+ Python demos share the same lint config shape — a per-demo `.ruff.toml` selecting
  `["E4", "E7", "E9", "F", "I", "RUF022"]` with `force-sort-within-sections` isort.

## Licensing

Dual-licensed under Apache-2.0 or MIT (`LICENSE.APACHE-2`, `LICENSE.MIT`) at the repo root. Individual
demos may carry their own `LICENSE` file reflecting third-party requirements — respect those when editing
files under that demo's directory.
