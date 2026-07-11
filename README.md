# Astra CLI

**A terminal AI assistant that understands natural language and drives your developer
tools** — git, docker, kubernetes, filesystem, code, shell, web research, and MCP plugins
— through a multi-agent LangGraph pipeline with local memory, RAG, and multi-provider LLM
fallback.

```
$ astra "what's the git status here, and run the linter"
```

[![PyPI](https://img.shields.io/pypi/v/astra-cli-agent.svg)](https://pypi.org/project/astra-cli-agent/)
[![Python](https://img.shields.io/pypi/pyversions/astra-cli-agent.svg)](https://pypi.org/project/astra-cli-agent/)

---

## What it does

Every request goes through a real multi-agent pipeline — a router+planner picks the right
agent(s) and breaks the request into steps, worker agents execute them against real tools,
a reviewer agent checks the output and can loop back for a revision — not a single prompt
wrapping a shell call.

| Domain | Agent | Capabilities |
|---|---|---|
| Git | `GitAgent` | status, diff, log, blame, branch, commit (auto-generates messages), push, merge, checkout, stash |
| Filesystem & code | `FileAgent` | read/write/move/delete files, lint & format (`ruff`), run tests (`pytest`), symbol search |
| Shell | `FileAgent` | runs arbitrary commands behind a hard denylist + confirmation gate |
| Docker | `DockerAgent` | ps, images, build, run, logs, exec, inspect, rm/rmi, prune |
| Kubernetes | `KubernetesAgent` | get/describe pods, deployments, services, nodes; apply, scale, rollout restart, delete, exec |
| Web research | `ResearchAgent` | GitHub repo/issue search, Stack Overflow search, general web search, fetch-with-citation |
| Debugging | `DebugAgent` | root-causes errors and stack traces, proposes concrete fixes |
| Documentation | `DocumentationAgent` | writes docstrings, README sections, usage guides |
| MCP | `MCPAgent` | connects to any configured MCP server (stdio, HTTP, or SSE) and discovers/calls its tools |

**Example prompts by domain** — every one of these routes automatically, no flags needed
beyond `--yes` for anything risky:

| Domain | Example prompt |
|---|---|
| Git | `astra "what's the git status here, and diff the uncommitted changes"` |
| Git | `astra "commit my staged changes with a good message" --yes` |
| Filesystem & code | `astra "create a file called notes.md with a todo list"` |
| Filesystem & code | `astra "run the linter and the test suite on this project"` |
| Shell | `astra "run df -h and tell me how much disk space is free" --yes` |
| Docker | `astra "list all running docker containers"` |
| Docker | `astra "build a docker image from the Dockerfile in this directory" --yes` |
| Kubernetes | `astra "list the pods in the default namespace"` |
| Kubernetes | `astra "scale the api deployment to 3 replicas" --yes` |
| Web research | `astra "search stack overflow for how to fix a detached HEAD in git"` |
| Web research | `astra "find github repos for a python rate limiter library"` |
| Debugging | `astra "here's a stack trace, find the root cause and propose a fix: <paste>"` |
| Documentation | `astra "write a README section documenting the config file format"` |
| MCP | `astra "list the tools available from my configured MCP servers"` |

Also included:

- **Safety by default** — risky operations (`git push`, file delete, shell, container/exec
  commands) require interactive confirmation unless you pass `--yes`; a shell denylist
  blocks obviously destructive commands (`rm -rf /`, fork bombs, `curl | sh`, etc.)
  regardless of confirmation.
- **Local memory + RAG** — `astra ingest .` indexes a codebase (SQLite + FAISS); relevant
  chunks are pulled into the planner's context automatically on later requests.
- **Multi-provider LLM fallback** — Groq → Anthropic → OpenAI → local Ollama, tries each in
  order and falls through on failure.
- **`astra chat`** — an interactive REPL with real token-level streaming, tab-completion,
  and persistent history.
- **Cost/token tracking** — `astra memory costs` shows a real per-provider/per-model
  breakdown and daily spend trend, no external service required.
- **First-run onboarding** — `astra setup` walks you through picking a provider and tests
  your key with a real request before saving it.

## Install

```bash
pip install astra-cli-agent
```

or with [uv](https://docs.astral.sh/uv/) / [pipx](https://pipx.pypa.io/) (recommended —
installs it in an isolated environment while still putting `astra` on your PATH):

```bash
uv tool install astra-cli-agent
# or
pipx install astra-cli-agent
```

This installs the `astra` command — the PyPI distribution name (`astra-cli-agent`) is
different from the command you actually run, since `astra-cli` was already taken by an
unrelated project.

Requires Python 3.12+.

## Quickstart

Astra needs at least one LLM provider before it can answer anything. Run the setup wizard
once:

```bash
astra setup
```

Pick a provider (Groq, Anthropic, OpenAI, or a local Ollama server) and paste a key — it's
tested with one real, cheap request before being saved to `~/.astra/.env` (never logged,
never committed). Don't have a key yet? [Groq's free tier](https://console.groq.com/keys)
is the fastest way to try Astra.

If you skip this step, the first command that actually needs an LLM (`ask`, a bare prompt,
or `chat`) will offer to run the wizard for you automatically.

Then just talk to it — **from inside whatever project directory you want it to act on**,
since every tool call is scoped to your current working directory:

```bash
astra hello                                   # sanity check: install, config, provider status
astra ask "what's 12 * 8"                     # one-shot LLM call, no agent pipeline
astra "what's the git status here"            # routes to GitAgent
astra "create a file called notes.md with a todo list"
astra "run the linter on this project"
astra ingest .                                # index this codebase for retrieval
astra "which file implements the fallback router"   # answer grounded in the ingested code
astra chat                                    # interactive REPL, streaming + history
astra memory costs                            # real cost/token breakdown for this session
```

Risky operations prompt for confirmation by default:

```bash
astra "delete the file scratch.txt"           # asks first
astra "delete the file scratch.txt" --yes     # skips the prompt
```

> Astra performs real actions when you approve them — it's not a dry-run tool. Try it in a
> throwaway directory first if you want to get a feel for it before pointing it at
> something you care about.

## Configuration

Astra reads settings in order (each layer overrides the previous):

1. packaged defaults
2. `~/.astra/config.yaml` (user overrides)
3. environment variables, e.g. `ASTRA_APP__LOG_LEVEL=DEBUG`

Provider API keys and other secrets live in `~/.astra/.env` (written by `astra setup`) —
global, not tied to any one project. Conversation history and cost tracking live in
`~/.astra/astra.db`.

## MCP servers

Add entries under `mcp.servers` in `~/.astra/config.yaml` to give Astra access to any MCP
server's tools/resources/prompts — same shape as Claude Desktop's `mcpServers` config for
stdio transport, plus `http`/`sse` for already-running remote servers:

```yaml
mcp:
  servers:
    - name: my-server
      command: npx
      args: ["-y", "@some/mcp-server"]
```

`astra mcp tools` lists everything discovered from your configured servers.

## All commands

Beyond natural-language prompts, every subcommand below is a direct, non-LLM entrypoint:

```bash
astra hello                       # install/config/provider sanity check
astra version                     # print the installed version
astra setup                       # interactive provider onboarding wizard
astra ask "<prompt>"              # one-shot LLM call, bypasses the agent pipeline
astra chat                        # interactive REPL with streaming + history
astra ingest [path]               # index a codebase for retrieval (defaults to .)

astra memory history [-n N]       # recent conversation history
astra memory clear                # wipe conversation history
astra memory costs                # per-provider/per-model cost & token breakdown
astra memory projects             # list ingested (RAG-indexed) projects
astra memory search "<query>"     # test retrieval directly against an ingested project
astra memory prefs set <k> <v>    # store a preference
astra memory prefs get <k>        # read a preference
astra memory prefs list           # list all preferences

astra mcp servers                 # list configured MCP servers
astra mcp tools                   # list tools discovered from configured MCP servers
```

Any bare prompt (`astra "..."`) or `astra run "<prompt>"` triggers the full
router→planner→execute→reviewer→memory pipeline described above.

## License

MIT
