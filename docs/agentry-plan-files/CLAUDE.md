# agentry — standing rules for the coding agent

You are building the **agentry hub**: a Rust daemon (`agentry-hub`) plus a TypeScript Slack
relay (`relay/`) that turn this repo's single-user agent launcher into a Slack-driven,
human-gated, multi-agent system. The existing CLI (`src/`) and WASM actor (`actor/`) stay in
the tree but are NOT part of the new runtime path (see `plans/00-DECISIONS.md` D3).

## Read order — every session, before writing any code

1. This file.
2. `plans/00-DECISIONS.md` — locked choices. Never contradict them silently.
3. `plans/01-SPEC.md` — frozen interface contracts (schema, HTTP API, hook protocol,
   provisioning). All code conforms to SPEC. SPEC changes require a proposed diff and an
   explicit human "approved" before implementation.
4. `plans/PROGRESS.md` — current state. Trust files over memory; you have no memory of
   previous sessions.
5. The single milestone file you were assigned (e.g. `plans/ws0-foundation/M0.1-scaffold.md`).

Then, before touching code: restate the milestone plan in your own words, list any conflicts
you see with SPEC/DECISIONS/current code, and wait for confirmation if conflicts exist.
If there are no conflicts, proceed.

## Scope discipline

- Work ONLY on the assigned milestone. Do not start the next milestone, "improve" other
  areas, or refactor outside the milestone's deliverables list.
- Do not edit files under `plans/` except `plans/PROGRESS.md`. Milestone files, SPEC, and
  DECISIONS are human-owned.
- Do not modify `src/` (legacy CLI) or `actor/` unless the milestone explicitly says so.

## Repo layout (target)

```
Cargo.toml            # workspace: members = ["actor", "hub", "hook"]
src/                  # legacy Model A CLI (frozen; read-only reference)
actor/                # legacy Theater actor (frozen; superseded — see D3)
hub/                  # agentry-hub daemon (Rust, axum + rusqlite)
hook/                 # agentry-hook binary (Rust, minimal deps) — runs inside agent sessions
relay/                # Slack relay (TypeScript, Bolt, Socket Mode)
plans/                # this plan set
scripts/              # existing helpers + new e2e scripts under scripts/e2e/
Makefile              # single entry point for all checks and dev tasks
.env.example          # every env var in SPEC §2, with placeholder values
```

## Commands

- `make preflight` — checks toolchain + env; prints PASS/FAIL per item. Run at session start.
- `make check` — the gate. Must pass before any milestone is declared done:
  `cargo fmt --all --check` && `cargo clippy -p agentry-hub -p agentry-hook --all-targets -- -D warnings`
  && `cargo test -p agentry-hub -p agentry-hook` && `cd relay && npm run check`.
  Note: the `actor` crate targets wasm32 and is EXCLUDED from clippy/test; do not "fix" that.
- `make dev-hub` / `make dev-relay` — run services locally.
- `scripts/e2e/*.sh` — manual end-to-end scripts requiring real credentials; run only when a
  milestone's acceptance criteria say so, never in `make check`.

## Quality bar (production-quality code, demo-scope architecture)

- Rust: no `unwrap()`/`expect()` outside tests and `main()` startup (startup may `expect` with
  a message naming the missing config). Errors via `thiserror` per crate; anyhow only in bins.
- Every log line that concerns a session carries `session_id`; gate lines carry `gate_id`.
  Use `tracing` with the JSON formatter.
- Fail closed: any gate that times out, errors, or can't reach a human is a DENY.
- All external writes (HubSpot, Slack posts on retry paths) are idempotent or deduplicated.
- Config only via environment variables defined in SPEC §2; `.env.example` stays in sync.
- Schema changes only via new numbered migration files; never edit an applied migration.
- Tests are mandatory: unit tests colocated, integration tests in `hub/tests/` against a
  temp-dir SQLite and an ephemeral-port hub. Test doubles (e.g. a `FakeSpawner` implementing
  the `Spawner` trait) are allowed in test code only.

## Banned in runtime code paths

Mocks, stubs, placeholder data, hardcoded sample values, silent fallbacks ("if X fails, just
print/skip"), swallowed exceptions or `let _ =` on fallible results, TODO/FIXME markers,
invented env vars or endpoints not in SPEC, secrets in files or code, `--dangerously-*` flags
added beyond what SPEC specifies, and parallel reimplementations of existing modules instead
of editing them. The ban is on FAKE BEHAVIOR — automated tests are required, not banned.

## Security red lines (non-negotiable)

- HubSpot: the hub refuses to start the gateway unless `HUBSPOT_PORTAL_KIND=sandbox`
  (SPEC §9). Changing this requires a human edit to DECISIONS D7 — never do it yourself.
- Agent sessions never receive SaaS credentials. `HUBSPOT_PRIVATE_APP_TOKEN` exists only in
  the hub's environment.
- Hub and relay bind 127.0.0.1 only. Never add 0.0.0.0 binds or tunnels.
- Gate timeout or error ⇒ deny. No "allow on timeout" option may exist in code.
- Manager-role tokens must be rejected by every gate-decision path (SPEC §6).
- Secrets only via env; commit `.env.example`, never `.env`. Session bearer tokens are stored
  hashed (SHA-256) at rest.
- Never run destructive git operations (force-push, history rewrite, branch deletion) or
  delete rows from the database. Sessions end by status transition, never deletion.

## Stop and ask a human when

- Any credential or env var from SPEC §2 is missing at the point you need it.
- A milestone step conflicts with SPEC, DECISIONS, or existing code.
- You believe a SPEC contract (schema, endpoint, JSON shape) must change.
- An acceptance criterion cannot be met as written.
- Anything would touch a non-sandbox external system, or you're unsure whether it would.
- You are about to add a dependency not listed in the milestone file.

## Session end protocol

1. Run `make check`; paste the result.
2. Walk the milestone's acceptance criteria one by one with the exact command and outcome.
3. Update `plans/PROGRESS.md` (status, notes, blockers, next step).
4. Commit in small, conventional-message commits (`feat(hub): …`, `test(hook): …`).
5. Stop at the milestone boundary. Do not begin the next milestone.
