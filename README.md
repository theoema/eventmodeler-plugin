# EventModeler plugin

Event modeling for coding agents. Installing the plugin gives your agent:

- **The method as skills.** There are 19 `eventmodeling-*` skills, from brainstorming events to validating a finished model. Your harness lists them and loads the right one when a task calls for it.
- **The EventModeler MCP server** (`https://api.eventmodeler.com/mcp`). The agent reads and edits your models through it, live, alongside you. The first time you use it, the harness asks you to sign in to EventModeler.

If you already added the EventModeler MCP server by hand, remove that entry after you install the plugin. Otherwise your agent sees every tool twice.

## Install

| Harness | Install |
|---|---|
| Claude Code | `/plugin marketplace add theoema/eventmodeler-plugin`, then `/plugin install eventmodeler@eventmodeler` |
| Codex | `codex plugin marketplace add theoema/eventmodeler-plugin`, then `codex plugin add eventmodeler@eventmodeler` |
| GitHub Copilot CLI | `copilot plugin marketplace add theoema/eventmodeler-plugin`, then `copilot plugin install eventmodeler@eventmodeler` |
| VS Code (agent mode) | Run **Chat: Install Plugin from Source** and enter `https://github.com/theoema/eventmodeler-plugin` |
| Cursor | Clone this repository into `~/.cursor/plugins/local/eventmodeler` |
| Gemini CLI | `gemini extensions install https://github.com/theoema/eventmodeler-plugin --auto-update` |
| Factory Droid | `droid plugin marketplace add theoema/eventmodeler-plugin`, then `droid plugin install eventmodeler@eventmodeler --scope user` |
| Devin / Windsurf | `devin plugins install theoema/eventmodeler-plugin` |

Other harnesses that read Agent Skills (OpenCode, Amp, Kiro, Antigravity and others) can use the plugin too:

1. Copy `skills/` into the harness's skills directory (often `.agents/skills`).
2. Add `https://api.eventmodeler.com/mcp` as a remote MCP server.

## Updating

| Harness | How updates arrive |
|---|---|
| Claude Code | Follows this repository's latest commit. Turn on auto-update for the marketplace under `/plugin` → Marketplaces, or run `/plugin marketplace update`. |
| Codex | Run `codex plugin marketplace upgrade`. |
| Gemini CLI | Updates itself if you installed with `--auto-update`. |

## Using it

Ask for what you want modeled, for example "model the checkout of a web shop" or "add a refund slice to the Orders model". The agent loads `eventmodeling-core-rules` and the skill for the step, reads the model, builds it cell by cell and runs `check_model` before it calls the work done.

This repository is generated from the EventModeler monorepo. Don't edit it here.
