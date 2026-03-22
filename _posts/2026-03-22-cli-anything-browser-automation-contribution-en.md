---
title: "Adding Browser Automation to CLI-Anything: First MCP Backend Pattern"
description: "Contributed browser automation support to CLI-Anything via DOMShell MCP server integration. Implemented the project's first MCP backend pattern."
date: 2026-03-22 21:00:00 +0300
categories: [Open Source, Development]
tags: [mcp, python, cli-anything, open-source, browser-automation]
image:
  path: /assets/img/posts/2026-03-22-browser-automation-cli/mcp-architecture-diagram.png
  alt: "MCP Architecture"
---

Browser automation is genuinely interesting. AI agents navigating websites, filling forms - it's not just nice to have anymore. I recently contributed browser automation support to CLI-Anything via DOMShell's MCP server. Here's how it went.

![MCP Architecture](/assets/img/posts/2026-03-22-browser-automation-cli/mcp-architecture-diagram.png)
*MCP architecture - the bridge between AI agents and external services.*

## How It Started

On March 16, [@apireno](https://github.com/apireno) opened [#90](https://github.com/HKUDS/CLI-Anything/issues/90) with a compelling idea: **"CLI wrappers make software agent-native"** - and this should work for browsers too.

They mentioned [DOMShell](https://github.com/apireno/DOMShell), which maps Chrome's Accessibility Tree to a virtual filesystem. Agents control the browser with `ls`, `cd`, `grep`, `click` - shell commands, not DOM queries. The benchmark was solid: 50% fewer API calls than screenshot-based browsing in Claude tests.

![Accessibility Tree](/assets/img/posts/2026-03-22-browser-automation-cli/accessibility-tree-view.png)
*Accessibility Tree in Chrome DevTools - built for screen readers, works great for agents.*

### What is the Accessibility Tree?

The Accessibility Tree is what the browser derives from DOM for screen readers. It's simpler, semantic. Elements are classified by role - button, link, textbox. More stable than DOM, more understandable for agents.

DOMShell's insight: filesystem primitives beat DOM queries. `ls` and `grep` instead of `querySelector` and `getElementById`. Agents explore faster, call fewer APIs.

On March 21, based on [@omerarslan0](https://github.com/omerarslan0)'s proposal, I created [#118](https://github.com/HKUDS/CLI-Anything/pull/118). This became CLI-Anything's **first MCP server backend pattern**.

```bash
# How it looks
cli-anything-browser page open https://example.com
cli-anything-browser fs ls /
cli-anything-browser fs cd /main
cli-anything-browser fs grep "Login"
cli-anything-browser act click /main/button[0]
```

## Architecture

Most CLI-Anything harnesses call backend APIs directly. This one was different: **MCP Server integration**.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   CLI Commands   │────▶│  browser_cli.py │────▶│   MCP Backend   │
│  (Click groups)  │     │  (CLI entry)    │     │ (domshell_      │
└─────────────────┘     └─────────────────┘     │  backend.py)    │
                                                 └────────┬────────┘
                                                          │
                    ┌─────────────────────────────────────┼────────────┐
                    │                                     │            │
                    ▼                                     ▼            ▼
            ┌───────────────┐                 ┌────────────┐    ┌──────────┐
            │ Spawn npx     │                 │  DOMShell  │    │  Chrome  │
            │ subprocess    │◀──stdio─────────▶│  MCP Server│◀───│ + Ext    │
            └───────────────┘                 └────────────┘    └──────────┘
```

### MCP Backend Pattern

No CLI-Anything harness had used an MCP server before. The Python SDK implementation:

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(
    command="npx",
    args=["@apireno/domshell"]
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        result = await session.call_tool("domshell_ls", {"path": "/"})
```

MCP server is **stateless**. Each command spawns a new subprocess. State (URL, working directory, history) lives on the CLI side.

### Commands

Four main groups: **page** (open, reload, back, forward, info), **fs** (ls, cd, cat, grep, pwd), **act** (click, type), and **session** (status, daemon-start, daemon-stop).

### Daemon Mode

Spawning npx for every command costs 1-3 seconds. **Daemon mode** keeps a persistent connection. The difference is noticeable.

```bash
cli-anything-browser session daemon-start
# All commands use the same connection now
cli-anything-browser fs ls /
cli-anything-browser session daemon-stop
```

## Testing

I went with a multi-layered approach. Unit tests (31 total) mocked MCP responses, tested path resolution and state management. No Chrome required.

```bash
pytest cli_anything/browser/tests/test_core.py -v
# 31/31 PASSED
```

E2E tests (10 total) needed Chrome + DOMShell extension, tested against real pages. Tests skip when DOMShell isn't installed, so CI/CD doesn't break.

```bash
DOMSHELL_E2E=1 pytest cli_anything/browser/tests/test_full_e2e.py -v
```

## Review Process

After submitting, [@omerarslan0](https://github.com/omerarslan0) did a thorough review. I got 9 feedback items. The critical ones:

- **Daemon mode context manager leak** - Fixed with `_daemon_client_context` global
- **go_back()/go_forward()** - Switched from local history to native MCP tools
- **REPL quoted arguments** - Fixed with `shlex.split()`, quoted arguments parse correctly now
- **Security** - `act type` no longer echoes typed text, passwords don't leak into terminal scrollback

Each fix was a separate commit, documented in review comments.

![Browser CLI Interface](/assets/img/posts/2026-03-22-browser-automation-cli/browser-cli-interface.png)
*CLI interface - controlling the browser with filesystem commands.*

## Merge

The PR merged on March 22. 3,095 additions, 21 files changed, 15 commits. This became CLI-Anything's reference implementation for MCP backend patterns - a template for future integrations.

The browser harness went into CLI-Hub:

```json
{
  "name": "browser",
  "display_name": "Browser",
  "version": "1.0.0",
  "description": "Browser automation via DOMShell MCP server",
  "requires": "Node.js, npx, Chrome + DOMShell extension",
  "homepage": "https://github.com/apireno/DOMShell",
  "install_cmd": "pip install git+https://github.com/HKUDS/CLI-Anything.git#subdirectory=browser/agent-harness",
  "entry_point": "cli-anything-browser",
  "category": "web",
  "contributor": "furkankoykiran"
}
```

## What I Learned

**MCP Python SDK**: `stdio` transport with subprocess, async-to-sync wrapper, tool calling patterns.

**Accessibility Tree vs DOM**: More stable, less fragile, ideal for agents.

**Open Source**: Issue to implementation, managing review feedback, documentation, test coverage.

**Daemon Mode**: Context manager lifecycle, state persistence, event loop limitations.

## Future

I left out of V1: screenshot capture, wait-for-element, form fill helper, headless mode, multi-browser (Firefox, Safari), concurrent MCP operations.

[@apireno](https://github.com/apireno) suggested `eval` and `js` escape hatches for when elements don't have clean AX representations - good additions for a follow-up.

This contribution did more than add browser automation to CLI-Anything. It established a pattern for MCP server integrations. The "MCP Backend Pattern" section I added to `cli-anything-plugin/HARNESS.md` will guide other developers.

Implementing a new pattern in open source is interesting work. The feedback from [@omerarslan0](https://github.com/omerarslan0), [@yuh-yang](https://github.com/yuh-yang), and [@apireno](https://github.com/apireno) during review made this better.

---

**References:**
- [Issue #90](https://github.com/HKUDS/CLI-Anything/issues/90)
- [PR #118](https://github.com/HKUDS/CLI-Anything/pull/118)
- [DOMShell](https://github.com/apireno/DOMShell)
- [CLI-Anything](https://github.com/HKUDS/CLI-Anything)
