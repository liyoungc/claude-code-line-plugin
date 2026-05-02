# LINE channel for Claude Code

Connect a LINE Official Account to your Claude Code session via an MCP
server. When a colleague messages your LINE OA, the bot forwards the
message into your active Claude Code session and exposes tools to reply,
fetch recent messages, and download attachments.

This is a port of Anthropic's official
[discord plugin](https://github.com/anthropics/claude-code/tree/main/plugins/discord)
to the LINE Messaging API. It is **not** a generic LINE bot framework —
its purpose is to let you discuss code, designs, and decisions on LINE
with your Claude Code instance acting as a participant.

> ⚠️ Different from `hermes-plugin-line`. That plugin adapts a LINE OA into
> the Hermes Agent gateway. **This plugin** plugs a LINE OA into the
> Claude Code CLI itself as a channel.

## Prerequisites

- [Bun](https://bun.sh) — `curl -fsSL https://bun.sh/install | bash`
- [cloudflared](https://github.com/cloudflare/cloudflared) — `brew install
  cloudflared` (only if you want auto-tunneling; see *Tunneling* below)
- A LINE Official Account with the **Messaging API** enabled. Get one at
  [LINE Developers Console](https://developers.line.biz/console/).

## Quick Setup

### 1. Create a LINE Messaging API channel

1. In the [LINE Developers Console](https://developers.line.biz/console/),
   create a Provider, then a **Messaging API** channel.
2. **Basic settings** → copy the **Channel secret**.
3. **Messaging API** → issue a **Channel access token (long-lived)**. Copy it.
4. **Messaging API** → set **Use webhook** to **Enabled**.
5. **Messaging API** → **Auto-reply messages** = Disabled, **Greeting
   messages** = your call (the plugin doesn't override these). Otherwise
   the LINE platform will respond instead of forwarding to your bot.
6. To use the bot in groups: in the OA Manager (admin.line.biz), go to
   **Settings → Response settings** and enable "Allow bot to join groups
   and multi-person chats."

### 2. Install the plugin

These are Claude Code commands — run `claude` to start a session first.

If you've added this repo as a marketplace:

```
/plugin install line@<your-marketplace>
```

Or install directly from GitHub:

```
/plugin install line@github:liyoungc/claude-code-line-plugin
```

### 3. Save credentials

```
/line:configure <CHANNEL_ACCESS_TOKEN> <CHANNEL_SECRET>
```

Writes `LINE_CHANNEL_ACCESS_TOKEN=...` and `LINE_CHANNEL_SECRET=...` to
`~/.claude/channels/line/.env`. The file is `chmod 600`. You can also
write that file by hand or set the variables in your shell environment —
shell takes precedence.

### 4. Relaunch with the channel flag

The MCP server won't connect without this — exit your session and start
a new one:

```sh
claude --channels plugin:line@<your-marketplace>
```

On startup the server will:

- Bind a local HTTP listener on `127.0.0.1:8765`
- Spawn `cloudflared tunnel --url` to get a fresh `https://….trycloudflare.com`
- `PUT https://api.line.me/v2/bot/channel/webhook/endpoint` to register
  that URL with LINE — **no manual webhook URL setup required**

Watch the stderr output (the Claude Code MCP log) for:

```
line channel: webhook listener on http://127.0.0.1:8765/webhook
line channel: cloudflared tunnel up at https://abcd-1234.trycloudflare.com
line channel: webhook endpoint set to https://abcd-1234.trycloudflare.com/webhook
```

### 5. Pair

With Claude Code running from step 4, message your LINE OA — it replies
with a 6-character pairing code. In your Claude Code session:

```
/line:access pair <code>
```

Your next LINE message reaches the assistant.

### 6. Lock it down

Pairing is for capturing LINE userIds. Once you and your colleagues are
in:

```
/line:access policy allowlist
```

## Tunneling

The plugin defaults to a cloudflared **quick tunnel** — fresh
`*.trycloudflare.com` URL on every session, automatically registered with
LINE. This is fine for occasional/personal use.

For a stable URL (e.g. your VPS reverse proxy), set:

```sh
LINE_WEBHOOK_URL=https://yourdomain.example.com/webhook
LINE_TUNNEL=off
```

The plugin will skip cloudflared and `PUT` your fixed URL to LINE.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send a message via LINE Push API. Auto-chunks at 5000 chars. Optional `quote_to` for LINE Quote replies. `files` is parsed but image upload requires hosted URLs (not yet implemented). |
| `fetch_messages` | Read recent inbound messages from this session's in-process buffer. LINE has no history API, so this is best-effort. |
| `download_attachment` | Pull an inbound image/video/audio/file by message ID into `~/.claude/channels/line/inbox/`. |

## Why no `react` / `edit_message`?

LINE doesn't support editing sent messages, and its reaction API is
limited to specific message types and user contexts. The official discord
plugin's `react` and `edit_message` don't have stable LINE equivalents.

## Permission relay

When Claude requests a tool permission, the plugin pushes a prompt to
each allowlisted user:

> 🔐 Permission: Bash
> Reply "yes abcde" to allow or "no abcde" to deny.

Reply in LINE with `yes <code>` or `no <code>` (5-letter code, no `l`).

## Access control

See [ACCESS.md](./ACCESS.md) for full DM policy, group/room enablement,
mention detection, delivery config, and the `access.json` schema.

## Acknowledgements

The architecture and many implementation details follow Anthropic's
official `claude-plugins-official/discord` plugin. License: MIT.
