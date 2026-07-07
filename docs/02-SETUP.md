# 02 — Setup: human checklist vs agent preflight

The coding agent CANNOT do the items in Part A (they require browser consoles, account
ownership, or credential minting). If a Part A item is missing when needed, the agent stops
and asks — it never fabricates credentials, tokens, or workspace IDs.

## Part A — Human-only checklist (do these once, before the milestones that need them)

Needed before M0.1:
1. A Linux or macOS machine (or WSL2). Native Windows is not supported for runtime.
2. Toolchains: rustup with stable Rust; Node.js 20+ and npm; git; sqlite3 CLI. (Nix users:
   `nix develop` covers Rust; Node still required for relay + HubSpot upstream.)
3. Claude Code installed (`claude --version` works) and an Anthropic API key. Put the key in
   `.env` as `ANTHROPIC_API_KEY` (copy `.env.example` → `.env` after M0.1 creates it).
4. Generate `HUB_API_TOKEN` and `RELAY_TOKEN` (e.g. `openssl rand -hex 32` each) into `.env`.

Needed before M1.1:
5. Slack: in a workspace you control, create an app → enable Socket Mode → create an
   app-level token with `connections:write` (`SLACK_APP_TOKEN`, starts `xapp-`) → add bot
   scopes `chat:write` and `commands` → create slash command `/agent` (any placeholder URL;
   Socket Mode ignores it) → install to workspace → copy bot token (`SLACK_BOT_TOKEN`,
   starts `xoxb-`) → invite the bot to your test channel.

Needed before M2.3:
6. HubSpot: create a developer account and a TEST/SANDBOX portal (never your production
   portal — DECISIONS D7). In that portal: Settings → Integrations → Private Apps → create
   app with CRM objects read/write scopes (contacts at minimum) → copy the token into `.env`
   as `HUBSPOT_PRIVATE_APP_TOKEN`, and set `HUBSPOT_PORTAL_KIND=sandbox`.
7. Confirm the official server runs: `PRIVATE_APP_ACCESS_TOKEN=<token> npx -y
   @hubspot/mcp-server` starts without error (Ctrl-C after it initializes).

Needed before M0.4:
8. podman installed; `podman run --rm alpine echo ok` prints ok. Rootless is fine.

Needed before first real demo:
9. A disposable "toy" git repo (local path is fine) the agents can clone and modify.

## Part B — Agent-verifiable preflight

M0.1 delivers `scripts/preflight.sh` (invoked by `make preflight`). It checks, printing one
`PASS`/`FAIL <reason>` line per item and exiting non-zero on any FAIL: rustc/cargo/node/npm/
git/sqlite3/claude present with versions; `.env` exists and every §2-required var for the
CURRENT milestone stage is non-empty (stage passed as `make preflight STAGE=ws2`); `HOOK_BIN`
exists and is executable; `HUB_DATA_DIR`/`HUB_WORKSPACES_DIR` creatable; hub port free;
`HUBSPOT_PORTAL_KIND` equals `sandbox` when set at all. Preflight never prints secret values.

## Part C — Standing operational rules

- `.env` is git-ignored; `.env.example` is committed and kept in exact sync with SPEC §2.
- Rotating a token = update `.env`, restart the affected service; nothing is cached in code.
- The human runs `scripts/e2e/*.sh` demos; the agent may author them but only executes one
  when a milestone's acceptance criteria explicitly says to and Part A prerequisites pass.
