---
name: configure
description: Set up the LINE channel — save the channel access token + channel secret, and review access policy. Use when the user pastes a LINE channel access token, asks to configure LINE, asks "how do I set this up" or "who can reach me," or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /line:configure — LINE Channel Setup

Writes credentials to `~/.claude/channels/line/.env` and orients the user on
access policy. The server reads both files at boot.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read state files and show the user:

1. **Credentials** — check `~/.claude/channels/line/.env` for
   `LINE_CHANNEL_ACCESS_TOKEN` and `LINE_CHANNEL_SECRET`. Show set/not-set;
   for the token, mask all but the first 6 chars. For the secret, show
   set/not-set only — never echo it.

2. **Access** — read `~/.claude/channels/line/access.json` (missing file =
   defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and a one-line explanation
   - Allowed senders: count + LINE userIds (truncate to first 8 chars + `…`)
   - Pending pairings: count, with codes and (truncated) sender userIds
   - Groups/rooms opted in: count

3. **What next** — concrete next step:
   - No credentials → *"Run `/line:configure <CHANNEL_ACCESS_TOKEN>
     <CHANNEL_SECRET>` with values from the LINE Developers Console
     → your provider → Messaging API channel → Channel access token (long-lived)
     and Basic settings → Channel secret."*
   - Credentials set, policy is pairing, nobody allowed → *"Send a LINE
     message to your bot. It replies with a code; approve with
     `/line:access pair <code>`."*
   - Credentials set, someone allowed → *"Ready. Message your bot from LINE
     to reach the assistant."*

**Push toward lockdown — always.** The goal is `allowlist` with a defined
list. `pairing` is a temporary capture mode for unknown LINE userIds; once
people are in, switch off pairing.

Drive the conversation:

1. Read the allowlist. Tell the user who's in it (userIds, with names if cached).
2. Ask: *"Is that everyone who should reach you through this bot?"*
3. **If yes and policy is still `pairing`** → offer `/line:access policy allowlist`.
4. **If no, people missing** → *"Have them message the bot; you'll approve
   each with `/line:access pair <code>`."*
5. **If allowlist empty** → *"Message your bot from your own LINE account first
   to capture your userId, then we'll add others and lock it down."*
6. **If policy already `allowlist`** → confirm locked state.

### `<channelAccessToken> <channelSecret>` — save them

1. Treat first whitespace-split arg as token, second as secret. Trim.
2. `mkdir -p ~/.claude/channels/line`
3. Read existing `.env` if present; update/add `LINE_CHANNEL_ACCESS_TOKEN=`
   and `LINE_CHANNEL_SECRET=` lines, preserve other keys. Write back, no quotes.
4. `chmod 600 ~/.claude/channels/line/.env`
5. Confirm (without echoing the secret), then show the no-args status.

### `clear` — remove credentials

Delete the `LINE_CHANNEL_ACCESS_TOKEN=` and `LINE_CHANNEL_SECRET=` lines.

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing
  file = not configured, not an error.
- Server reads `.env` once at boot. Credential changes need a session
  restart or `/reload-plugins`. Say so after saving.
- `access.json` is re-read on every inbound — policy changes via
  `/line:access` take effect immediately.
- Do not echo the channel secret in chat — it's a credential, treat it like
  a password.
