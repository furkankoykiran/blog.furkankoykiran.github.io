---
title: "freqtrade-mcp: Ask Claude to Check Your Trades"
description: "A TypeScript MCP server that connects Claude and other AI agents to a running Freqtrade bot — check balances, manage pairs, and execute trades in natural language."
date: 2026-04-04 13:00:00 +0300
categories: [MCP, Crypto Trading, Open Source]
tags: [mcp, freqtrade, typescript, claude, crypto]
image:
  path: /assets/img/posts/2026-04-04-freqtrade-mcp/banner.png
  alt: "Freqtrade MCP Server"
---

I run a Freqtrade bot. Like most people who run trading bots, I tell myself I'm "not constantly checking it" — and then check it constantly.

The ritual: open browser, log into FreqUI, click through tabs, squint at numbers, close browser, reopen browser 10 minutes later. It adds up.

At some point I thought: I already have Claude open. Why am I switching contexts to a UI when the bot has a perfectly good REST API?

So I built [@furkankoykiran/freqtrade-mcp](https://github.com/furkankoykiran/freqtrade-mcp) — a TypeScript MCP server with 15 tools that lets you ask Claude what your bot is doing. Works with Claude Desktop, Cursor, Cline, anything that speaks [Model Context Protocol](https://modelcontextprotocol.io). You ask in plain English. The bot answers.

Yes, I built an AI wrapper around my trading bot so I can talk to it through a different AI. I'm at peace with this.

![Freqtrade FreqUI](/assets/img/posts/2026-04-04-freqtrade-mcp/banner.png)
*The Freqtrade trading interface — now accessible via natural language through Claude.*

---

## What You Can Do

### Portfolio & Performance

The questions I actually ask my bot, at various hours of the day:

> *"How much profit have I made total? Please be honest."*

> *"What's my current balance — how much is deployed vs sitting in USDT doing nothing?"*

> *"Which pairs are performing well? Show me the top 5 so I can feel better about my decisions."*

> *"Am I net positive this month or should I not look?"*

`get_profit_stats`, `get_balance`, and `get_performance` handle all of these. Claude gets back structured JSON from the Freqtrade API and gives you a readable summary instead of making you parse numbers yourself at 11pm.

---

### Open Trades & History

> *"What trades are open right now? Any in the red that I should know about?"*

> *"Show me trade #42 — when did it open, what's the P/L, and should I be worried?"*

> *"Give me the last 10 closed trades."*

The bot can have dozens of open positions at once. Scanning a raw JSON blob yourself at midnight is not the move. Claude summarizes it. You move on.

---

### Market Data on Demand

> *"Fetch the last 50 BTC/USDT candles on the 4h timeframe."*

> *"What does the ETH/USDT 1h chart look like right now?"*

The `get_market_data` tool pulls live OHLCV data directly from the exchange through Freqtrade. Ask Claude what the 4h chart looks like right now. It'll tell you something. Whether you should act on it is your problem.

![Freqtrade Backtesting](/assets/img/posts/2026-04-04-freqtrade-mcp/freqtrade-backtesting.png)
*Freqtrade's strategy analysis interface — the same data the MCP server can fetch on demand.*

---

### Pair List Management

> *"Add LUNA/USDT to the blacklist."*

> *"What's currently on my whitelist? Remove BNB/BTC from the blacklist."*

Normally this means editing a JSON config file, reloading, and hoping you didn't break anything. Or navigating FreqUI through three menu levels to find the right panel. With the MCP server it's a sentence. Done.

---

### Manual Trade Execution

> *"Buy 100 USDT of ETH right now."*

> *"Open a short on BTC/USDT with a 50 USDT stake."*

> *"Exit trade #7 at market price."*

`execute_trade` and `force_exit_trade` bypass strategy signals entirely — useful when you want to act on something the bot wouldn't otherwise touch.

> **Note:** This executes immediately against the live exchange. "Use with intention" is doing a lot of heavy lifting in that sentence. No confirmation dialog. No undo. The exchange doesn't care that you were just testing the API.

---

### Bot Lifecycle

> *"Stop the bot — I need to edit the config before it does something."*

> *"Reload the config without restarting, the bot is mid-cycle."*

> *"What version is the bot on? Did it run recently or has it been sitting there silently failing?"*

Stopping a live trading bot always feels slightly dramatic. One call handles it. `get_bot_info` combines version and health data so you know whether the bot is actually running or just pretending to be.

---

## Setup

Enable the REST API in your Freqtrade `config.json` (it's off by default, for good reason), then add the server to your Claude Desktop config:

```json
{
  "mcpServers": {
    "freqtrade": {
      "command": "npx",
      "args": ["-y", "@furkankoykiran/freqtrade-mcp"],
      "env": {
        "FREQTRADE_API_URL": "http://127.0.0.1:8080",
        "FREQTRADE_USERNAME": "Freqtrader",
        "FREQTRADE_PASSWORD": "YourPassword"
      }
    }
  }
}
```

Restart Claude Desktop. All 15 tools show up immediately, no build step required.

---

## Available Tools

| Category | Tools |
|---|---|
| **Account** | `get_profit_stats`, `get_balance`, `get_performance` |
| **Trades** | `get_open_trades`, `get_trade`, `get_trade_history` |
| **Market Data** | `get_market_data` |
| **Pair Lists** | `get_whitelist`, `get_blacklist`, `add_to_blacklist`, `remove_from_blacklist` |
| **Execution** | `execute_trade`, `force_exit_trade` |
| **Lifecycle** | `start_bot`, `stop_bot`, `reload_config`, `get_bot_info`, `get_locks`, `delete_lock` |

---

## Conclusion

The honest version: I was tired of opening FreqUI to check one number. The REST API was already there, just buried behind a browser tab I had to switch to.

Now I ask Claude. It tells me. I close the laptop slightly less anxious than before.

The project is open source and MIT licensed. The README has the full setup guide, including how to configure the Freqtrade REST API if you haven't done that yet.

**[github.com/furkankoykiran/freqtrade-mcp](https://github.com/furkankoykiran/freqtrade-mcp)**

---

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Other Blog Posts

- [Adding Browser Automation to CLI-Anything: First MCP Backend Pattern](/posts/cli-anything-browser-automation-contribution-en/)
- [awesome-trending-repos: Modern Web Interface for GitHub Trending](/posts/awesome-trending-repos-web-en/)
- [My Open Source Contributions to MCP Ecosystem and AI Platforms](/posts/mcp-ai-contributions/)
