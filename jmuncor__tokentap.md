# tokentap — CLI token tracker for LLM tools with a live terminal dashboard

Token tracker for LLM CLI tools with live terminal dashboard. tokentap is a Python-based CLI utility that runs an HTTP proxy to observe and summarize token usage across multiple LLM providers while you work in the terminal. It’s designed for developers who use command‑line LLM tools and want real‑time visibility into context usage, request history, and prompt content without changing their existing workflow.

By streaming intercepted traffic into a live terminal dashboard, tokentap helps you keep an eye on context limits, understand how different tools consume tokens, and maintain a local history of your interactions. For more context and updates, see the [tokentap website](https://tokentap.ai).

## Table of Contents

- [Requirements / Prerequisites](#requirements--prerequisites)
- [Installation / Setup](#installation--setup)
- [Quick Start / Basic Usage](#quick-start--basic-usage)
- [Configuration](#configuration)
- [Features](#features)
- [Documentation / Resources](#documentation--resources)
- [License](#license)

## Requirements / Prerequisites

- Python runtime (any modern Python 3 version) is required to run this CLI tool.  
- Core dependencies are `aiohttp`, `rich`, `tiktoken`, and `click`; these are installed automatically as Python package dependencies.  
- No external database or services are required beyond network access to your chosen LLM providers.

## Installation / Setup

Install tokentap from PyPI using pip:

```bash
pip install tokentap
```

This installs the `tokentap` CLI entry point into your active Python environment.

## Quick Start / Basic Usage

```bash
# 1. Start the TokenTap proxy and dashboard
tokentap start
```

This launches the HTTP proxy (by default on `http://127.0.0.1:8080`) and opens the live terminal dashboard.

```bash
# 2. Run your LLM CLI tool through TokenTap
# Example: wrap a CLI command and specify the provider
tokentap run --provider anthropic anthropic-cli chat --model claude-3-opus
```

As your CLI sends requests, TokenTap intercepts them via the proxy and updates the dashboard in real time with token usage, context utilization, and a rolling request log.

## Configuration

Tokentap works out of the box with sensible defaults, but several paths and runtime settings can be customized if needed:

- `TOKENTAP_DIR` (default: `~/.tokentap`)  
  Root data directory where tokentap stores its local state. Other paths below are relative to this directory by default.

- `HISTORY_FILE` (default: `~/.tokentap/history.json`)  
  Persistent log of intercepted requests and token usage. This file backs the history shown in the dashboard and can be archived or rotated if it grows large.

- `PROMPTS_DIR` (default: `~/.tokentap/prompts`)  
  Directory for saving and reusing prompts. Useful for organizing common prompt templates outside of your CLI workflows.

- `IPC_FILENAME` (default: `tokentap_ipc.jsonl`)  
  JSONL file used as a lightweight IPC channel between the HTTP interceptor and the live dashboard process. Normally created inside the tokentap data directory.

- `DEFAULT_PROXY_PORT` (default: `8080`)  
  Port used by the local HTTP proxy when no explicit port is provided via CLI options. Change this if `8080` conflicts with another service on your machine.

These values are defined as configuration points in the codebase and can be overridden by adjusting how you launch tokentap or by modifying your local configuration/environment according to your setup preferences.

## Features

- Live terminal dashboard for real-time tracking of LLM token usage across your CLI tools.  
- HTTP proxy layer that intercepts and inspects requests to multiple LLM providers.  
- Context token limit tracking with a visual “fuel gauge” showing how close you are to the limit.  
- Request log view with last prompt preview, token counts, and provider metadata.  
- Persistent history and prompt storage in a local data directory for later inspection and analysis.

## Documentation / Resources

For the latest documentation, examples, and release notes, see the project website at https://tokentap.ai. Additional updates, issue tracking, and source-level details are also available in the GitHub repository’s issues and code comments.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file in this repository for the full text.