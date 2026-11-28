# agy-acp-bridge

OpenAI-compatible `/v1/chat/completions` over the **official Antigravity CLI**
(`agy acp`, [ACP](https://agentclientprotocol.com) JSON-RPC 2.0 over stdio).
Requests leave through Google's own binary, credentials and harness — no OAuth
token replayed through a foreign HTTP stack, which is the signal that gets
accounts flagged.

```
client ──HTTP──> bridge :20130 ──ACP/stdio──> agy acp ──> Google
```

Stdlib only, no pip installs. Multi-account with primary + fallback, real tool
calling, live session reuse.

## Install

**1. CLI**

```bash
curl -fsSL https://antigravity.google/cli/install.sh | bash   # -> ~/.local/bin/agy
agy acp --help
```

**2. Bridge**

```bash
git clone https://github.com/rassyy/agy-acp-bridge
cd agy-acp-bridge
sudo install -m 644 agy-bridge.service /etc/systemd/system/agy-bridge.service
# edit the unit: set AGY_ADMIN_TOKEN, AGY_BRIDGE_BIND=0.0.0.0 for a remote panel
sudo systemctl daemon-reload && sudo systemctl enable --now agy-bridge
curl -s http://127.0.0.1:20130/health
```

**3. Tool-calling hook** — required for clients that send `tools`

```bash
mkdir -p /root/agy-bridge/work/.agents
cp hooks/hooks.json hooks/deny_builtin.py /root/agy-bridge/work/.agents/
```

The directory must be the **session cwd** (`AGY_CWD`), because customization
discovery walks up from the cwd passed to `session/new`. Next to `bridge.py` it
is silently ignored.

**4. Add an account** — `http://<host>:20130/admin?t=<AGY_ADMIN_TOKEN>`

Profile name → **start login** → open the Google URL, sign in → copy the `code=`
value from the callback URL (the page may look blank) → paste → **submit** →
**test turn**.

**5. Use it**

```bash
curl -s http://127.0.0.1:20130/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"gemini-3.8-flash-medium",
       "messages":[{"role":"user","content":"hello"}]}'
```

Omit `model` to use `AGY_DEFAULT_MODEL`. The provider card also registers
Gemini 4 Hydrogen / Carbon / Argon / Neon and Claude Sonnet / Opus 5.5 tiers;
catalogs differ per account, so an unoffered id returns a clear
`session/set_model: Unknown model ID` error instead of silently using another
model. `"model": "<profile>/<model>"` pins an account. Responses carry `profile` and
`fallback` so you can see which account served the turn.

## Accounts

| status | meaning | action |
|---|---|---|
| `ok` | a real turn returned text | usable |
| `verify` | `403 VALIDATION_REQUIRED` | not banned — open the verification link signed in as that account, re-test |
| `tos` | service disabled for ToS violation | dead |

`fetchAvailableModels` returns 200 with a full model list **even for banned
accounts**. Only a real inference turn tells the truth; never use a metadata or
quota call as a liveness check.

OAuth codes are PKCE-bound to the process that printed the URL — a code from an
older link gives `invalid_grant`. Keep the login process alive while pasting.

## Routing

One primary carries the traffic; the next profile is touched only when the
primary fails. `ok` outranks untested, `tos`/`verify` are never selected, and a
429/quota failure parks a profile for 5 minutes instead of demoting it. Set the
primary from the panel, `POST /admin/primary`, or `AGY_PRIMARY_PROFILE`.

Ban banners arrive as **ordinary assistant text**, not HTTP errors. The bridge
buffers the first ~160 characters of each turn, marks the profile on a ToS or
verification notice, and retries on the next one instead of returning the banner
as an answer.

## Tool calling

ACP has no OpenAI `tools` channel, so the caller's function schemas travel in the
prompt and the model answers with `tool_call`-tagged blocks carrying **OpenAI
function-call shape**:

```json
{"id":"call_1","type":"function","function":{"name":"FN","arguments":"{\"key\":\"value\"}"}}
```

`arguments` is always a JSON **string** on the way out. Prose around the blocks
survives; several blocks become several calls. The bridge extracts them and
**removes the consumed spans from `content`**, so raw JSON never reaches the
client.

Extraction tries four patterns in order: tagged blocks, bare OpenAI-shaped JSON,
the legacy `{"tool_calls":[...]}` envelope, then a bare `{"name":...,"arguments":...}`
object (fenced, as an array, or inside prose). The last one exists because some
agent CLIs — OpenCode among them — put their **own** tool format in the model's
system prompt, which overrides this contract and comes back untagged.

Requiring `id`/`type`/`function` in the bare-JSON pattern is what keeps a
JSON-mode answer like `{"city":"Jakarta","temp":30}` from being eaten as a tool
call; a bare `{name, arguments}` object is only accepted when the name matches a
declared function. Names outside the caller's list are dropped.

**The agent's own tools are never surfaced to a client that declared its own.**
Prompt-level prohibition does not achieve this — the agent's internal system
prompt wins and it still reaches for `run_command`. The working lever is the
CLI's `PreToolUse` hook: `decision: "deny"` hard-blocks execution and its
`reason` string is fed back to the model, which is what turns a blocked action
into the correct output.

Denial is conditional per session. The bridge writes
`<AGY_CWD>/.bridge-mode/<sessionId>.json` when a request declared tools; the
handler reads `conversationId` from its stdin payload and denies only then.
Without client tools the agent keeps its own abilities. Decisions are appended
to `/var/log/agy-bridge-hooks.log`.

The matcher names action tools explicitly and never wildcards: the loop's own
control tools (e.g. `finish`) must stay allowed or a turn never terminates.

If the agent still emits only a native call and no text, the bridge corrects it
**inside the same session** (`AGY_TOOL_RETRIES`, default 1) — asking for a
contract block when the client declared tools, or for a plain-text answer when it
did not — then fails over, or translates as a last resort.

## Session reuse

A request that continues a conversation reuses the ACP session that already
holds its context, so only the new tail of the history is transmitted. Sessions
are keyed on model + message-hash chain; a branched or edited history misses and
creates a fresh session. Every session created is closed — on idle, on eviction,
on failure, and when a same-key entry is superseded.

The primary profile is pre-warmed at boot, because the first `session/new` in a
fresh CLI process costs ~1.5s while later ones cost ~40ms. A turn whose upstream
stream emits no event for `AGY_STALL_TIMEOUT` seconds is cancelled rather than
holding the worker for the full turn timeout.

`GET /health` reports `cache: {hit, miss, closed, sessions}`. Each turn logs
`turn cache=hit|miss sent=NB new=Xms total=Ys`, where `sent` is the prompt size
actually transmitted.

## 9router provider card (optional)

Adds **Antigravity ACP** as a provider card with model rows, per-model **Test**,
and an **Accounts** panel that drives the bridge through a server-side proxy
route — the admin token never reaches the browser.

```bash
python3 9r-patch/apply.py /path/to/9router/build   # after git pull, BEFORE npm run build
cd /path/to/9router/build && npm run build
# swap .next/standalone into the live install, then restart
grep -rl 'Antigravity ACP' .next/static/chunks/    # confirm it compiled in
```

The provider list and dashboard are compiled into client chunks: editing source
changes nothing until you rebuild. `apply.py` is idempotent and stops loudly if
an upstream anchor moved instead of half-applying.

## Gotchas

1. `--app-data-dir` takes a bare directory **name**, not a path — an absolute
   path fails with `must not be absolute`. It resolves under `~/.gemini`.
2. `HOME` must be in the child env or `session/new` dies with
   `language server failed to start: $HOME is not defined`. systemd units do not
   inherit it.
3. Never hold a non-reentrant lock across `rpc()` calls — `chat()` takes the lock
   then calls `rpc()`, which takes it again. Use `RLock`, acquire with a timeout,
   and close the turn generator in a `finally` so a disconnected client cannot
   leak the worker lock.
4. Registry `transport.baseUrl` is used verbatim; 9router only appends
   `/chat/completions` for ids starting with `openai-compatible-`. Use the full
   endpoint or you get a 404 wrapped as `model_not_found`.
5. Don't add a second session check inside a 9router `/api/*` route —
   `dashboardGuard` already handles auth and the extra check 401s the dashboard.
6. `session/set_model` is undocumented but implemented by `agy`. Call it and
   **fail the turn when it rejects the model** — swallowing the error answers
   HTTP 200 from `agy`'s fallback model and lies about which model ran.
6b. Some Gemini tool-call turns emit orphan HTML closing tags as assistant
   content (`</span>`, `</span></span>`) next to `tool_calls`. The bridge drops
   content that is only closing tags, in stream and non-stream paths, so the
   client never shows UI residue instead of an answer.
7. `max_tokens` and `stop` are applied downstream: the model still generates in
   full, so they save output, not time.

## Endpoints

| method | path | purpose |
|---|---|---|
| GET | `/health` | liveness, workers, cache counters |
| GET | `/v1/models` | models from the real CLI, per active profile (named profiles via a HOME shim, since `agy models` has no `--app-data-dir`) |
| POST | `/v1/chat/completions` | stream + non-stream |
| GET | `/admin?t=…` | account panel |
| GET | `/admin/profiles` | profiles, status, primary |
| GET | `/admin/logins` | in-flight logins |
| POST | `/admin/login/start` | `{profile}` → `{loginId, url}` |
| POST | `/admin/login/complete` | `{loginId, code}` |
| POST | `/admin/test` | `{profile, model}` → one real turn |
| POST | `/admin/primary` | `{profile}` |

## Environment

| var | default | meaning |
|---|---|---|
| `AGY_BIN` | `/root/.local/bin/agy` | CLI path |
| `AGY_DEFAULT_MODEL` | `claude-sonnet-4-6` | model used when a request omits `model`; set this to an accepted slug on accounts whose catalog differs (Hydro tiers are `gemini-4-hydrogen-{low,medium,high,xhigh}`) |
| `AGY_BRIDGE_PORT` | `20130` | listen port |
| `AGY_BRIDGE_BIND` | `127.0.0.1` | `0.0.0.0` for a remote panel |
| `AGY_ADMIN_TOKEN` | *(unset = open)* | required as `?t=…` on `/admin*` |
| `AGY_APPDATA_ROOT` | `/root/.gemini` | where CLI profiles live |
| `AGY_CWD` | `/root/agy-bridge/work` | session cwd — hook discovery root |
| `AGY_WORKERS_PER_PROFILE` | `1` | parallel CLI processes per account |
| `AGY_TURN_TIMEOUT` | `300` | seconds per turn |
| `AGY_LOCK_WAIT` | `20` | seconds to wait for a busy worker |
| `AGY_STALL_TIMEOUT` | `120` | no-event seconds before a turn is cancelled |
| `AGY_PRIMARY_PROFILE` | *(unset)* | force the primary |
| `AGY_PREWARM` | `1` | pre-warm at boot (`0` disables) |
| `AGY_PREWARM_PROFILES` | `1` | how many profiles to pre-warm |
| `AGY_SESSION_CACHE` | `1` | session reuse (`0` disables) |
| `AGY_SESSION_IDLE` | `900` | seconds before an idle session is closed |
| `AGY_SESSION_MAX` | `4` | live sessions per worker |
| `AGY_TOOL_RETRIES` | `1` | in-session corrections per turn |

## Layout

```
bridge.py              the bridge (stdlib only)
agy-bridge.service     systemd unit
hooks/
  hooks.json           PreToolUse matcher -> deny_builtin.py
  deny_builtin.py      conditional deny, reason steers the model
9r-patch/
  antigravity-acp.js   provider registry entry
  AcpAccountsCard.js   Accounts panel component
  agy-acp-route.js     server-side bridge proxy (/api/agy-acp/*)
  apply.py             idempotent installer
```

## License

MIT
