# 01 — SPEC (frozen interface contracts)

All code conforms to this document. Milestones reference sections as "SPEC §N". Changing
anything here requires a proposed diff and explicit human approval BEFORE implementation.
Additive clarifications approved by a human are appended, never silently assumed.

## §1 Components and ports

| Component | Where | Listens | Auth |
|---|---|---|---|
| agentry-hub | `hub/` (Rust) | `127.0.0.1:8790` | Bearer tokens (§3) |
| relay | `relay/` (TS, Bolt Socket Mode) | outbound to Slack; notify listener `127.0.0.1:8791` | `RELAY_TOKEN` on `/notify` |
| agentry-hook | `hook/` (Rust bin) | none (client) | per-session token from env |
| agent sessions | host subprocess (M0.3) or podman container (M0.4) | none | — |

## §2 Environment variables (complete list; `.env.example` mirrors this)

Hub: `HUB_BIND` (default `127.0.0.1:8790`), `HUB_API_TOKEN` (admin bearer, required),
`HUB_DATA_DIR` (default `~/.local/share/agentry-hub`; holds `hub.db`, `transcripts/`,
`plans/`), `HUB_WORKSPACES_DIR` (default `~/work/agentry-hub-sessions`),
`AGENTRY_RECIPES` (default `~/.config/agentry/recipes`; same convention as legacy CLI),
`ANTHROPIC_API_KEY` (required), `CLAUDE_BIN` (default `claude`), `HOOK_BIN` (absolute path to
built `agentry-hook`, required), `SPAWNER` (`host` | `container`, default `host`),
`AGENT_IMAGE` (default `localhost/agentry-agent:latest`; M0.4),
`RELAY_NOTIFY_URL` (default `http://127.0.0.1:8791/notify`), `RELAY_TOKEN` (required),
`HUBSPOT_PRIVATE_APP_TOKEN` (required for gateway), `HUBSPOT_PORTAL_KIND` (must equal
`sandbox`; gateway refuses to start otherwise — D7).

Relay: `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` (Socket Mode app-level token), `HUB_URL`
(default `http://127.0.0.1:8790`), `HUB_API_TOKEN` (same value as hub's), `RELAY_BIND`
(default `127.0.0.1:8791`), `RELAY_TOKEN` (same value as hub's).

Injected into every agent session by the spawner: `ANTHROPIC_API_KEY`, `AGENTRY_HUB_URL`
(`http://127.0.0.1:8790`), `AGENTRY_SESSION_ID`, `AGENTRY_SESSION_TOKEN`,
`AGENTRY_POLICY_PATH` (absolute path to the session's `policy.json`).

## §3 Auth model

- Admin bearer = `HUB_API_TOKEN`. Used by the relay and human curl. Grants everything except
  nothing — full API.
- Session bearer = 32 random bytes hex, minted at session creation, returned once in the
  spawn plumbing (never in API responses), stored as SHA-256 in `sessions.token_hash`.
  Role `worker` or `manager` (sessions.role). Grants: POST own events, GET/long-poll own
  gates, GET own instructions, POST own transcript, and the MCP endpoint matching its role
  (`worker` → `/mcp/gateway`, `manager` → `/mcp/control`).
- Every gate-decision path (REST §6 and any future surface) rejects session bearers of BOTH
  roles with 403 `manager_cannot_decide` / `worker_cannot_decide`. Only admin decides.
- All endpoints require `Authorization: Bearer <token>` except `GET /healthz`.

## §4 Database (SQLite; migration `hub/migrations/0001_init.sql`)

```sql
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,                 -- uuid v4
  short TEXT NOT NULL UNIQUE,          -- first 8 alphanumeric chars of id
  role TEXT NOT NULL DEFAULT 'worker', -- worker | manager
  mode TEXT NOT NULL,                  -- direct | plan_first
  recipe_name TEXT NOT NULL,
  recipe_path TEXT NOT NULL,
  repo TEXT,                           -- clone source (path or URL); NULL for manager
  workspace_path TEXT,
  spawner TEXT NOT NULL,               -- host | container
  status TEXT NOT NULL,                -- see §5
  phase INTEGER NOT NULL DEFAULT 0,    -- 0 none, 1 plan, 2 execute
  claude_session_id TEXT,              -- from claude -p JSON output; enables --resume
  plan_path TEXT,                      -- $HUB_DATA_DIR/plans/<id>.md once captured
  summary TEXT,                        -- final `result` text from claude -p JSON output
  error TEXT,
  slack_channel_id TEXT,
  slack_thread_ts TEXT,
  linked_ticket TEXT,
  token_hash TEXT NOT NULL,
  created_at TEXT NOT NULL,            -- RFC3339 UTC everywhere
  updated_at TEXT NOT NULL,
  ended_at TEXT
);
CREATE TABLE events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL REFERENCES sessions(id),
  ts TEXT NOT NULL,
  kind TEXT NOT NULL,                  -- §11 enumerates kinds
  tool_name TEXT,
  gate_id INTEGER,
  payload TEXT NOT NULL DEFAULT '{}'   -- JSON
);
CREATE INDEX events_session ON events(session_id, id);
CREATE TABLE gates (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL REFERENCES sessions(id),
  kind TEXT NOT NULL,                  -- file | mcp
  tool_name TEXT NOT NULL,
  summary TEXT NOT NULL,               -- one-line human summary for Slack
  payload TEXT NOT NULL,               -- JSON: full tool input (args may be truncated at 8 KiB)
  status TEXT NOT NULL DEFAULT 'pending',  -- pending | allowed | denied | expired
  decided_by TEXT,                     -- slack user id or 'system:timeout'
  created_at TEXT NOT NULL,
  decided_at TEXT,
  expires_at TEXT NOT NULL
);
CREATE INDEX gates_pending ON gates(status, expires_at);
CREATE TABLE instructions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id TEXT NOT NULL REFERENCES sessions(id),
  source TEXT NOT NULL,                -- 'slack:<user_id>' | 'manager:<session_id>'
  text TEXT NOT NULL,
  created_at TEXT NOT NULL,
  delivered_at TEXT
);
```

## §5 Session status machine

`created → planning → awaiting_approval → executing → done | failed`; `awaiting_approval →
rejected`; `stopped` reachable from any non-terminal status via explicit stop. `mode=direct`
skips `planning`/`awaiting_approval` (created → executing). Terminal: done, failed, rejected,
stopped. Rows are never deleted; `ended_at` set on terminal transition. Every transition
writes a `session_status` event and (if Slack links present) a `session_status` notification.

## §6 Hub HTTP API (prefix `/v1`, JSON; error envelope §12)

Sessions — admin unless noted:
- `POST /v1/sessions` `{recipe, repo?, prompt, mode, role?, slack_channel_id?,
  slack_thread_ts?, linked_ticket?}` → `201 {id, short, status}`. `repo` overrides the
  recipe's; error `no_repository` if neither present (workers only).
- `GET /v1/sessions?status=&repo=&recipe=&thread=` → `{sessions:[…]}` (token_hash omitted).
- `GET /v1/sessions/{id}` → full row + `pending_gate_id?`.
- `POST /v1/sessions/{id}/approve` | `/reject` — valid only in `awaiting_approval`, else 409.
- `POST /v1/sessions/{id}/stop` — kills the running phase, status `stopped`.
- `POST /v1/sessions/{id}/message` `{text}` — valid on sessions with a `claude_session_id`
  and no phase currently running; runs `--resume` with `text` as the prompt (used for
  manager conversations, M3.3).
- `GET /v1/sessions/{id}/events?after_id=0` → `{events:[…]}` (admin or that session's token).
- `POST /v1/sessions/{id}/events` `{kind, tool_name?, payload}` → 201 (session token only).
- `POST /v1/sessions/{id}/transcript` (session token; body = raw JSONL, max 25 MiB) → stores
  `$HUB_DATA_DIR/transcripts/<id>.jsonl`, writes `transcript_stored` event.

Gates:
- `POST /v1/gates` `{kind, tool_name, summary, payload, timeout_seconds?}` (session token;
  session inferred) → `201 {id, expires_at}`. Emits `gate_created` event + notification.
- `GET /v1/gates/{id}?wait=25` (session token, own gates) — long-poll: returns immediately
  if decided, else holds up to `wait` seconds (max 30) then returns current state:
  `{id, status, decided_by?}`.
- `POST /v1/gates/{id}/decision` `{decision: "allow"|"deny", decided_by}` (admin ONLY; 403
  for any session token per §3) — valid only while `pending`, else 409. Emits `gate_decided`
  event; long-pollers wake.
- Background sweeper (every 5 s): `pending` past `expires_at` → `expired`
  (`decided_by='system:timeout'`), treated as deny by all consumers.

Instructions:
- `POST /v1/sessions/{id}/instructions` `{text, source}` (admin, or manager session token —
  the ONLY cross-session write a manager token may perform) → 201.
- `GET /v1/sessions/{id}/instructions?undelivered=true` (that session's token) → returns
  undelivered rows and atomically stamps `delivered_at`; emits `instruction_delivered`.

Recipes & history:
- `POST /v1/recipes` `{prompt}` → generates recipe (M1.3) → `201 {name, path, description}`.
- `GET /v1/recipes` → scan of `AGENTRY_RECIPES` (name, description, path, has_policy).
- `GET /v1/history?repo=&ticket=&thread=&q=&limit=50` → sessions (all statuses) newest-first
  with `{id, short, recipe_name, repo, status, summary, created_at, ended_at}`; `q` is LIKE
  over summary + recipe_name + repo.

`GET /healthz` → `200 {"ok":true,"version":…}` (no auth).

## §7 Hook protocol (`agentry-hook`, invoked by Claude Code per provisioned settings)

Env available: §2 injected set. Reads hook JSON on stdin per Claude Code's contract; writes
JSON to stdout; exit 0 unless the subcommand's contract says otherwise. Network failure
handling: `posttool`/`session-end` log to stderr and exit 0 (observability must not break the
agent); `pretool` on any hub error for a GATED tool emits a DENY (fail closed).

- `pretool` (PreToolUse, no matcher = all tools, `"timeout": 600`): (1) fetch undelivered
  instructions; (2) look up the action for `tool_name` in `policy.json` (§10) — `allow` |
  `deny` | `gate`, default per policy; (3) `deny` → output
  `hookSpecificOutput.permissionDecision:"deny"` with reason "denied by session policy";
  `gate` → POST `/v1/gates` then loop `GET /v1/gates/{id}?wait=25` until decided or the
  policy deadline, then output `allow`/`deny` accordingly (expired ⇒ deny, reason includes
  gate id); `allow` → no permissionDecision. In ALL cases, if instructions were fetched,
  include `additionalContext: "Operator instruction(s): …"` in the same JSON output.
- `posttool` (PostToolUse, all tools): POST `/v1/sessions/{id}/events`
  `{kind:"tool_use", tool_name, payload:{input_digest, ok}}`. Truncate/digest inputs > 8 KiB.
- `stop` (Stop): read stdin; if `stop_hook_active` is true → exit 0 (loop guard). Else fetch
  undelivered instructions; if any → output `{"decision":"block","reason":"Operator
  instruction(s): …"}`; else exit 0.
- `session-end` (SessionEnd): read `transcript_path` from stdin; if file exists and ≤ 25 MiB,
  POST it to `/v1/sessions/{id}/transcript`.
- `mcp-shim` (fallback transport, D14/M2.3): bridge stdio JSON-RPC ↔
  `POST $AGENTRY_HUB_URL/mcp/<target>` with the session bearer. Built only if M2.3's primary
  HTTP transport fails its acceptance test.

## §8 Provisioning (what the spawner materializes per session)

Layout: `HUB_WORKSPACES_DIR/<uuid>/{repo/, claude-settings.json, mcp.json, policy.json}`.
`repo/` is a shallow clone (`git clone --depth 1 <repo> repo`); the recipe's `CLAUDE.md` is
copied to `repo/CLAUDE.md`; `CLAUDE.md` and `PLAN.md` are appended to
`repo/.git/info/exclude` so agents never commit them. Manager sessions get an empty `repo/`.

`claude-settings.json` (rendered; `<HOOK>` = `HOOK_BIN` absolute path):
```json
{
  "hooks": {
    "PreToolUse":  [{"hooks": [{"type": "command", "command": "<HOOK> pretool",  "timeout": 600}]}],
    "PostToolUse": [{"hooks": [{"type": "command", "command": "<HOOK> posttool", "timeout": 30}]}],
    "Stop":        [{"hooks": [{"type": "command", "command": "<HOOK> stop",     "timeout": 30}]}],
    "SessionEnd":  [{"hooks": [{"type": "command", "command": "<HOOK> session-end", "timeout": 60}]}]
  }
}
```

`mcp.json` (only servers granted by recipe policy; literal per-session values, file mode 600):
```json
{"mcpServers": {"hubspot": {"type": "http", "url": "http://127.0.0.1:8790/mcp/gateway",
  "headers": {"Authorization": "Bearer <SESSION_TOKEN>"}}}}
```
Manager sessions instead get `control` → `/mcp/control`.

Invocation (host spawner; container equivalent in M0.4):
```
claude -p "<phase prompt>" --output-format json \
  --settings <ws>/claude-settings.json --mcp-config <ws>/mcp.json \
  --permission-mode bypassPermissions [--resume <claude_session_id>]
```
cwd = `<ws>/repo`, env = §2 injected set. Parse stdout JSON: capture `session_id` →
`sessions.claude_session_id`, `result` → `sessions.summary` (final phase),
`is_error` → failure. Phase prompts: plan phase wraps the user prompt with the output
contract "Write your plan to PLAN.md in the repository root, then stop. Do not implement.";
execute phase is "Execute the approved plan in PLAN.md." with `--resume`.

## §9 MCP gateway (`POST /mcp/gateway`, worker session bearer)

Stateless streamable-HTTP JSON-RPC (one JSON response per POST; no SSE, no server sessions).
Methods: `initialize` (respond with protocol version echo, `capabilities.tools`, serverInfo
`agentry-gateway`), `notifications/initialized` (202 empty), `ping`, `tools/list`,
`tools/call`. Anything else → JSON-RPC `-32601`.

Upstream: per-session lazy `@hubspot/mcp-server` stdio child (env
`PRIVATE_APP_ACCESS_TOKEN=$HUBSPOT_PRIVATE_APP_TOKEN`), initialized once, killed on session
end. Startup precondition: `HUBSPOT_PORTAL_KIND=sandbox` else the hub aborts gateway startup
with a fatal log (D7).

`tools/call` flow: classify via `hub/config/gateway_tools.toml` — `read = [<glob patterns>]`;
anything not matching read is a WRITE (fail closed). Read → forward, record `mcp_call` event
`{tool, direction:"read"}`. Write → create gate `kind=mcp` (summary: tool + salient args),
wait for decision up to policy deadline; allow → forward upstream and record event
`{direction:"write", gate_id, forwarded:true}`; deny/expired → return tool result
`isError:true`, text "Denied by operator (gate <id>). Do not retry without new instructions."
Dedupe: SHA-256 of (session_id, tool, canonical args); an identical write within 10 minutes
of an ALLOWED one returns the cached result instead of re-executing (idempotency).

## §10 Recipe `[policy]` table and rendered `policy.json`

`recipe.toml` additions (legacy CLI ignores them):
```toml
[policy]
default = "allow"              # action for unlisted tools: allow | deny | gate
gate_timeout_seconds = 540     # max 540 (hook budget is 600)
[policy.tools]                 # per-tool overrides; keys are Claude Code tool names
Write = "gate"
Edit = "gate"
MultiEdit = "gate"
Bash = "gate"                  # crm-style recipes set "deny"
[policy.mcp]
servers = ["hubspot"]          # gateway upstreams to grant; [] = no mcp.json entry
```
Rendered `policy.json` = the same data plus `on_timeout: "deny"` (constant; no other value
may exist). MCP write-gating is enforced by the gateway (§9), not the hook — the hook's
policy governs built-in tools only.

## §11 Event kinds

`session_created`, `session_status` (`{from,to}`), `phase_started` (`{phase}`),
`phase_completed` (`{phase, is_error}`), `tool_use`, `gate_created`, `gate_decided`
(`{decision, decided_by}`), `instruction_created`, `instruction_delivered`, `mcp_call`,
`transcript_stored`, `notify_failed`, `error`.

## §12 Errors, notifications, logging

Errors: non-2xx bodies are `{"error":{"code":"snake_case","message":"human text"}}`.

Hub→relay notification: `POST RELAY_NOTIFY_URL`, bearer `RELAY_TOKEN`, retry 3× exponential
backoff, failure ⇒ `notify_failed` event. Body:
```json
{"type": "session_status" | "plan_ready" | "gate_created" | "message",
 "session": {"id","short","recipe_name","status","summary":null,"error":null,
             "slack_channel_id","slack_thread_ts"},
 "plan_excerpt": "…first 3500 chars…",          // plan_ready only
 "gate": {"id","kind","tool_name","summary","expires_at"},   // gate_created only
 "text": "…"}                                    // message only
```
Relay renders: plan_ready → excerpt + Approve/Reject buttons (`action_id`
`plan_approve:<session_id>` / `plan_reject:<session_id>`); gate_created → summary +
Allow/Deny (`gate_allow:<gate_id>` / `gate_deny:<gate_id>`) + expiry note; button handlers
call the corresponding hub endpoints with the admin token and update the message in place.

Logging: `tracing` JSON to stdout; every session-scoped line includes `session_id`, gate
lines `gate_id`. Secrets and bearer values never logged.

## §13 Control-plane MCP (`POST /mcp/control`, manager session bearer) — M3.3

Same transport as §9. Tools (all with JSON schemas and MCP annotations; read-only ones
marked `readOnlyHint: true`): `agentry_list_sessions {status?, repo?, recipe?}`,
`agentry_get_session {id}`, `agentry_get_events {id, after_id?, limit?}`,
`agentry_search_history {q?, repo?, ticket?, limit?}`,
`agentry_send_instruction {session_id, text}` (source = `manager:<caller>`),
`agentry_spawn_session {recipe, prompt, repo?}` (mode=direct, role=worker, inherits the
manager's Slack thread links). There is NO gate-decision tool, by design (D12).
