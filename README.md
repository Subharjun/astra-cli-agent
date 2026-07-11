# Astra CLI

**A terminal AI assistant that understands natural language and drives your developer
tools** — git, docker, kubernetes, filesystem, code, shell, web research, and MCP plugins
— through a multi-agent pipeline with local memory, retrieval-augmented context, and
multi-provider LLM fallback.

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

| Domain | Capabilities |
|---|---|
| Git | status, diff, log, blame, branch, commit (auto-generated messages), push, merge, checkout, stash |
| Filesystem & code | read/write/move/delete files, lint & format, run tests, symbol search |
| Shell | runs commands behind a hard denylist + confirmation gate |
| Docker | ps, images, build, run, logs, exec, inspect, rm/rmi, prune |
| Kubernetes | get/describe pods/deployments/services/nodes; apply, scale, rollout restart, delete, exec |
| Web research | GitHub repo/issue search, Stack Overflow search, general web search, fetch-with-citation |
| Debugging | root-causes errors and stack traces, proposes concrete fixes |
| Documentation | writes docstrings, README sections, usage guides |
| MCP | connects to any configured MCP server (stdio, HTTP, or SSE) and calls its tools |

Also included:

- **Safety by default** — risky operations (`git push`, file delete, shell, container/exec
  commands) require interactive confirmation unless you pass `--yes`; a shell denylist
  blocks obviously destructive commands regardless of confirmation.
- **Local memory + retrieval** — `astra ingest .` indexes a codebase; relevant context is
  pulled into requests automatically afterward.
- **Multi-provider LLM fallback** — tries each configured provider in order, falls through
  on failure.
- **`astra chat`** — an interactive REPL with real token-level streaming, tab-completion,
  and persistent history.
- **Cost/token tracking** — `astra memory costs` shows a real per-provider/per-model
  spend breakdown.
- **First-run onboarding** — `astra setup` walks you through picking a provider and tests
  your key with a real request before saving it.

## Install

```bash
pip install astra-cli-agent
```

or with [uv](https://docs.astral.sh/uv/) / [pipx](https://pipx.pypa.io/) (recommended —
isolated environment, still puts `astra` on your PATH):

```bash
uv tool install astra-cli-agent
# or
pipx install astra-cli-agent
```

This installs the `astra` command — the PyPI distribution name (`astra-cli-agent`) is
different from the command you actually run.

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
or `chat`) offers to run the wizard for you automatically.

Then just talk to it — **from inside whatever project directory you want it to act on**,
since every tool call is scoped to your current working directory:

```bash
astra hello                                   # sanity check: install, config, provider status
astra ask "what's 12 * 8"                     # one-shot LLM call, no agent pipeline
astra "what's the git status here"            # routes to the git tools
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

## License

MIT
