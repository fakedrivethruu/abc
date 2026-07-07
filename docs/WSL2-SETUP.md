# WSL2 setup — what I need from you before M0.1

This is the one-time host setup so the agentry-hub can build and run. Do these in order.
Everything here is **on your side** (installing tools, obtaining secrets). Once it's green,
I scaffold M0.1 and run `make preflight` / `make check` to prove it.

Estimated time: ~30–45 min, most of it unattended downloads.

---

## 0. Why WSL2 (not the current Windows/OneDrive path)

- The hub, hook, Makefile, and `preflight.sh` assume a Unix shell — they can't be *verified*
  on native Windows/PowerShell.
- SPEC §8 requires `mcp.json` / `policy.json` at file mode `600`. Windows-mounted paths
  (`/mnt/c/...`) **cannot hold Unix permissions**, so the security model literally can't be
  satisfied there.
- A Rust + Node build tree (`target/`, `node_modules/`) inside **OneDrive** will thrash sync
  and cause file-lock errors.

**Rule: the working copy lives on the WSL2 native filesystem (e.g. `~/agentry`), not under
`/mnt/c` and not in OneDrive.**

---

## 1. Install WSL2 + Ubuntu

In an **Admin PowerShell** (on Windows):

```powershell
wsl --install -d Ubuntu
```

Reboot if prompted, then launch **Ubuntu** from the Start menu and create your Linux user
when asked. Confirm you're on WSL2:

```powershell
wsl -l -v      # STATE=Running, VERSION=2 for Ubuntu
```

Everything below runs **inside the Ubuntu shell**, from your Linux home (`cd ~`).

---

## 2. Base packages

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential pkg-config libssl-dev git curl sqlite3 make
```

- `build-essential` + `libssl-dev` → needed for the Rust crates (reqwest/openssl, rusqlite).
- `sqlite3` → the CLI, so I can inspect the DB during M0.2+.
- `make` → drives the whole workflow.

---

## 3. Rust (stable, via rustup)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh   # accept defaults
source "$HOME/.cargo/env"
rustup component add rustfmt clippy
rustc --version && cargo --version && cargo clippy --version
```

`rustfmt` + `clippy` are required — `make check` runs `cargo fmt --check` and
`cargo clippy -D warnings`.

---

## 4. Node.js 20+ and npm

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version    # v20.x or newer
npm --version
```

The relay is TypeScript (`@slack/bolt`, `tsx`, `vitest`); `make check` runs
`cd relay && npm run check`.

---

## 5. Claude Code CLI + API key  ← the key runtime dependency

The hub spawns real `claude` sessions, so the CLI must work headlessly in WSL2.

```bash
# install however you normally get Claude Code in Linux, then:
claude --version
```

**Auth is headless — API key, NOT interactive OAuth (DECISIONS D8).** Get an Anthropic API
key from the console and have it ready as an env var (I'll wire it into `.env`, never commit
it):

```bash
export ANTHROPIC_API_KEY=sk-ant-...      # I'll move this into .env for you
```

Quick sanity check that headless mode works:

```bash
echo "say hello in 3 words" | ANTHROPIC_API_KEY=sk-ant-... claude -p
```

If that prints a reply, the hub's spawner will work.

---

## 6. Get the repo onto the Linux filesystem

Pick **one**:

**A. Fresh clone (cleanest):**
```bash
cd ~
git clone <this-repo-url> agentry
cd agentry
```

**B. Copy the current working copy out of OneDrive:**
```bash
cp -r "/mnt/c/Users/austi/OneDrive/Documents/GitHub/abc" ~/agentry
cd ~/agentry
```

Then tell me the path (I'll assume `~/agentry`) and I'll work there from now on.

---

## 7. Secrets checklist

| Secret | Needed for | Who provides | When |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | hub spawns agents | **you** | M0.3 (get it now) |
| `HUB_API_TOKEN` | admin bearer | **I generate** | M0.1 |
| `RELAY_TOKEN` | hub↔relay auth | **I generate** | M0.1 |
| `SLACK_BOT_TOKEN` (`xoxb-…`) | relay → Slack | **you** | M1.1 (not now) |
| `SLACK_APP_TOKEN` (`xapp-…`) | Socket Mode | **you** | M1.1 (not now) |
| `HUBSPOT_PRIVATE_APP_TOKEN` | CRM gateway | **you** | M2.3 (not now) |

Only `ANTHROPIC_API_KEY` is required to reach the first live milestone (M0.3). The rest can
wait for their workstream.

---

## 8. Slack app — start whenever you like (blocks M1.1, not M0)

This has real lead time, so you can prep it in parallel. In <https://api.slack.com/apps>:

1. **Create New App** → From scratch → pick your workspace.
2. **Socket Mode** → enable. Generate an **App-Level Token** with scope `connections:write`
   → this is `SLACK_APP_TOKEN` (`xapp-…`).
3. **OAuth & Permissions** → Bot Token Scopes: `chat:write`, `commands`.
4. **Slash Commands** → create `/agent` (request URL can be a placeholder — Socket Mode
   doesn't use it).
5. **Install to Workspace** → copy the **Bot User OAuth Token** → this is `SLACK_BOT_TOKEN`
   (`xoxb-…`).
6. In Slack, create a test channel and **invite the bot** to it.

Hold both tokens until M1.1; nothing inbound/public is ever needed (Socket Mode dials out).

---

## 9. What I do once you confirm

Reply with **"WSL2 ready, use `~/agentry`"** (and hand me `ANTHROPIC_API_KEY` when convenient),
and I will:

1. Scaffold **M0.1** — Cargo workspace (`hub` + `hook`), `relay/` TS package, `Makefile`,
   `scripts/preflight.sh`, `.env.example`, and generate `HUB_API_TOKEN` / `RELAY_TOKEN`.
2. Run `make preflight` and `make check` and show you green.
3. Continue to **M0.2** (store/schema) → **M0.3** (spawner + first live `claude` session),
   then into **M1**.

---

## Quick "am I ready?" one-liner

Run this in `~/agentry` after steps 1–6; all lines should print a version with no errors:

```bash
echo "--- versions ---"; \
rustc --version; cargo --version; cargo clippy --version; \
node --version; npm --version; \
git --version; sqlite3 --version; make --version | head -1; \
claude --version; \
echo "--- key present? ---"; \
[ -n "$ANTHROPIC_API_KEY" ] && echo "ANTHROPIC_API_KEY set" || echo "ANTHROPIC_API_KEY MISSING"
```

Paste me that output and I'll confirm we're clear to scaffold.
