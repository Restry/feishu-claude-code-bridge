# lark-channel-bridge

A lightweight bot that bridges Feishu / Lark messenger with your local Claude Code CLI. Run one command, scan a QR code to bind a Lark app, and talk to Claude from chat — read screenshots, edit code, anything you'd do at the terminal.

[中文 README](./README.zh.md)

关于能实现的效果，详情可以阅读[飞书文档](https://larkcommunity.feishu.cn/docx/OaRIdFIRFoLM3xxTmKwcetHqn5e)

## What it does

- Forwards Feishu / Lark messages (DM directly, or `@bot` in a group) to your local `claude` CLI, running in a working directory you control.
- **Streaming card**: Claude's text and tool calls update on a single Lark card in real time — no waiting for the final reply.
- **Per-chat sessions**: each chat keeps its own Claude session, so conversations resume where they left off.
- **Preempt + batch**: a new message interrupts the running run; rapid-fire messages get coalesced into one request.
- **Multiple workspaces**: `/ws` switches between named project directories, with sessions tracked per workspace.
- **Images and files**: send them to the bot directly — Claude reads the locally downloaded paths.
- **Interactive cards**: `/help`, `/ws list`, `/status` return cards with buttons you can click.

## Prerequisites

- Node.js **>= 20**
- `claude` CLI installed and authenticated. Two paths:
  - **OAuth**: run `claude` once interactively to log in.
  - **API key** (recommended for headless / server deployments): set `ANTHROPIC_API_KEY` (and optionally `ANTHROPIC_BASE_URL` for proxies / gateways) in the environment that spawns the bridge. See [MCP server mode](#mcp-server-mode) below for how to pass these through when an MCP client launches the bridge as a subprocess.
- A Lark / Feishu **PersonalAgent** app — only needed for the Feishu bridge mode (the QR-code wizard on first launch can create one for you). **Not needed if you only use the MCP server.**

## Install

```bash
npm i -g lark-channel-bridge
# or
pnpm add -g lark-channel-bridge
```

## First run

```bash
lark-channel-bridge run
```

The first run detects there's no app configured and **opens a QR-code wizard**:

1. A QR code renders in your terminal.
2. Scan it with the Feishu / Lark app.
3. Pick or create a PersonalAgent app.
4. Credentials are written to `~/.lark-channel/config.json`.

## Commands

### Host CLI

**Process-level** (run the bridge directly in your shell):

```
lark-channel-bridge run [-c <config>]     Run the bot in the foreground
lark-channel-bridge ps                    List all running bridge processes on this machine
lark-channel-bridge kill <id|#>           Kill a bridge process (SIGTERM, SIGKILL after 2s)
lark-channel-bridge --help                List all commands
```

**Service-level** (run the bridge as a background OS-managed daemon):

> ⚠️ **Install globally before using service-level commands**. The daemon's launchd plist / systemd unit / Windows task hard-codes the path to the bridge CLI; if you invoke via `npx lark-channel-bridge start`, that path lives in npm's temp cache (`~/.npm/_npx/<hash>/...`) and will be garbage-collected — your daemon stops working as soon as the cache is cleaned. Use `npm install -g lark-channel-bridge` first, then run `lark-channel-bridge start`. `bridge run` is fine via npx (one-shot process).

```
lark-channel-bridge start                 Install (if needed) and start the daemon
lark-channel-bridge stop                  Stop the daemon and disable autostart
lark-channel-bridge restart               Restart the daemon in place
lark-channel-bridge status                Show daemon status (pid, log paths, last exit)
lark-channel-bridge unregister            Remove the service definition and stop
```

The daemon auto-restarts on crash and on user login. Platform mapping:
- **macOS** → `launchd` user agent at `~/Library/LaunchAgents/ai.lark-channel-bridge.bot.plist`
- **Linux** → `systemd` user unit at `~/.config/systemd/user/lark-channel-bridge.bot.service`. For the daemon to survive logout, run `loginctl enable-linger $USER` once.
- **Windows** → Task Scheduler task `LarkChannelBridge.Bot`, triggered ONLOGON. Launcher script at `~/.lark-channel/daemon-launcher.cmd`.

Daemon logs go to `~/.lark-channel/logs/daemon-stdout.log` and `daemon-stderr.log` alongside the bridge's per-day structured logs.

> When the same app is started multiple times, Lark's open platform routes events to one of the live WebSocket connections at random. `run` detects existing processes for the same app and (in a TTY) prompts: `[c]ontinue / [k]ill old / [a]bort`. In non-TTY mode it warns and continues.

### Slash commands inside Feishu / Lark

| Command | Effect |
|---|---|
| `/new`, `/reset` | Clear the current chat's session |
| `/cd <path>` | Switch working directory (resets session) |
| `/ws list` | List named workspaces (card + buttons) |
| `/ws save <name>` | Save current cwd as a named workspace |
| `/ws use <name>` | Switch to a named workspace |
| `/ws remove <name>` | Delete a named workspace |
| `/status` | Current cwd / session / agent (card + buttons) |
| `/config` | Adjust preferences (reply style, tool-call display, ...) |
| `/stop` | Stop the run in progress (also the `⏹` button on the card) |
| `/timeout [N\|off\|default]` | Idle-watchdog (minutes) for the current session. `/config` sets the global default. See FAQ below. |
| `/ps` | List all `start` processes on this host, marking the one replying |
| `/exit <id\|#>` | Stop a `start` process (your own → graceful; another's → SIGTERM) |
| `/reconnect` | Force a WebSocket reconnect (use when the bot stops responding after a network blip) |
| `/doctor [description]` | Feed recent logs and your description back to Claude for self-diagnosis |
| `/help` | Help card |
| Any other `/xxx` | Forwarded verbatim to Claude |

**Reply policy**: in a DM, the bot replies to anything. In a **group (including topic groups), the bot only replies when `@`-mentioned** (default since 0.1.22); unmentioned messages are ignored. `@all` is never answered. Cloud-doc comments must mention the bot. To restore the older "always answer in groups" behaviour: `/config` → "Require @bot in groups" → No.

## MCP server mode

Besides the Feishu bridge, the same package exposes Claude Code as an **MCP stdio server** so that any MCP-aware client (Hermes, Claude Desktop, etc.) can call Claude Code asynchronously without blocking its own chat loop.

```bash
lark-channel-bridge mcp \
  --cwd /path/to/default/workspace \
  --cwd-root /path/to/default/workspace \
  --cwd-root /tmp
```

- `--cwd` — default working directory for spawned Claude tasks. Caller can override per call.
- `--cwd-root` — repeatable whitelist. If set, task `cwd` must resolve under one of these roots. Recommended for any deployment that exposes the server to untrusted callers.

### Tools exposed

| Tool | Purpose |
|---|---|
| `claude_run(prompt, cwd?, session_id?, model?)` | Start a task; returns immediately with `task_id`. |
| `claude_status(task_id)` | Snapshot: status, text-so-far, current tool, counters. |
| `claude_wait(task_id, from_seq?, timeout_ms?)` | Short-block (≤ 120 s) for new events since `from_seq`. |
| `claude_cancel(task_id)` | SIGTERM (then SIGKILL) a running task. |
| `claude_list()` | List all tasks (running + terminal). |
| `claude_forget(task_id)` | Drop a finished task from memory. |

Long-running tasks (1–2 hours) don't sit in a single synchronous tool call. The client pattern is: `claude_run` → keep chatting → `claude_wait` / `claude_status` when you want progress → next `claude_wait` with the highest seen `seq` to stream incrementally.

### Constraints & boundaries (verified)

Task state lives **in process memory**, and the MCP server is stdio — its lifetime is bound to a single MCP client connection. That implies a few boundaries you must know:

- **Not persistent / not cross-process**: when the server process exits, every task record (everything `claude_list` / `claude_status` could see) is gone. A second client connection = a second server process = an empty task table. Don't expect "the task I started yesterday is still listable today", and don't expect two clients to share one task view.
- **What it is**: a **session-scoped task scheduler**, not a durable task queue. The intended pattern — `claude_run` to get a `task_id` → poll `claude_wait` / `claude_status` within the same connection until terminal — is fully reliable.
- **Resume is explicit**: continuation requires passing `session_id` to `claude_run` (under the hood `claude --resume <id>`); it does not auto-continue from `cwd`. Multiple tasks started in the same `cwd` are isolated and never cross-talk. Passing a non-existent `session_id` fails that task outright (`No conversation found`) — it does not silently fall back to a fresh session.
- **Event buffer cap**: each task retains at most the latest 5000 events; older ones are dropped (`seq` stays monotonic). Always pass the highest `seq` you've seen for incremental pulls.

> Persistence (listable across restarts, shared task list across clients) requires adding a layer yourself: spill the task table to disk / SQLite and restore it on server startup. The current version does not do this.

### Hermes config example

Edit `~/.hermes/config.yaml`, add a top-level `mcp_servers:` block:

```yaml
mcp_servers:
  cc:
    command: node
    args:
      - /path/to/feishu-claude-code-bridge/bin/lark-channel-bridge.mjs
      - mcp
      - --cwd
      - /home/me/Projects
      - --cwd-root
      - /home/me/Projects
      - --cwd-root
      - /tmp
    env:
      # Pass auth through — MCP clients spawn the bridge as a bare subprocess,
      # so .zshrc / .bashrc env is NOT inherited.
      ANTHROPIC_API_KEY: "sk-..."
      # Optional, for proxies / gateways:
      ANTHROPIC_BASE_URL: "https://api.example.com"
    timeout: 180         # max blocking time per MCP tool call (claude_wait caps internally at 120 s)
    connect_timeout: 30
```

Then in a Hermes chat, run `/reload-mcp` to hot-load without restarting. Verify with `hermes mcp test cc`.

> ⚠️ **Auth gotcha**: MCP clients spawn the bridge directly (`node ...`), bypassing your interactive shell. If `claude` is authenticated via the OAuth flow and you rely on shell env vars to find a config, those env vars are **not inherited**. Either set `ANTHROPIC_API_KEY` explicitly in the `env:` block (recommended) or wrap the `command` in `zsh -ic`.

> ⚠️ **Security**: tasks run with `--permission-mode bypassPermissions` (no confirmation prompts). For multi-tenant or untrusted callers, always use `--cwd-root` and treat the host machine as fully delegated to whoever can call the MCP server.

## Data directories

| Path | Content |
|---|---|
| `~/.lark-channel/config.json` | App credentials (App ID / Secret), mode 600 |
| `~/.lark-channel/sessions.json` | Claude session id + cwd per chat / topic (+ optional `/timeout` override) |
| `~/.lark-channel/workspaces.json` | Named-workspace map |
| `~/.lark-channel/processes.json` | Process registry for live `start` instances (used by `ps`/`stop`); dead PIDs are auto-pruned |
| `~/.lark-channel/media/<chatId>/` | Downloaded images / files, cleaned up after 24h |
| `~/.lark-channel/logs/YYYY-MM-DD.log` | Structured run logs (JSONL), rotated daily; older than 7 days are pruned at startup (`LARK_CHANNEL_LOG_DAYS` env var overrides). `/doctor` reads these. |

> Upgrading from before 0.1.11? Run `lark-channel-bridge migrate` once — it moves anything under `~/.config/lark-channel-bridge/` and `~/.cache/lark-channel-bridge/` to the new location and upgrades `config.json` to the new schema.

## Access control

**Private by default: out of the box, only *you* can use the bot.** "You" = whoever created / owns the Feishu app (the person who scanned the QR to set it up). The bot figures out who the app owner is automatically from Feishu, so **solo use needs zero configuration** — you can DM it and `@`-mention it in any group, and everyone else's messages are silently ignored (no "permission denied" reply, which would only confirm the bot exists).

To let other people or groups in, add them to one of three lists:

| List | Controls | Add | Remove |
|------|----------|-----|--------|
| **Allowed users** | who can DM the bot | `/invite user @them` | `/remove user @them` |
| **Allowed chats** | which groups the bot answers in (for **everyone** in them) | `/invite group` (current group) / `/invite all group` (every group the bot is in) | `/remove group` (current group) |
| **Admins** | who can change settings, and use the bot in any group | `/invite admin @them` | `/remove admin @them` |

> `/invite` and `/remove` can only be run by **you (the creator) and admins**. The `@` in the command points at the *target person* (not the bot) — the bot resolves the mention to their identity, so you never deal with raw IDs.

### Two identities that bypass everything

- **You (the creator)**: subject to no list at all — DMs, any group, every command. You **can never lock yourself out**: even if the lists get messed up, DM the bot and send `/config` to get back in. Transfer the app's ownership in the Feishu console and the bot follows the new owner automatically.
- **Admins**: can DM, run management commands like `/config`, and **bypass the allowed-chats list** — the bot answers them in any group, listed or not. Good for teammates who co-maintain the bot.

### Common setups

- **Just me** → nothing to do; this is the default.
- **Let a teammate DM the bot** → `/invite user @them`
- **Open a work group to everyone in it** → send `/invite group` inside that group
- **First-time setup, onboard every group the bot is already in** → `/invite all group` pulls them all into the list at once; trim with `/remove group` afterwards
- **Add a co-admin** → `/invite admin @them`

### Worth knowing

- Changes take effect on the **next message** — no restart needed.
- **In groups you must `@` the bot first** (DMs don't need it). That's a separate toggle (`/config` → "require @ in groups"), independent of the lists above.
- Strangers get pure silence — no reply at all. The one exception: if someone `@`-mentions the bot in a group that hasn't been opened up, the bot posts a friendly one-liner telling them an admin can run `/invite group` to enable it.

### Advanced: editing the config file directly

If you'd rather not do it inside Feishu, `/invite` and `/config` both write to `preferences.access` in `~/.lark-channel/config.json`:

```json
{
  "preferences": {
    "access": {
      "allowedUsers": ["ou_xxxxxxxxxxxxx"],
      "allowedChats": ["oc_xxxxxxxxxxxxx"],
      "admins":       ["ou_xxxxxxxxxxxxx"]
    }
  }
}
```

`allowedUsers` / `admins` take user `open_id`s; `allowedChats` takes group `chat_id`s. The easiest way to find an ID by hand: have the person message the bot (or `@` it in the group), then check that day's log:

```bash
grep '"event":"enter"' ~/.lark-channel/logs/$(date +%Y-%m-%d).log | tail -5
```

Each line carries `chatId` (group / DM id) and `senderId` (user `open_id`). After a manual edit, **restart the bridge** or send `/reconnect` from any allowed chat to apply it. For day-to-day tweaks `/invite` / `/config` are easier; direct edits are mainly for deployment scripts that pre-seed access.

## FAQ

**The bot stays silent / Claude never replies.** Usually the `claude` CLI itself is not logged in, or the session points to a cwd that no longer exists. Send `/status` to inspect; `/new` to start a fresh session.

**Claude subprocess looks frozen (card stuck on the last frame).** Since 0.1.20 there's an idle watchdog: if Claude emits nothing for N minutes the process is killed and the card is annotated `⏱ N min no response, auto-terminated`. Disabled by default. Enable with `/config` (global, in minutes), or `/timeout 10` to set it on the current session; `/timeout off` disables for the session; `/timeout default` clears the session override.

**Claude says it can't see the image I sent.** Upgrade to the latest version — releases before 0.1.0 had a filename-dedup bug.

## Optional telemetry

By default the bridge reports **nothing**: no metrics, no logs leave your machine, and it pulls in zero telemetry dependencies. The hook below is inert unless you opt in.

To wire up your own monitoring, point an environment variable at a module that default-exports (or exports `createAdapter`) an `AdapterFactory`:

```bash
LARK_CHANNEL_TELEMETRY_MODULE=your-telemetry-package lark-channel-bridge start
```

That module receives every `log.*` event plus error/metric hooks and forwards them wherever you like. The interface is exported from the package root:

```ts
import type { AdapterFactory, TelemetryAdapter, TelemetryEvent } from 'lark-channel-bridge';

const createAdapter: AdapterFactory = (meta) => ({
  emit(event) {/* ship event */},
  recordError(err, ctx) {/* ship exception */},
  recordMetric(name, value, tags) {/* ship metric */},
  flush(timeoutMs) {/* drain buffered events */},
});
export default createAdapter;
```

A missing module, a bad factory, or a throwing adapter all degrade to noop — telemetry can never stop the bridge from starting or break logging.

## License

[MIT](./LICENSE)

<img src="./assets/feedback-group-qr.png" alt="Feedback group QR code" width="360">
