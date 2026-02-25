# Mistral Vibe

Minimal CLI coding agent by Mistral. This Python-based tool provides an interactive terminal UI for chatting with a coding assistant that can review, edit, and manipulate your codebase directly from the command line, with support for advanced workflows like Agent Client Protocol (ACP) mode and MCP integrations. See the [project homepage](https://github.com/mistralai/mistral-vibe) for details and examples.

## Table of Contents

- [Installation](#installation)
- [Quick Start / Usage](#quick-start--usage)
- [Configuration](#configuration)
- [Features](#features)
- [Advanced Usage](#advanced-usage)
- [Contributing](#contributing)
- [Changelog](#changelog)
- [License](#license)

## Installation

Mistral Vibe is distributed as a Python package and requires Python 3.12 or later and a terminal environment.

Install it from PyPI with:

```bash
pip install mistral-vibe
```

## Quick Start / Usage

After installation, you can use `mistral-vibe` either from the terminal or directly from Python.

### CLI: run the coding assistant

Start an interactive TUI session in your current project directory:

```bash
vibe
```

- Use this to chat with the agent, review code, and apply edits in your workspace.
- By default, it operates on files under the current working directory.

To run the Agent Client Protocol (ACP) process (for tools that speak ACP):

```bash
vibe-acp
```

This starts a long‑running agent process that external clients can connect to.

### Programmatic usage (Python)

You can call the agent directly from Python code:

```python
from vibe.core.programmatic import run_agent

result = run_agent(prompt="Refactor this function", files=["main.py"])
print(result)
```

This runs a single-shot agent invocation with the given prompt and list of files, and returns the agent’s response as a string.

## Configuration

Vibe reads configuration from environment variables and an optional `vibe.toml` file in the current working directory.

- **Authentication (`.env` / `MISTRAL_API_KEY`)**
  - The default provider is Mistral, using the `MISTRAL_API_KEY` environment variable:
    ```bash
    export MISTRAL_API_KEY="your-mistral-api-key"
    ```
  - You can keep this in a local `.env` file (e.g. when using a shell/env loader). Vibe itself just relies on `MISTRAL_API_KEY` being present in the environment.
  - Other providers (like `llamacpp`) may not require an API key by default, but can be wired similarly via their `api_key_env_var`.

- **Agent and tool configuration (`vibe.toml`)**
  - If a `vibe.toml` file is present in the current working directory, it is loaded as the main configuration.
  - It is used for high-level agent settings (model/backend selection, system prompt) and tool setup (enabling/disabling tools, adding custom tools, MCP servers, etc.).
  - Minimal example:
    ```toml
    [agent]
    provider = "mistral"
    model = "mistral-large-latest"

    [project]
    # project-specific context and logging options live here

    [tools]
    # built-in tools can be configured, enabled, or disabled

    [tools.custom]
    tool_paths = ["./tools"]  # additional directories/files with custom tools

    [[mcp_servers]]
    name = "my-mcp"
    command = "my-mcp-server"
    ```
  - Configuration values can also be overridden via environment variables using the `VIBE_` prefix (e.g. `VIBE_PROVIDER`, `VIBE_MODEL`), following the internal settings schema.

## Features

- Minimal terminal-based coding assistant focused on code editing, refactoring, and review without leaving your shell workflow.  
- Interactive Textual/Rich TUI with chat-style interaction, keyboard shortcuts, and contextual tool output panes.  
- Built-in tools for editing files, searching and replacing across your project, running shell commands, and inspecting repository structure.  
- Pluggable model and backend configuration, including model metadata such as pricing and capabilities to help you choose the right runtime.  
- Agent Client Protocol (ACP) mode and MCP server integrations so you can embed the agent into existing editor/agent clients and multi-tool setups.

## Advanced Usage

### `vibe` vs `vibe-acp`

- `vibe`  
  - Full-screen Textual/Rich TUI.  
  - Best for interactive coding sessions, browsing files, and iterating on changes.  
  - Uses the built-in agent with tools and project context.

- `vibe-acp`  
  - Runs in Agent Client Protocol (ACP) mode.  
  - Intended to be driven by external ACP-aware clients (IDEs, editors, other CLIs).  
  - You typically don’t “chat” with it directly; instead you connect via an ACP client and let that orchestrate requests/results.

Use `vibe` for manual terminal work; use `vibe-acp` when integrating Mistral Vibe into a larger toolchain via ACP.

---

### Textual/Rich TUI interaction model

Inside the `vibe` TUI you generally:

- Type free-form prompts for coding tasks, refactors, or reviews.
- Use slash-commands for control actions:
  - `/help` – list available commands and tools.
  - `/clear` – clear conversation history.
  - `/compact` – summarize and compact the current conversation.
  - `/log` – show the path to the current interaction log file.
  - `/reload-config` – reload configuration from disk.
  - `/exit` – quit the session.
- Navigate output using standard terminal keybindings (scrollback, focus changes, etc., as provided by Textual).

The UI is designed so you can stay in the terminal while editing, inspecting diffs, and re-running prompts.

---

### Built-in tools: editing, search/replace, shell

The agent has several built-in tools that can be selectively enabled or disabled. Typical categories:

- **File editing**
  - Create and edit project files directly from the agent.
  - The agent can propose patches and apply them without you leaving the terminal.
- **Search / replace**
  - Search across your project for symbols, strings, or patterns.
  - Apply targeted replacements, or let the agent coordinate a refactor that touches multiple files.
- **Shell commands**
  - Run basic shell commands (e.g., tests, linters, formatters) so the agent can see results and iterate.
  - Useful for “run tests and fix failures” loops.

From the CLI you can control what the agent is allowed to use:

```bash
# Enable only a subset of tools (others disabled)
vibe --enabled-tools bash* --enabled-tools re:edit-.*

# Programmatic usage with strict tool set
python -c "from vibe.core.programmatic import run_agent; print(
    run_agent(prompt='Optimize this', files=['main.py'], enabled_tools=['edit-file'])
)"
```

`--enabled-tools` supports exact names, glob patterns (e.g. `bash*`), and regex with a `re:` prefix.

---

### Programmatic and model/backend configuration

For Python integration, use the programmatic API:

```python
from vibe.core.programmatic import run_agent

result = run_agent(
    prompt="Add type hints and docstrings",
    files=["app.py"],
    model="mistral-large-latest",  # any supported Mistral model
    max_turns=4,
    max_price=0.25,               # USD cost cap for this run
)
print(result)
```

Key options:

- `model`: choose which Mistral model to use.
- `max_turns`: cap the number of assistant responses for deterministic runs.
- `max_price`: hard dollar limit for the session; the run stops if this is exceeded.
- `enabled_tools`: restrict which tools the agent can call.

Pricing metadata is tracked internally; `max_price` uses that to prevent overruns without you having to compute token usage yourself.

---

### Custom tools, skills, and MCP servers

You can extend the agent with your own capabilities:

- **Custom tools**
  - Implement tools following Vibe’s `BaseToolConfig` conventions.
  - Point Vibe at your implementations via `tool_paths` (directories or files).  
    - Directories are shallow-searched for valid tool definitions.
    - Files are loaded directly if valid.

- **Skills**
  - Organize higher-level behaviors as “skills.”
  - Control discovery with:
    - `skill_paths`: extra directories to search for skills.
    - `enabled_skills`: glob/regex-based allowlist.
    - `disabled_skills`: explicit blocklist.
  - Patterns use shell globs (e.g. `search-*`) or regex with `re:`.

- **MCP server integrations**
  - Configure `mcp_servers` to point at compatible Model Context Protocol servers.
  - Once configured, those servers become additional tool backends the agent can call (e.g., to access external APIs, databases, or services).
  - In ACP mode (`vibe-acp`), these MCP-backed tools are also available to ACP clients that speak to Vibe.

These options can be set via the configuration system (see the Configuration section) or via programmatic configuration before invoking `run_agent`.

## Contributing

Contributions are welcome. Please see the `CONTRIBUTING` guide in the repository for the full workflow, coding standards, and review process.

Common ways to contribute include:

- Filing bug reports or feature requests via GitHub Issues: https://github.com/mistralai/mistral-vibe/issues  
- Submitting pull requests that improve the CLI, TUI, ACP integration, or documentation  
- Refining configuration handling (e.g., `vibe.toml`, `.env`, `MISTRAL_API_KEY`) and backend/model metadata

Before opening a pull request, ensure your changes are tested locally and aligned with the existing project structure and style.

## Changelog

See the full history of changes in the project’s [CHANGELOG](CHANGELOG.md). This file is updated with notable fixes, new features, and other release notes over time.

## License

This project is licensed under the Apache-2.0 License. See the [LICENSE](LICENSE) file for the full license text.