# mcp-agent-mail

mcp-agent-mail provides a coordinated mailbox and messaging layer for MCP-based coding agents. It runs as an MCP-compatible server and CLI, giving agents a shared “email-style” workspace for exchanging messages, reserving files, and coordinating work across projects.

Under the hood it is built on fastmcp, FastAPI, and Uvicorn, with SQLAlchemy-backed state, Pydantic models, and a lightweight HTTP layer for the web viewer and MCP transport. The server exposes tools such as `send_message` for rich, Markdown-based communication (including attachments, threads, and topics) and integrates with Git worktrees to enforce file reservations and guardrails across agents.

## Introduction / Overview

mcp-agent-mail is a Python CLI tool that provides a multi-agent mailbox and messaging layer for MCP-compatible coding agents. It sits between your MCP runtime and one or more agents to coordinate messages, manage shared context, and enforce guardrails on file access and workflow coordination.

The system maintains an inbox/archive of threads and attachments that multiple agents can read and write, with a web-based viewer for humans to inspect and share message bundles. Workflow macros are available to bootstrap sessions, summarise long-running threads, and orchestrate collaboration between agents without each agent having to implement its own mailbox logic.

## Table of Contents

- [Requirements / Prerequisites](#requirements--prerequisites)
- [Installation / Setup](#installation--setup)
- [Quick Start / Basic Usage](#quick-start--basic-usage)
- [Configuration](#configuration)
- [Advanced Usage / Commands / API](#advanced-usage--commands--api)
- [Features](#features)
- [Demo / Examples](#demo--examples)
- [License](#license)

## Requirements / Prerequisites

- Python 3.11+ (tested with 3.12; use a modern CPython environment compatible with type hints and `async` I/O).
- A working MCP-compatible runtime or agent framework to attach the mailbox server to (for example, an MCP host that can connect to an HTTP or process-based MCP server).
- Ability to install Python packages from PyPI in your environment.
- Core runtime libraries pulled in as dependencies:
  - `fastmcp` for MCP server integration
  - `uvicorn` as the ASGI HTTP server
  - `jinja2` for HTML templating
  - `sqlalchemy` for persistence and mailbox state
  - `pydantic` for configuration and data validation
- Optional: PostgreSQL if you plan to use the `postgres` extra (`asyncpg`-based) instead of purely local storage.
- Basic access to Git worktrees if you want to use the file reservation system with Git/worktree-aware guardrails.

## Installation / Setup

Install `mcp-agent-mail` into the Python environment where your MCP runtime or agent framework runs.

```bash
pip install mcp-agent-mail
```

This installs the MCP server, HTTP service, and supporting libraries as a standard Python package. Use Python 3.11+ (tested with 3.12) in a virtual environment or your existing MCP runtime environment.

## Quick Start / Basic Usage

The fastest way to get a mailbox-backed MCP server running is to start the CLI, then (optionally) embed it directly into your own Python process.

Command-line (recommended for first run):

```bash
# Start the default MCP + HTTP server
mcp-agent-mail
```

You can also invoke the same entry point via the module interface:

```bash
python -m mcp_agent_mail
```

Minimal Python integration, creating an MCP server you can attach to your own runtime:

```python
from mcp_agent_mail.app import build_mcp_server

# Build the MCP server instance (mailbox + tools + HTTP/MCP plumbing)
server = build_mcp_server()

# Attach `server` to your MCP runtime / transport here.
# For example, plug it into your existing stdin/stdout or HTTP bridge.
```

If you prefer to drive the CLI programmatically (e.g. from another Python script), call the Typer application directly:

```python
from mcp_agent_mail import cli

if __name__ == "__main__":
    # Equivalent to running `mcp-agent-mail` on the command line
    cli.app()
```

## Configuration

Configuration is primarily driven by a small set of files and environment variables. You can keep everything in-repo (for local dev) or wire it into containerized / multi-service setups.

- `agent_capabilities.json` / `agent_capabilities.yaml`  
  - Define what each MCP-compatible agent can do (tools, limits, and behavior hints).  
  - Typically checked into the repo root or a dedicated `config/` directory.  
  - Use JSON or YAML depending on your preference; the schema is the same conceptually—describe agents, their roles, and any constraints for mailbox workflows.

- `WORKTREES_ENABLED` (environment variable)  
  - Global toggle for worktree-aware behavior in the file reservation and project identity logic.  
  - Default is effectively disabled (`false`); set `WORKTREES_ENABLED=true` in your environment (e.g., `.env`, container env, or process-level export) to opt into Git worktree–friendly guardrails.

- `compose.yaml` / `docker-compose.yml`  
  - Orchestrate the MCP HTTP server, backing database, and any supporting services in a containerized environment.  
  - Usually live at the project root and define how the `mcp_agent_mail` service is built, exposed (ports), and wired to volumes, networks, and environment variables.  
  - Use these when you need a reproducible, multi-service deployment rather than a single local process.

In addition to these top-level knobs, more granular settings (HTTP, database, storage, CORS, LLM, tool filtering, notifications, and background maintenance intervals) are commonly provided via environment variables or a runtime settings layer. For production, prefer centralizing them in your deployment’s config mechanism (e.g., `.env` + compose, Kubernetes secrets/config maps, or your MCP runtime’s configuration system).

## Advanced Usage / Commands / API

This section assumes you already have the CLI and basic MCP server running. It focuses on embedding the server and CLI into larger systems, and on higher-level multi-agent patterns.

### Embedding the MCP server

You can attach the mailbox server to an existing MCP runtime instead of running it as a standalone process. This is useful when you already have a host application that manages MCP transports (e.g., stdio, WebSocket, HTTP).

```python
from mcp_agent_mail.app import build_mcp_server

# Create the MCP server instance configured from your environment/config files
server = build_mcp_server()

# Attach `server` to your MCP runtime / transport
# Example (pseudo-code, framework-specific):
#
# runtime = MyMcpRuntime(transport=StdioTransport())
# runtime.register_server(server)
# runtime.run()
```

Typical integration patterns:

- **Agent host process**: Start your own agent runtime, then register the mailbox MCP server as one of the tool servers available to all agents.
- **Multi-server hub**: Use a hub that aggregates multiple MCP servers; `build_mcp_server()` becomes one of the registered backends exposed under a `resource://mail/...` or `tool://mail/...` namespace (depending on your setup).
- **Custom lifecycle**: Wrap `build_mcp_server()` in your own startup/shutdown hooks so the mailbox is available only for specific workflows or tenants.

The MCP server will use the same configuration files and environment variables described elsewhere (e.g., capabilities, worktree settings, HTTP endpoints).

### Embedding the CLI app

If you already have a Python entrypoint or orchestrator process, you can embed the Typer-based CLI instead of invoking the `mcp-agent-mail` command externally.

Minimal embedding:

```python
from mcp_agent_mail import cli

if __name__ == "__main__":
    # Delegate to the full CLI as if `mcp-agent-mail` was called
    cli.app()
```

Common patterns:

- **Subcommand in a larger CLI**: Mount `cli.app` under a parent Typer or Click app so `my-tool mail ...` delegates to this project’s commands.
- **Programmatic invocation**: Use Typer’s `CliRunner` or `app` callbacks within tests or automation scripts to drive mailbox operations without shelling out.
- **Restricted surface**: Wrap specific `cli.app` commands in your own functions to expose only a subset of mailbox capabilities to end users.

### Multi-agent mailbox workflows

At its core, this project provides a **multi-agent mailbox** to coordinate MCP-compatible coding agents. Typical patterns:

- **Per-agent inboxes**: Each agent (e.g., “planner”, “implementer”, “reviewer”) has an inbox and archive. Agents read from their own mailbox, act, then write replies or artifacts back to another agent’s inbox.
- **Shared threads**: Multiple agents collaborate in a single thread that records messages, decisions, and code artifacts. You can use threads as the canonical history backing your agent orchestration.

Example conceptual flow:

1. A planner agent posts a task description and initial file reservations to a new thread.
2. An implementer agent consumes that thread, writes code changes, and updates the mailbox with status and diffs.
3. A reviewer agent summarises the thread and either approves or opens follow-up tasks in new threads.

### File reservation and worktree-aware guardrails

The mailbox server can coordinate **file reservations** and optional Git/worktree guardrails so agents avoid conflicting edits.

Typical usage patterns:

- **Per-thread reservations**: When starting work on a task, reserve a set of files or globs for that thread (e.g., `src/api/*.py`). Other agents can see that reservation and avoid making conflicting changes.
- **Worktree isolation**: With worktrees enabled, each task or thread can be associated with its own worktree or branch, keeping experimental changes isolated until they’re ready to merge.
- **Conflict handling**: When a second agent tries to reserve or modify a file already locked by another thread, the mailbox layer can surface that conflict so your orchestrator can queue work, re-route it, or ask for human intervention.

These patterns are especially useful in CI-style pipelines or autonomous coding systems where multiple agents modify the same repository in parallel.

### Workflow macros for orchestration

The project exposes **workflow macros** that bundle common multi-step operations (registering agents, preparing context, summarizing threads, updating inboxes) into a single call. These are typically exposed as MCP tools your agents can invoke.

Example macro usage (conceptual):

```python
# Bootstrap a new coding session for a given project and agent program
macro_start_session(
    human_key="/abs/path/backend",
    program="codex",
    model="gpt5",
    file_reservation_paths=["src/api/*.py"],
)
```

This kind of macro can:

- Register or refresh the agent in the mailbox.
- Create or attach to a thread for the session.
- Reserve relevant files and initialise worktree context.

Another example pattern is briefing an agent before joining an active discussion:

```python
# One-shot prepare/briefing step before an agent joins an existing thread
macro_prepare_thread(
    # typical arguments would include thread id, agent identity, and format options
)
```

Such macros generally:

- Ensure the agent is registered and up-to-date.
- Summarise the existing thread to a concise brief.
- Fetch the agent’s inbox context so it can respond coherently.

You can:

- **Call macros directly** from a coordinator process to standardise how sessions start and how new agents are onboarded to a thread.
- **Expose macros as first-class tools** within your MCP runtime so agents can self-serve: e.g., “prepare me for this thread” or “start a new coding session for this repo”.

By combining the embedded MCP server, file reservations, and workflow macros, you can build higher-level orchestration such as:

- Autonomous multi-agent code review lanes (planner → implementer → reviewer).
- Parallel feature branches coordinated via worktrees and reservations.
- Human-in-the-loop workflows where humans interact via the web inbox while agents operate via MCP tools against the same mailbox.

## Features

- Multi-agent mailbox for MCP-compatible coding agents, with per-project threads, search, and archiving.  
- Structured messaging APIs for agents, including thread summaries, key-point extraction, and action-item detection.  
- File reservation system that locks paths per agent/project, preventing concurrent edits and merge conflicts.  
- Git- and worktree-aware guardrails that respect multiple worktrees and repository layout when reserving files.  
- HTTP server exposing mailbox and reservation operations with rate limiting, authentication hooks, and detailed logging.  
- MCP server wrapper so the mailbox and reservations are directly callable from MCP runtimes and tools.  
- Web-based unified inbox UI for browsing projects, threads, and attachments, plus an archive viewer for historical work.  
- Shareable bundles to package message threads and related context for handoffs between agents or teams.  
- Workflow macros for bootstrapping new sessions, briefing agents via summaries, and coordinating multi-agent flows.

## Demo / Examples

The repository includes an `examples` area with richer, end-to-end scenarios that go beyond the minimal snippets shown earlier. You can use these as reference templates and adapt them to your own MCP runtime and agent stack:

- **MCP runtime integration** – Sample configurations showing how to:
  - Spin up the MCP HTTP server via `build_mcp_server()`
  - Attach the mailbox tools to an existing MCP-compatible coding agent
  - Wire in rate limiting, auth, and logging for a production-like setup

- **Mailbox workflows** – Example JSON-RPC tool calls and macros demonstrating:
  - Sending and replying to messages between agents (including rich content like inline images)
  - Using workflow macros such as `macro_start_session` and `macro_prepare_thread` to bootstrap sessions, brief helper agents, and fetch inbox context
  - Reserving files for edits and coordinating changes across agents in a Git/worktree-aware way

- **Compose / container setups** – Sample `compose.yaml` / `docker-compose.yml`-style configurations illustrating:
  - Running the MCP server and web inbox UI together
  - Exposing the unified inbox and archive viewer
  - Sharing configuration (e.g., `agent_capabilities.json`/`.yaml`) across services

- **Starter playbooks** – Small, scenario-focused “playbooks” that:
  - Show how to structure projects into mailboxes per repo or team
  - Demonstrate how agents can select the right verbs/tools for specific tasks (e.g., summarization, onboarding to an active thread, archival)

Browse the examples directory to copy a close match to your use case, then adjust project paths, models, and agent names to align with your environment.

## License

This project is licensed under the MIT License. See the [LICENSE](./LICENSE) file for full details.