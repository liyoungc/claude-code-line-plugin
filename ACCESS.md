# LINE — Access & Delivery

LINE allows anyone who adds your Official Account as a friend to message
it. There is no shared-server gate like Discord, so pairing/allowlist on
this side is your **only** access control.

For DMs, the default policy is **pairing**. An unknown sender gets a
6-character code in reply and their message is dropped. You run
`/line:access pair <code>` from your Claude Code session to approve them.

All state lives in `~/.claude/channels/line/access.json`. The
`/line:access` skill commands edit this file; the server re-reads it on
every inbound message, so changes take effect without a restart. Set
`LINE_ACCESS_MODE=static` to pin config to what was on disk at boot
(pairing is unavailable in static mode since it requires runtime writes).

## At a glance

| | |
| --- | --- |
| Default policy | `pairing` |
| Sender ID | LINE userId (33-char, e.g. `U4af49806...`) |
| Group key | LINE groupId or roomId (33-char) |
| Config file | `~/.claude/channels/line/access.json` |

## DM policies

`dmPolicy` controls how DMs from senders not in `allowFrom` are handled.

| Policy | Behavior |
| --- | --- |
| `pairing` (default) | Reply with a pairing code, drop the message. Approve with `/line:access pair <code>`. |
| `allowlist` | Drop silently. No reply. Use this once everyone is in. |
| `disabled` | Drop everything. |

```
/line:access policy allowlist
```

## User IDs

LINE identifies users by **userId** strings (33 chars, opaque). The bot
only sees them when a user messages it — there's no Discord-style "Copy
User ID." Pairing is the primary capture mechanism.

```
/line:access allow U4af49806295e1234abcd...
/line:access remove U4af49806295e1234abcd...
```

## Groups and rooms

LINE has two multi-user contexts:

- **group** — a named group with admin controls (`groupId`)
- **room** — multi-person chat without a group (`roomId`)

Both are off by default. Opt each one in with the appropriate ID.
Identifying IDs: invite the bot to the group, post a `@Lynx`-style
mention, and look at the server stderr log — the `chat_id` in the
forwarded notification is the groupId/roomId.

```
/line:access group add C1234567890abcdef1234567890abcdef
```

With the default `requireMention: true`, the bot only responds when
@-mentioned (LINE's structured mention) or matched against
`mentionPatterns`. Pass `--no-mention` to process every group message.
Pass `--allow id1,id2` to restrict which group members can trigger it.

```
/line:access group add C1234... --no-mention
/line:access group add C1234... --allow U4af...,U5bg...
/line:access group rm C1234...
```

## Mention detection

In groups with `requireMention: true`, any of the following triggers the
bot:

- A structured `@<bot-display-name>` mention in the message — LINE
  attaches `mention.mentionees` with the bot's userId
- An `@all` mention
- A regex match against `mentionPatterns`

```
/line:access set mentionPatterns '["^hey lynx\\b", "\\b@lynx\\b"]'
```

## Delivery

Configure outbound behavior with `/line:access set <key> <value>`.

**`loadingAnimation`** — when `true` (default), the bot calls LINE's
chat-loading-animation API on each inbound 1:1 message so the user sees a
"…" indicator until the bot replies. LINE only supports this in 1:1
chats; in groups it has no effect.

**`textChunkLimit`** — split threshold for long text. LINE's hard cap is
5000 chars per text message. Default 5000.

**`chunkMode`** — `length` (cut at limit) or `newline` (prefer paragraph
boundaries; default).

**`mentionPatterns`** — array of regex strings (case-insensitive) that
count as a mention in groups.

## Skill reference

| Command | Effect |
| --- | --- |
| `/line:access` | Print policy, allowlist, pending pairings, enabled groups. |
| `/line:access pair a4f91c` | Approve pairing code `a4f91c`. |
| `/line:access deny a4f91c` | Discard a pending code. |
| `/line:access allow U4af...` | Add a userId directly. |
| `/line:access remove U4af...` | Remove from the allowlist. |
| `/line:access policy allowlist` | Set `dmPolicy`. |
| `/line:access group add C1234...` | Enable a group/room. Flags: `--no-mention`, `--allow id1,id2`. |
| `/line:access group rm C1234...` | Disable a group/room. |
| `/line:access set <key> <value>` | Set a config key. |

## Config file

`~/.claude/channels/line/access.json`. Absent = `pairing` policy with
empty lists.

```jsonc
{
  "dmPolicy": "pairing",
  "allowFrom": ["U4af49806295e..."],
  "groups": {
    "C1234567890abcdef...": {
      "requireMention": true,
      "allowFrom": []
    }
  },
  "mentionPatterns": ["^hey lynx\\b"],
  "loadingAnimation": true,
  "textChunkLimit": 5000,
  "chunkMode": "newline"
}
```
