# Operations runbook

Day-2 operations for the LINE plugin: upgrading, restarting, diagnosing.

## First-time setup / 首次安裝

The plugin is third-party, so Claude Code's channel allowlist blocks it
by default. Three pieces have to line up — most setup failures trace to
exactly one of them.

> 此 plugin 為第三方，預設被 Claude Code 的 channel allowlist 擋。要讓
> LINE 訊息能即時顯示在對話流，三個環節缺一不可，多數失敗都只是其中
> 一個沒對上。

### 1. Channel allowlist (CLI flag) / Channel 白名單（CLI flag）

Wrap your launcher in a shell function:

> 把啟動指令包成 shell function：

```zsh
clauded_line() {
  if [ $# -eq 0 ]; then
    claude --dangerously-load-development-channels plugin:line@claude-code-line-plugin --remote-control --permission-mode bypassPermissions
  else
    claude --dangerously-load-development-channels plugin:line@claude-code-line-plugin --dangerously-skip-permissions "$@"
  fi
}
```

Notes — these all matter:

- The flag is **valued** (takes the `plugin:<name>@<marketplace>` entry),
  not boolean. The value must immediately follow the flag.
- **Do NOT also pass `--channels plugin:line@...`.** The two flags
  populate separate arrays internally; entries from `--channels` go
  through the allowlist gate, entries from the dev flag bypass it. If
  you put the same plugin in both, the allowlist-checked copy fails and
  the warning fires even though the dev copy registered. Drop `--channels`
  entirely — the dev flag both registers AND marks-as-dev.
- `~/.claude/settings.json` keys:
  - `channelsEnabled: true` — required at user level (without it, the
    channel feature is off entirely)
  - `allowedChannelPlugins` — **policy-only** (managed-settings.json /
    MDM); silently ignored when set in user settings. Don't bother adding
    it — the dev flag is your only path on a personal machine.

> 重點：
> - 這個 flag **要帶值**（值是 `plugin:<name>@<marketplace>`），不是
>   boolean。值必須緊跟 flag 後面，否則會被誤認為下一個 flag。
> - **不要同時加 `--channels plugin:line@...`**。兩個 flag 在內部走
>   不同陣列：`--channels` 會經過 allowlist 檢查，dev flag 會被標
>   `dev:true` 略過檢查。同時加會出現重複 entry，被 allowlist 擋的那
>   份觸發警告，整個 plugin 看起來像沒註冊。直接拿掉 `--channels`，
>   dev flag 同時負責註冊跟標記 dev。
> - `~/.claude/settings.json` 設定：
>   - `channelsEnabled: true` — user-level 要設（沒設整個 channel 功能
>     是關的）
>   - `allowedChannelPlugins` — **policy-only**（managed-settings.json
>     / MDM），寫進 user settings 會被靜默忽略。個人機器別浪費時間，
>     dev flag 才是唯一路。

### 2. LINE Official Account "Verify" step / LINE 官方帳號的 Verify 按鈕

Setting the webhook URL via API (`PUT /v2/bot/channel/webhook/endpoint`)
shows it as `active:true` in `GET /v2/bot/channel/webhook/endpoint` and
the API test endpoint (`POST /v2/bot/channel/webhook/test`) succeeds —
but **LINE's edge does not actually deliver real user messages** to the
new URL until you click **Verify** in the LINE Developers Console
(Messaging API tab).

After any webhook URL change (new tunnel hostname, fresh quick-tunnel,
moved between machines), open the console and click Verify. Don't trust
API-only status checks.

> 透過 API (`PUT /v2/bot/channel/webhook/endpoint`) 設定 webhook URL 後，
> `GET` 會回 `active:true`、`POST /v2/bot/channel/webhook/test` 也會
> 成功 — 但 **LINE 的 edge 在你進 LINE Developers Console (Messaging
> API tab) 按下 Verify 之前不會真的把使用者訊息送到新 URL**。任何
> 換 tunnel hostname、quick-tunnel 重起、換機器之後都要去點一下
> Verify，不要相信 API-only 狀態檢查。

Also confirm in the LINE Official Account Manager (`manager.line.biz`):

> 同時去 LINE Official Account Manager (`manager.line.biz`) 確認：

| Setting / 設定 | Value / 數值 |
|---|---|
| Chat mode / 聊天模式 | Bot (not Chat) / Bot（不是 Chat） |
| Webhooks | Enabled / 啟用 |
| Auto-reply messages / 自動回應訊息 | Disabled / 停用 |
| Greeting messages / 加入好友的歡迎訊息 | Disabled (optional) / 停用（可選）|

### 3. Public tunnel actually reachable / Tunnel 真的能通到本機

If `LINE_TUNNEL=off` is in `.env`, the plugin assumes you run cloudflared
yourself (launchd, systemd, separate process). After a machine reboot,
**that service may not auto-start** — check first:

> 如果 `.env` 設了 `LINE_TUNNEL=off`，表示 plugin 不會自己起 cloudflared，
> 你必須自己跑（launchd / systemd / 獨立 process）。**重開機之後該服務
> 可能沒自動起來** — 第一個檢查：

```sh
launchctl list | grep cloudflared    # macOS
systemctl --user status cloudflared  # Linux
```

If missing, restart it. The tunnel terminating endpoint is local — your
public hostname (e.g. `hermes-lynx.liyangchen.me`) resolves to Cloudflare
IPs regardless of which machine actually runs the daemon, so DNS lookup
proves nothing about whether the tunnel is alive.

`bin/line-doctor` will show `502/503` against the public URL when the
tunnel side is down.

> 沒在跑就重啟。Tunnel 的本地端點是 local 的 — 你公開的 hostname
> （例：`hermes-lynx.liyangchen.me`）不管 tunnel daemon 在哪台機器都
> 解析到 Cloudflare 的 IP，所以 DNS 查得到不代表 tunnel 還活著。
> Tunnel 死掉時 `bin/line-doctor` 對 public URL 會回 `502/503`。

## Verifying end-to-end / 端到端驗證

After setup, the success indicators (in order of strength):

> 設好後，由弱到強的成功訊號：

1. **No "approved channels allowlist" warning** at the bottom of Claude
   Code's UI on startup.
   <br/>啟動時 Claude Code 底部**沒有「approved channels allowlist」警告**。
2. **`fetch_messages` returns messages** — confirms webhook → plugin
   pipeline.
   <br/>`fetch_messages` 拿得到訊息 — 證明 webhook → plugin 那段通了。
3. **Inbound LINE message appears as inline `<channel source="plugin:line:line" ...>`
   block** in the conversation — confirms the channel notification
   subscription is wired (this is what triggers Claude to auto-respond).
   <br/>LINE 訊息以 inline `<channel source="plugin:line:line" ...>` 區塊
   出現在對話流 — 證明 channel notification subscription 有 wire 起來
   （這才會 trigger Claude 自動回應）。

If (2) works but (3) doesn't, you're hitting either the CLI flag form
issue or the intra-session double-spawn (see "Multiple plugin processes"
under Common failure modes below).

> (2) 通但 (3) 不通 = 不是 CLI flag 形式錯，就是 intra-session double-spawn
> （見下方 Common failure modes 的相關段落）。

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

### "Cloudflared metrics show 0 inbound but I sent messages" / Cloudflared 顯示沒收到訊息

Real LINE delivery isn't reaching the tunnel. Two causes:

> LINE 真實訊息根本沒送到 tunnel。兩個原因：

1. **LINE OA webhook needs Verify** — see First-time setup §2. The API
   says `active:true` but real events still go to the OLD url until you
   click Verify in the console. Common after any URL change.
   <br/>**LINE OA webhook 要 Verify** — 見 First-time setup §2。API 顯示
   `active:true`，但真實事件還是寄到舊 URL，要進 console 按 Verify。
   換 URL 後最常見的雷。
2. **LINE OA "Use webhook" toggle is off** — same console, Messaging API
   settings. Check `manager.line.biz` for chat mode = Bot, webhooks
   enabled, auto-reply disabled.
   <br/>**LINE OA「Use webhook」toggle 關著** — 同一個 console，Messaging
   API 設定。去 `manager.line.biz` 確認聊天模式 = Bot、webhook 啟用、
   自動回應停用。

`POST /v2/bot/channel/webhook/test` succeeding is misleading — that
bypasses both gates. Trust real-message delivery, not the test endpoint.

> `POST /v2/bot/channel/webhook/test` 成功會誤導你 — 那條 API 兩道閘
> 都繞過。看真實訊息通不通，不要看 test endpoint。

### "Marketplace install pulled the OLD plugin version" / 安裝抓到舊版

Symptom: `~/.claude/plugins/cache/claude-code-line-plugin/line/`
contains old version subdirs only (e.g. `0.0.1/ 0.0.2/`) after
`/plugin install`, even though the github repo has `v0.0.3`.

> 症狀：`/plugin install` 後 cache 目錄裡只有舊版子目錄（例如
> `0.0.1/ 0.0.2/`），即使 github repo 上已經是 `v0.0.3`。

Cause: the marketplace clone at
`~/.claude/plugins/marketplaces/claude-code-line-plugin/` is not
auto-pulled. Manually:

> 原因：`~/.claude/plugins/marketplaces/claude-code-line-plugin/` 這個
> marketplace clone **不會自動 pull**。手動：

```sh
git -C ~/.claude/plugins/marketplaces/claude-code-line-plugin pull --ff-only
```

Then `/plugin uninstall` + `/plugin install` picks up the new version.

> 然後 `/plugin uninstall` + `/plugin install` 才會抓到新版。

### "Inline `<channel>` block doesn't appear though `fetch_messages` works" / fetch_messages 通但 inline 區塊沒出現

Two distinct possibilities:

> 兩個可能：

1. **CLI flag form is wrong** — see First-time setup §1. Either the
   bottom warning about "approved channels allowlist" is still firing
   (channel never registered) or `--channels` was passed alongside the
   dev flag (duplicate entries, allowlist gate fires).
   <br/>**CLI flag 形式錯** — 見 First-time setup §1。要嘛底部還在跳
   「approved channels allowlist」警告（channel 根本沒註冊），要嘛
   `--channels` 跟 dev flag 同時寫（重複 entry，被 allowlist 檔）。
2. **Intra-session double-spawn** — `/plugin uninstall` + `/plugin install`
   without restarting the session can leave the OLD plugin process alive
   while spawning the NEW one. Both are children of the same Claude
   session PID. `ps -ef | grep claude-code-line-plugin` shows two
   `bun run --cwd .../X.Y.Z` lines with different versions and the same
   PPID. The HTTP listener is held by whichever bun won the bind race;
   the MCP buffer read by `fetch_messages` belongs to whichever process
   Claude Code's MCP client connected to last — these can be different
   processes. Webhook arrives at process A's buffer, fetch_messages
   reads process B's empty buffer, channel notification was wired to
   process C... Kill the older bun (`kill <pid>`) then `/reload-plugins`.

   The v0.0.3 self-takeover only handles **cross-session** takeover via
   pidfile, not this **intra-session** case.

   > **Intra-session double-spawn** — 沒重啟 session 直接 `/plugin
   > uninstall` + `/plugin install` 可能讓舊 plugin process 不死，新
   > process 同時 spawn。兩個都是同一個 Claude session 的 child。
   > `ps -ef | grep claude-code-line-plugin` 會看到兩行 `bun run --cwd
   > .../X.Y.Z`、版本不同、PPID 相同。HTTP listener 是 race 贏的那個
   > 拿到；MCP buffer (fetch_messages 讀的那個) 屬於 Claude Code MCP
   > client 最後連上的 process — 這兩個可能是**不同 process**。Webhook
   > 進到 process A 的 buffer，fetch_messages 讀 process B 的空 buffer，
   > channel notification 又 wire 到 process C ... 砍掉舊的 bun
   > (`kill <pid>`) 然後 `/reload-plugins`。
   >
   > v0.0.3 的 self-takeover 只處理**跨 session** takeover (透過 pidfile)，
   > 不處理這種 **intra-session** 的雙重 spawn。

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
