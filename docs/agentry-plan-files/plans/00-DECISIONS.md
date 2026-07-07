# 00 — Locked decisions

Each entry is settled. The coding agent implements them as written. To change one: propose a
diff to this file, stop, and wait for explicit human approval. "Status: accepted" means built
or to be built as stated; "Status: accepted (compromise)" means a deliberate scope compromise
the agent must neither "fix" nor forget.

**D1 — Hub in Rust; relay in TypeScript.** The hub (`hub/`, crate `agentry-hub`) is the
durable, stateful, security-sensitive core: axum + tokio + rusqlite. The Slack relay
(`relay/`) is TypeScript on Slack's official Bolt SDK with Socket Mode, because Slack's
first-party SDK support is materially better than community Rust crates. The relay is thin
and stateless; its language never leaks into the hub. Status: accepted.

**D2 — Monorepo.** Everything lives in the existing agentry repo: `hub/` and `hook/` join the
Cargo workspace; `relay/` and `plans/` are top-level directories. No new repositories.
Status: accepted.

**D3 — The Theater actor is superseded.** `actor/` stays in the tree untouched but is not in
the new runtime path; the hub absorbs its role. The hub binds `127.0.0.1:8790` (the actor's
`:8090` is retired; do not run both expecting them to cooperate). Reviving the actor later as
a thin podman shim remains possible but is out of scope. Status: accepted.

**D4 — SQLite, single writer, WAL.** One rusqlite connection owned by a dedicated store task;
all access via typed messages/handles (`tokio-rusqlite` or an mpsc-command wrapper — agent's
choice within this constraint). `PRAGMA journal_mode=WAL`, `foreign_keys=ON`. Migrations are
numbered SQL files applied via `PRAGMA user_version`. Nothing is ever deleted; terminal
states are status transitions. Status: accepted.

**D5 — Spawner is a trait; host first, containers required.** `Spawner` trait with two
implementations: `HostSpawner` (spawns `claude -p` as a host subprocess; M0.3) and
`ContainerSpawner` (podman; M0.4). Host-first is a sequencing choice so WS1/WS2 can proceed
in parallel with container work — it is NOT the production posture. M0.4 is mandatory before
this system is pointed at anything beyond local demos. Selected via `SPAWNER` env.
Status: accepted (host spawner is a compromise until M0.4 lands).

**D6 — Two-layer gating.** (a) Repo/file/shell writes: a provisioned `PreToolUse` hook
(`agentry-hook pretool`) consults per-session policy and round-trips the hub for a human
decision; deny survives `bypassPermissions` by design of Claude Code hooks. (b) Production
SaaS writes: the hub-side MCP gateway (`/mcp/gateway`) holds the credentials, classifies
tools read vs write, passes reads, gates writes, and audits every call. Enforcement lives in
the hook and the gateway — the single choke points — not in prompts. Fail closed everywhere:
timeout/error ⇒ deny. Status: accepted.

**D7 — HubSpot via the official npm server, sandbox-locked.** Upstream is
`@hubspot/mcp-server` spawned by the hub as a per-session stdio child, authenticated with
`HUBSPOT_PRIVATE_APP_TOKEN` from the hub's env only. The hub refuses to start the gateway
unless `HUBSPOT_PORTAL_KIND=sandbox`. Pointing at a production portal requires: WS2
acceptance criteria passed, a human edit to this entry recording sign-off, and the env
change made by a human. Status: accepted (sandbox lock in force).

**D8 — Headless auth is `ANTHROPIC_API_KEY`.** All spawned `claude -p` runs (agents, recipe
generation) authenticate via the API key in their environment, not interactive OAuth.
Status: accepted.

**D9 — Plan approval is two-phase spawn with session resume.** Phase 1 runs with an injected
output contract ("write PLAN.md, then stop") and `--output-format json` to capture Claude's
session id; the hub persists the plan and waits in `awaiting_approval`; approval triggers
phase 2 via `--resume <claude_session_id>`. Claude Code's interactive plan mode is not used.
Status: accepted.

**D10 — Settings are injected via `--settings` and `--mcp-config` flags.** The provisioner
writes per-session `claude-settings.json` and `mcp.json` outside the repo clone and passes
them explicitly, avoiding project-settings trust prompts in headless mode. Claude runs with
`--permission-mode bypassPermissions`; this is safe ONLY because the pretool hook is the
enforcement point (its deny overrides bypass) and, from M0.4, the session is containerized.
Status: accepted (bypass on host spawner is a compromise until M0.4).

**D11 — Hub→relay notifications are HTTP push; the relay is stateless.** The hub stores
`slack_channel_id`/`slack_thread_ts` on the session and includes them in every notification
POST to `RELAY_NOTIFY_URL` (bearer `RELAY_TOKEN`). Retries with backoff; a notification
failure is logged and recorded as an event, never fatal to the session. Status: accepted.

**D12 — The manager never approves gates.** Manager-role session tokens get `/mcp/control`
tools (list/search/read/instruct/spawn) and are rejected with 403 by every gate-decision
path. Gate decisions come only from humans via the relay (admin token). Status: accepted.

**D13 — Email is deferred.** Designed for (generic link fields, transport-agnostic hub),
scoped in `plans/ws1-slack/M1.4-email-bridge.DEFERRED.md`, not to be started. Status:
accepted.

**D14 — MCP endpoints are stateless streamable-HTTP JSON, hand-implemented on axum.**
Single JSON response per POST; no server-side MCP sessions, no SSE streaming required. The
gateway is a JSON-RPC proxy; the control plane implements tools natively. If Claude Code's
HTTP client proves incompatible in practice, the documented fallback is a tiny stdio shim
(`agentry-hook mcp-shim`) bridging stdio↔hub HTTP — see M2.3. Do not adopt an MCP framework
crate without a DECISIONS update. Status: accepted.

**D15 — Recipes stay on the filesystem; TOML gains a `[policy]` table.** The search path and
`recipe.toml` + `CLAUDE.md` format from the legacy CLI are preserved (old code ignores
unknown TOML tables, so compatibility holds). Generated recipes are ordinary files; the DB
never becomes the recipe source of truth. Status: accepted.

**D16 — Workspaces are per-session shallow git clones.** The hub clones the recipe's repo
into `HUB_WORKSPACES_DIR/<uuid>/repo` per session. jj workspaces are not used in the hub
path (per-session clones make the shared-working-copy trick unnecessary). Status: accepted.
