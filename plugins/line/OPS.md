# Operations runbook

Day-2 operations for the LINE plugin: upgrading, restarting, diagnosing.

## Upgrade flow

The hard-learned shape of "upgrade the plugin" is one command short of
obvious. Plugin uninstall does **not** stop a running server; the new
version's port grab will silently fail if the old binary is still bound.

Use this sequence:

```
# In Claude Code
/plugin uninstall line@claude-code-line-plugin
/plugin install   line@claude-code-line-plugin
/reload-plugins

# In a terminal (only if line-doctor below shows orphans / port collision)
~/.claude/plugins/cache/claude-code-line-plugin/line/<version>/bin/line-restart

# Back in Claude Code
/reload-plugins        # let the plugin host respawn the server cleanly
/line:access           # verify status — should match what you had before
```

Since v0.0.3 the server self-takes-over on startup: it reads
`~/.claude/channels/line/server.pid`, sends `SIGTERM` to the previous
instance if alive, and retries the bind. So in **most** cases just
`/reload-plugins` is enough. `line-restart` is the escape hatch for
multi-version-installed-side-by-side scenarios (older versions don't
write the pidfile).

## Diagnostic flow

When LINE messages arrive at the bot but Claude doesn't see them, or vice
versa, run:

```sh
~/.claude/plugins/cache/claude-code-line-plugin/line/<version>/bin/line-doctor
```

It prints, in order:

1. **Port state** — is anyone listening on `LINE_PORT` (default 8765)?
   If yes, is the owner recorded in our pidfile (managed) or stale
   (orphan from a prior version)?
2. **`/health`** — does the local listener actually respond? (Catches
   "process is alive but routes are wrong" cases.)
3. **Public webhook** — POSTs an empty event to your `LINE_WEBHOOK_URL`.
   Reading the codes:
   - `401` → tunnel + path are correct; signature was rightly rejected.
     This is the **happy** state for an empty probe.
   - `404` → server is up but `LINE_WEBHOOK_PATH` doesn't match the route
     LINE actually receives. Common after changing tunnel config.
   - `502/503/504` → tunnel can't reach localhost — Cloudflare tunnel
     down, port wrong, server crashed.
4. **`server.err.log` tail** — persisted errors (port conflicts, registration
   failures). Plugin host swallows stderr of crashed servers; this file is
   how you read the obituary.
5. **Access summary** — DM policy, allowlist count, enabled groups,
   pending pairings.
6. **Discovered groups** — chats Lynx has been added to but you haven't
   `group add`'d yet.

## Common failure modes

### "I sent a message but nothing happens"

Run `line-doctor`. The walk:

| line-doctor says... | Meaning | Fix |
|---|---|---|
| `port NOT LISTENING` | Server died or never started | `/reload-plugins`; if still dead, check `server.err.log` |
| `port LISTENING ... orphaned/manual` | A non-managed server holds the port (likely older version) | `bin/line-restart` then `/reload-plugins` |
| public `404` | Path mismatch | Check `LINE_WEBHOOK_PATH` in `.env` matches what your tunnel forwards |
| public `502/503` | Tunnel can't reach localhost | Restart tunnel; verify `LINE_PORT` matches tunnel target |
| public `200/401` but no MCP forwarding | Server up, but plugin host's MCP stdio not connected | A manually-spawned server.ts won't send to Claude — kill it, `/reload-plugins` |
| `pending: N` but Claude never received pairing reply | `dmPolicy=disabled` or sender userId already in `allowFrom` | `/line:access` to inspect |

### "Webhook PUT failing on every startup"

Since v0.0.3 the server `GET`s the current LINE webhook endpoint and
skips the `PUT` if it already matches. If it still PUTs every time,
either `LINE_WEBHOOK_URL` differs from what's registered, or you're on
a quick tunnel where the URL changes each session (expected).

### "Bad signature on inbound" log floods

Since v0.0.3 the log line includes `user-agent` and body byte length:

```
line channel: bad signature on inbound (ua="LineBotWebhook/2.0" body=312B)
```

- `LineBotWebhook/2.0` UA = LINE platform retrying a real event your
  server failed to verify. Check that `LINE_CHANNEL_SECRET` in `.env`
  matches the secret in LINE Developer Console.
- Other UAs = port scanners / random probes. Safe to ignore.

## File reference

| Path | Purpose | Survives upgrade? |
|---|---|---|
| `~/.claude/channels/line/.env` | Channel token + secret | yes |
| `~/.claude/channels/line/access.json` | Access policy, pending pairings | yes |
| `~/.claude/channels/line/discovered.json` | Pending group enablements | yes |
| `~/.claude/channels/line/server.pid` | Current server PID for self-takeover | rewritten on each start |
| `~/.claude/channels/line/server.err.log` | Persisted critical errors | yes (append-only) |
| `~/.claude/channels/line/inbox/` | Downloaded attachments | yes |
| `~/.claude/channels/line/approved/` | Pairing-approval signal files | transient |
