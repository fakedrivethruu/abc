# agentry — Deep Project Context

> **Purpose of this document.** This is a self-contained, exhaustive briefing on the `agentry`
> project, written so that a language model (or a new engineer) with **no prior exposure** to the
> codebase can fully understand what exists today, how it is built and deployed, and where to plug
> in a specific proposed extension: **running many agents driven from Slack/email, with generated
> recipes, human-in-the-loop approval, write notifications, and cross-referencing of agents' work
> via shared state history.**
>
> Everything in Parts I–VIII describes **what is actually in the repository today** (verified
> against source, with `file:line` references). Part IX onward describes the **proposed extension**
> and is explicitly labeled as design, not existing code.
>
> Repository: `github.com/colinrozzi/agentry`. Language: Rust (2021 edition). License: MIT OR
> Apache-2.0.

---

## Part I — What agentry is

`agentry` is a **local, single-user tool for spinning up AI coding-agent sessions on demand**. The
agent in question is **Claude Code** (the `claude` CLI). agentry does not implement an agent; it is
an **orchestrator / lifecycle manager** that:

1. Stamps out an agent's *identity* from a reusable template (a **recipe**).
2. Gives that agent an *isolated working copy* of a git repository.
3. *Launches* Claude Code against that working copy.
4. *Tracks* what is running and lets you attach to / tear down each session.

### Design philosophy (from `README.md`)

- **One human, one machine, local-only.** "You sit at your laptop, want to spin up agents on demand
  for specific tasks or as long-lived specialists, see what's running, and tear them down cleanly."
- **No service, no central coordinator, no auth surface.** The README's closing section ("Why not a
  service?") is explicit: for the target use case a CLI fits better than a service — nothing to keep
  running, no API to maintain. *(Note: Model B, described below, does introduce one resident
  process, but it is still localhost-only and unauthenticated.)*
- **Recipes are cheap and disposable identity templates**, not heavyweight config.

### Two "shapes" of recipe the design anticipates

- **Bound recipes** (`inbox-dev`, `theater-dev`): repository is fixed in the recipe. These are
  *long-lived specialists*.
- **Generic recipes** (`coding`, `review`, `investigator`): no fixed repository; the repo is supplied
  at spawn time with `--repo`. These are *short-lived task workers*.

### Critically: there are TWO parallel implementations in this repo

The git history shows an evolution from a host-native design to a containerized one. **Both coexist
in the tree.** Understanding that there are two is essential.

| | **Model A** | **Model B** |
|---|---|---|
| Location | `src/` (the `agentry` binary) | `actor/` + `scripts/` |
| Isolation unit | jj workspace + tmux session | podman container |
| Orchestrator | the `agentry` CLI (runs, exits) | a long-lived Theater **WASM actor** on `127.0.0.1:8090` |
| Interface | CLI subcommands | HTTP/JSON, driven by `curl` shell helpers |
| Introduced | commit #1 (`0ed3d28`) | commits #2–#5 |

Git log (most recent first):

```
dcea6e3 scripts: agentry-spawn — workspace + gh + ssh mounts; claude begin cmd (#5)
500d24d scripts: add agentry-helpers.sh for talking to the actor (#4)
58489c9 actor: full CRUD HTTP API + JSON body parsing (#3)
7773ef2 actor: scaffold the WASM theater actor for container orchestration (#2)
0ed3d28 agentry v0: recipes + jj workspaces + tmux session lifecycle (#1)
5c120e3 first commit
```

---

## Part II — Repository layout (every file)

```
.
├── Cargo.toml                     # workspace root + `agentry` binary package
├── Cargo.lock                     # locked dependency graph
├── flake.nix                      # Nix build (intended/canonical build system)
├── flake.lock                     # locked flake inputs
├── .gitignore                     # /target, /result*, /.direnv, *.swp
├── README.md                      # user-facing overview (Model A focused)
├── src/                           # ── MODEL A: the CLI binary ──
│   ├── main.rs                    # clap CLI definition + dispatch
│   ├── cmd.rs                     # subcommand implementations (start/list/show/stop/attach/recipes)
│   ├── recipe.rs                  # Recipe struct, TOML parsing, search-path resolution
│   └── session.rs                 # Session struct, JSON state files, path conventions
├── actor/                         # ── MODEL B: the Theater WASM actor ──
│   ├── Cargo.toml                 # `agentry-actor` package (cdylib, wasm32 target)
│   ├── src/lib.rs                 # no_std WASM component: HTTP server → podman host calls
│   ├── agentry-actor.types        # WIT-like interface: imported host fns + exported fns
│   └── manifest.toml              # Theater deployment descriptor (wasm path + handlers)
├── scripts/                       # ── MODEL B: the client side ──
│   └── agentry-helpers.sh         # bash functions that curl the actor's HTTP API
└── docs/
    └── PROJECT_CONTEXT.md         # (this file)
```

The `agentry-actor` crate is a **workspace member** (`Cargo.toml` → `members = ["actor"]`) but is a
separate compilation unit targeting `wasm32-unknown-unknown`; it is not part of the host binary.

---

## Part III — Model A: the Rust CLI (host-native, jj + tmux)

### III.1 CLI surface (`src/main.rs`)

Uses `clap` derive. Top-level command `agentry` with subcommands:

| Command | Args | Purpose |
|---|---|---|
| `recipes list` | — | Enumerate recipes found in the search path |
| `recipes show <recipe>` | name or path | Print one recipe's metadata + resolved paths |
| `start <recipe>` | `--repo <p>`, `--for <ticket>` | Spawn a session from a recipe |
| `list` | — | List sessions (state files ⋈ live tmux) |
| `show <name>` | name or uuid | Full state for one session |
| `attach <name>` | name or uuid | `tmux attach -t agent-<name>` |
| `stop <name>` | name or uuid | Tear down tmux + workspace + state |

`main()` parses and dispatches to functions in `cmd.rs`. Note `--for` is `r#for` in Rust (reserved
keyword).

### III.2 Recipes (`src/recipe.rs`)

A **recipe** is a TOML file describing an agent identity template. The directory containing
`recipe.toml` is purely organizational — the tool only cares about the file and the paths it
references.

**`Recipe` struct** (`recipe.rs:13`):

```rust
pub struct Recipe {
    pub name: String,                    // short id: "inbox-dev", "coding", "review"
    pub description: String,             // #[serde(default)] — one-liner for `recipes list`
    pub repository: Option<PathBuf>,     // #[serde(default)] — fixed repo, or None (supply via --repo)
    pub claude_md_path: PathBuf,         // relative (to recipe.toml) path to the CLAUDE.md guide
    pub source: PathBuf,                 // #[serde(skip)] — path the recipe was loaded from (internal)
}
```

Example `recipe.toml`:

```toml
name = "inbox-dev"
description = "Mail server specialist"
repository = "/home/colin/work/actors/inbox"
claude_md_path = "./CLAUDE.md"
```

**Key methods / functions:**

- `Recipe::from_path(path)` (`recipe.rs:38`) — read + `toml::from_str`, then set `source = path`.
- `Recipe::claude_md_abs()` (`recipe.rs:49`) — resolve `claude_md_path` against `source`'s parent
  dir → absolute path.
- `Recipe::claude_md_content()` (`recipe.rs:58`) — read that file. **Currently dead code** (compiler
  warns `method claude_md_content is never used`).
- `resolve(reference)` (`recipe.rs:68`) — if `reference` contains `/` **or** ends with `.toml`,
  treat as a direct path; otherwise treat as a name and search `search_path()` for
  `<root>/<name>/recipe.toml`.
- `list_all()` (`recipe.rs:90`) — scan every search-path root for `*/recipe.toml`, parse each,
  **silently skip** ones that fail to parse, sort by name.
- `search_path()` (`recipe.rs:113`) — **`$AGENTRY_RECIPES`** (colon-separated, `$PATH`-style) if set;
  otherwise `[<config_dir>/agentry/recipes]` where `config_dir` is from the `directories` crate
  (`~/.config` on Linux, `%APPDATA%` on Windows).

### III.3 Sessions & state (`src/session.rs`)

A **session** is one running agent. Its entire persistent representation is a single JSON file.

**`Session` struct** (`session.rs:8`, `#[derive(Serialize, Deserialize)]`):

```rust
pub struct Session {
    pub uuid: String,               // full UUID v4
    pub name: String,               // "short": first 8 ALPHANUMERIC chars of the uuid
    pub recipe_name: String,        // identity used
    pub recipe_path: PathBuf,       // where that recipe was loaded from
    pub repository: PathBuf,        // repo the worktree was based on
    pub worktree: PathBuf,          // the jj workspace dir created for this session
    pub tmux_session: String,       // "agent-<short>"
    pub started_at: String,         // RFC3339 timestamp
    pub linked_ticket: Option<String>,  // optional external ticket id (from --for)
}
```

**IMPORTANT — what the Session does NOT record:** there is **no status field**, and **no record of
what the agent did** — no actions, diffs, outputs, transcript pointer, results, or relationships to
other sessions. State captures a session's *configuration and existence*, nothing about its *work*.
Liveness is *computed*, never stored (see `list`).

**Key methods / functions:**

- `Session::save()` (`session.rs:30`) — `mkdir -p state_dir`, write `<name>.json` (pretty JSON).
- `Session::delete()` (`session.rs:40`) — remove `<name>.json`.
- `state_dir()` (`session.rs:49`) — `directories::BaseDirs::state_dir()` **falling back to**
  `data_dir()`, joined with `agentry`. On Linux: `~/.local/state/agentry`. On Windows: `state_dir()`
  is `None`, so it falls back to `%APPDATA%\agentry`.
- `list_all()` (`session.rs:55`) — read every `*.json` in the state dir, deserialize, **skip files
  that fail to parse**, sort by `started_at`.
- `find(name_or_uuid)` (`session.rs:77`) — linear scan of `list_all()` matching `name` **or** `uuid`.
- `short_name(uuid)` (`session.rs:87`) — first 8 ASCII-alphanumeric chars of the uuid.
- `now_rfc3339()` (`session.rs:95`) — `time::OffsetDateTime::now_utc()` formatted RFC3339.
- `worktree_root()` (`session.rs:103`) — reads the **`HOME`** env var (**not** `USERPROFILE`) and
  returns `$HOME/work/agentry-sessions`. **Errors "HOME not set" if `HOME` is absent** (a real
  blocker on Windows).

### III.4 Command implementations (`src/cmd.rs`)

**`start(reference, repo_override, ticket)`** (`cmd.rs:11`) — the core flow:

1. `recipe::resolve(reference)`.
2. Determine repo: `repo_override` → else `recipe.repository` → else **error** "no repository
   specified".
3. Verify the recipe's `CLAUDE.md` exists (`claude_md_abs`); verify the repo dir exists.
4. Mint `uuid` (v4); `short = short_name(uuid)`; `tmux_session = workspace_name = "agent-<short>"`.
5. `mkdir -p worktree_root()`; `worktree = worktree_root/<uuid>`.
6. **`jj -R <repo> workspace add -r main --name agent-<short> <worktree>`** (`cmd.rs:45`). Creates a
   sibling jj working copy checked out at `main`. Fails → abort.
7. **Copy** the recipe's `CLAUDE.md` → `<worktree>/CLAUDE.md` (`cmd.rs:66`). *Verbatim byte copy — no
   templating.*
8. **`tmux new-session -d -s agent-<short> -c <worktree> claude`** (`cmd.rs:70`). Detached tmux
   session, single window running the `claude` CLI, cwd'd into the worktree. **If this fails, roll
   back** the jj workspace (`workspace forget` + `rm -rf`).
9. Build the `Session`, `session.save()`, print attach instructions.

**Why jj, not `git worktree`?** (`README.md` + `cmd.rs:42` comment) `git worktree` pins one branch
per worktree, so two agents couldn't both sit on `main`. `jj workspace add` allows many sibling
working copies on top of the same `main`. **The repo must be jj-colocated.**

**`list()`** (`cmd.rs:126`) — `session::list_all()`, then `tmux_alive_sessions()` (a `HashSet`), then
print a table. Status per row: `running` if the tmux session name is live, else `stale`. Liveness is
computed here, never persisted.

**`show(name)`** (`cmd.rs:182`) — `find()` + live check + print all fields.

**`stop(name)`** (`cmd.rs:202`) — deliberately **best-effort and idempotent**; every external call's
failure is ignored:
1. `tmux kill-session -t <tmux_session>`
2. `jj -R <repo> workspace forget <tmux_session>`
3. `git -C <repo> worktree remove --force <worktree>` (legacy sessions made by an older agentry)
4. `std::fs::remove_dir_all(<worktree>)` (final sweep)
5. `session.delete()` — **removes the state file** (so the session leaves no trace).

**`attach(name)`** (`cmd.rs:232`) — `tmux attach -t <tmux_session>` (blocks; becomes the attached
client).

**`recipes_list()` / `recipes_show()`** (`cmd.rs:247`, `cmd.rs:271`) — print recipe tables /
details.

**`tmux_alive_sessions()`** (`cmd.rs:290`) — `tmux ls -F '#{session_name}'`; returns empty on any
failure (so on a machine without tmux, everything shows `stale`, and read-only commands still work).

### III.5 On-disk footprint (Model A)

| Kind | Path (Linux) | Contents |
|---|---|---|
| Recipes (input, hand-authored) | `~/.config/agentry/recipes/<name>/recipe.toml` + `CLAUDE.md` | Identity templates |
| Session state (bookkeeping) | `~/.local/state/agentry/<short>.json` | One `Session` per live session |
| Workspaces (working trees) | `~/work/agentry-sessions/<uuid>/` | One jj workspace per session |

No database, no locking, no index — flat files. Fine for one human; **races under concurrency** (see
Part IX).

---

## Part IV — Model B: the Theater actor + podman (containerized)

Same recipe/session mental model, but each agent runs in a **podman container**, and orchestration
goes through a **long-lived WASM actor** exposing an HTTP API. Here there genuinely *is* a resident
process.

### IV.1 The actor (`actor/src/lib.rs`)

A `#![no_std]` (with `alloc`) WebAssembly **component** built for `wasm32-unknown-unknown`, producing
`agentry_actor.wasm`. It is a **guest module** for the **Theater** actor runtime — it does not run on
its own; Theater loads it and provides host capabilities. Built with the `packr-guest` crate (v0.6,
`packr_guest::setup_guest!()`).

**Records mirroring the podman host interface** (all `#[derive(GraphValue)]`):

```rust
struct MountSpec     { source: String, target: String, read_only: bool /* graph rename "read-only" */ }
struct ContainerSpec { image: String, name: String, env: Vec<(String,String)>,
                       mounts: Vec<MountSpec>, cmd: Vec<String>, tty: bool, interactive: bool }
struct ContainerInfo { id: String, name: String, image: String, status: String, exit_code: i32 }
struct ActorState    { listener_id: String }   // the actor's persistent state between calls
```

**JSON request body shapes** (`#[derive(Deserialize)]`, parsed with `serde-json-core`):

```rust
struct SessionRequest<'a> { image, name: &'a str, env: Vec<(String,String)>,
                            mounts: Vec<MountReq>, cmd: Vec<String>, tty: bool, interactive: bool }
struct MountReq<'a>       { source, target: &'a str, read_only: bool /* rename "read_only" */ }
```

**Imported host functions** (declared in `agentry-actor.types`, imported in `lib.rs:97+`):

- `theater:simple/runtime.log(msg)`
- `theater:simple/tcp`: `listen(addr) -> id`, `activate(id)`, `receive(id, max) -> bytes`,
  `send(id, bytes) -> u64`, `close(id)`
- `theater:simple/podman`: `run(ContainerSpec) -> id`, `stop(name)`, `rm(name, force)`,
  `list() -> Vec<ContainerInfo>`

**Exported functions** (Theater calls these):

- `theater:simple/actor.init(state) -> (ActorState, ())` (`lib.rs:137`) — logs, `tcp_listen`s on
  `127.0.0.1:8090`, stores the listener id.
- `theater:simple/tcp-client.handle-connection(state, conn_id)` (`lib.rs:149`) — per inbound TCP
  connection: `tcp_activate`, `tcp_receive(conn, 16384)` (**16 KB cap** — no streaming / large-body
  support), `route(bytes)`, `format_response`, `tcp_send`, `tcp_close`.

**The actor is a hand-rolled single-request-per-connection HTTP/1.1 server.** It does not use any
HTTP library. It parses the request line + skips headers itself (`parse_request_head`, `lib.rs:374`:
finds the first `\r\n` for method/path, and `\r\n\r\n` for the body offset) and emits responses with
`Connection: close` (`format_response`, `lib.rs:392`).

**Crucially, the actor never touches podman directly.** It calls *imported host functions*; Theater's
podman handler is what actually talks to the podman socket. The WASM sandbox is the isolation
boundary; the host mediates all real side effects.

### IV.2 HTTP API (`route`, `lib.rs:179`)

| Method | Path | Handler | Behavior | Status |
|---|---|---|---|---|
| GET | `/` | inline | Liveness: `agentry-actor alive` | 200 text/plain |
| GET | `/sessions` | `list_sessions` (`:272`) | `podman_list()` → JSON array | 200 |
| POST | `/sessions` | `start_session` (`:224`) | Parse `SessionRequest` → `ContainerSpec` → `podman_run` | 201 `{name, container_id}` |
| GET | `/sessions/<name>` | `show_session` (`:296`) | `podman_list()` → find by name | 200 / 404 |
| DELETE | `/sessions/<name>` | `delete_session` (`:313`) | `podman_stop` then `podman_rm(force)` (idempotent) | 204 |
| * | * | — | Unmatched | 405 / 400 |

JSON responses are **hand-rolled** (`container_info_json`, `json_error`, `escape_json` at
`lib.rs:337+`) — escapes only `" \ \n \r \t`. Response bodies for a container:
`{"name","image","status","exit_code","container_id"}`.

### IV.3 Deployment descriptor (`actor/manifest.toml`)

```toml
name = "agentry-actor"
version = "0.1.0"
package = "/home/colin/work/actors/agentry/target/wasm32-unknown-unknown/release/agentry_actor.wasm"
[[handler]] type = "runtime"
[[handler]] type = "tcp"
[[handler]] type = "podman"
```

Theater reads this to know which `.wasm` to load and which host capabilities (`runtime`, `tcp`,
`podman`) to wire up. **Note the `package` path is hard-coded to the author's machine** — it must be
edited for any other environment. Start the actor with:

```sh
theater spawn /path/to/agentry/actor/manifest.toml
```

### IV.4 The client side (`scripts/agentry-helpers.sh`)

You `source` this to get shell functions that `curl` the actor. Env-var defaults define the
conventions:

```sh
AGENTRY_ACTOR_URL := http://127.0.0.1:8090
AGENTRY_RECIPES   := $HOME/.config/agentry/recipes
AGENTRY_WORKSPACES:= $HOME/work/agentry-workspaces      # NOTE: different from Model A's ~/work/agentry-sessions
AGENTRY_IMAGE     := localhost/agent-poc:latest
INBOX_TOKEN_FILE  := $HOME/.config/inbox/token
```

**Functions:**

- **`agentry-spawn <recipe>`** — the containerized analog of `agentry start`:
  1. Requires `$AGENTRY_RECIPES/<recipe>/CLAUDE.md`, the inbox token file, and a **pre-cloned**
     workspace at `$AGENTRY_WORKSPACES/<recipe>` (you clone the repo there yourself — **no jj here**).
  2. Builds a container spec (hand-rolled JSON) that:
     - runs image `localhost/agent-poc:latest`, container named `agent-<recipe>`
     - env: `INBOX_TOKEN` (read from the token file), `TERM=xterm-256color`
     - **bind-mounts host credentials + code into the container:**
       | Host source | Container target | RO? |
       |---|---|---|
       | `$HOME/.claude.json` | `/root/.claude.json` | rw |
       | `$HOME/.claude` | `/root/.claude` | rw |
       | `$AGENTRY_WORKSPACES/<recipe>` | `/workspace` | rw |
       | recipe's `CLAUDE.md` | `/workspace/CLAUDE.md` | **ro** |
       | `$HOME/.config/gh` | `/root/.config/gh` | rw |
       | `$HOME/.ssh` | `/root/.ssh` | **ro** |
     - command `["claude", "begin"]`, with `tty: true`, `interactive: true`
  3. `curl -X POST .../sessions` with that spec.
- **`agentry-list`** — `curl .../sessions`.
- **`agentry-show <recipe>`** — `curl .../sessions/agent-<recipe>` (prints http code).
- **`agentry-stop <recipe>`** — `curl -X DELETE .../sessions/agent-<recipe>`.
- **`agentry-attach <recipe>`** — **bypasses the actor**: `exec podman attach agent-<recipe>` (detach
  with Ctrl-p Ctrl-q). Interactive TTY streaming is not supported through the actor's
  request/response API, so attach/exec go straight to podman.
- **`agentry-exec <recipe>`** — `exec podman exec -it agent-<recipe> bash`.

The important architectural takeaway: **the actor handles lifecycle; podman handles the live
terminal.** The actor is the coordination point; the shell helpers are a thin client.

---

## Part V — Data model & state summary

- **Recipe** = static, hand-authored identity template (TOML + a `CLAUDE.md`). Discovered by scanning
  a search path. No generation, no validation beyond TOML parse + existence checks.
- **Session (Model A)** = one JSON file of *metadata* per running agent. Deleted on stop. No status,
  no work record, no history.
- **Session (Model B)** = there is **no agentry-side session record at all**; the source of truth is
  **podman's own container list**, queried live (`podman_list`). A stopped+removed container simply
  disappears. `agent-<recipe>` container names are the only identifiers.
- **Correlation hook:** the `linked_ticket` field (Model A) and the `agent-<recipe>` naming (Model B)
  are the only ways a session is tied to anything external. There is no notion of "session X relates
  to session Y."

---

## Part VI — Build & toolchain

### VI.1 Intended / canonical build (Nix — `flake.nix`)

Uses `rust-overlay` + `crane` for reproducible builds. Outputs:

- `packages.default` — release binary via `craneLib.buildPackage`.
- `packages.clippy` — `cargo clippy -- -D warnings`.
- `packages.fmt` — `cargo fmt` check.
- `devShells.default` — a shell with the Rust toolchain + `ripgrep` + `tmux`.

README workflow:

```sh
nix develop --command cargo build     # debug → ./target/debug/agentry
nix build                             # release via flake
nix profile install /path/to/agentry  # put `agentry` on PATH (this IS the deployment)
nix profile upgrade agentry           # pick up local changes
```

### VI.2 Dependencies

**`agentry` (root `Cargo.toml`):** `clap` 4 (derive), `serde` 1 (derive), `serde_json` 1, `toml`
0.8, `anyhow` 1, `directories` 5, `uuid` 1 (v4), `time` 0.3 (rfc3339). Release profile:
`opt-level="s"`, `lto=true`.

**`agentry-actor` (`actor/Cargo.toml`):** `packr-guest` 0.6 (derive), `serde` 1 (`no_std`,
`alloc`), `serde-json-core` 0.6. `crate-type = ["cdylib"]`.

### VI.3 Actual toolchain setup performed on this Windows machine (2026-07)

The machine had **no Rust toolchain** initially. Setup that made both crates build:

1. Installed **rustup** + Rust **1.96.1** via `winget install Rustlang.Rustup`.
2. The default **MSVC** toolchain failed: `link.exe` not found (no Visual Studio Build Tools).
3. Installed and switched to the **GNU** toolchain: `rustup default stable-x86_64-pc-windows-gnu`.
4. GNU toolchain then needed `dlltool.exe`; installed **WinLibs MinGW-w64 (UCRT)** via
   `winget install BrechtSanders.WinLibs.POSIX.UCRT` (added to user PATH).
5. Added the WASM target: `rustup target add wasm32-unknown-unknown`.

**Result (verified):**

- `cargo build --bin agentry` → `target/debug/agentry.exe` (runs; one harmless dead-code warning for
  `Recipe::claude_md_content`).
- `cargo build -p agentry-actor --target wasm32-unknown-unknown` →
  `target/wasm32-unknown-unknown/debug/agentry_actor.wasm`.

**What is NOT installed** (runtime deps the code shells out to): `jj`, `tmux`, `podman`, `theater`,
`claude`, `nix`. These are Linux/macOS-oriented. On Windows only the read-only CLI commands
(`recipes list/show`, `list`, `show`) work; `start`/`attach` cannot run (`tmux` has no native Windows
build; `HOME` is unset — see `session.rs:104`). **Full operation requires Linux/macOS or WSL2.**

---

## Part VII — Runtime dependency map (what the code invokes)

| External tool | Used by | For |
|---|---|---|
| `jj` (Jujutsu) | Model A (`cmd.rs`) | `workspace add` / `workspace forget` |
| `tmux` | Model A (`cmd.rs`) | detached session running `claude`; liveness query; attach |
| `git` | Model A (`cmd.rs:218`) | legacy `worktree remove` fallback only |
| `claude` (Claude Code CLI) | both | the actual agent process |
| `theater` | Model B | hosts the WASM actor; provides `tcp` + `podman` host functions |
| `podman` | Model B | runs/stops/removes agent containers; direct attach/exec |
| `curl` | Model B client | talks to the actor's HTTP API |
| `nix` | build | canonical reproducible build |

---

# ───────────────────────────────────────────────
# PART IX+ : PROPOSED EXTENSION (design, not yet built)
# ───────────────────────────────────────────────

> Everything above documents existing code. Everything below is the **target design** to be scoped.

## Part IX — The goal

Deploy **many agents driven from a chat surface (Slack, and/or email)** with this flow:

1. **User inputs a prompt** to generate a new recipe.
2. **The recipe is "fitted"** — reshaped into an agent-friendly, constrained identity.
3. **The user accepts the critical path** — a human-in-the-loop approval gate on the plan.
4. **Any write requests are notified to the user directly** — surfaced (and optionally gated) before
   they take effect.

Plus: **agents can cross-reference each other's work via shared state history.**

### Two guiding architectural principles

1. **Build on Model B, not Model A.** Model A mutates flat state files from a short-lived process —
   no coordination point, and it will race under concurrent Slack-driven agents. Model B already
   funnels everything through **one resident process on `:8090`** (`actor/src/lib.rs`). That single
   hub is the correct home for recipe generation, an approval state machine, a Slack relay, and a
   history store.
2. **Three of the four requirements are Claude Code features, not agentry features.** Plan approval
   and write-notification already exist as **plan mode** and **hooks** inside Claude Code. The right
   move is for agentry to **provision Claude Code's config** (the `.claude` dir it already
   bind-mounts, `scripts/agentry-helpers.sh:46-47`) rather than reimplement policy in Rust. agentry's
   job becomes **routing and lifecycle**, not policy.

---

## Part X — Capability-by-capability gap analysis

### X.1 Steps 1–2: prompt → generated, agent-friendly recipe

- **Exists today:** nothing. Recipes are hand-authored; `Recipe::from_path` (`recipe.rs:38`) only
  reads; `CLAUDE.md` is copied verbatim (`cmd.rs:66`).
- **Why it's the cleanest extension:** the recipe layer is already file-based and pluggable —
  `resolve()`/`list_all()` just scan a directory, so anything written there is immediately usable
  with **zero changes** to existing code.
- **Design:** add `POST /recipes {prompt}` to the actor (or `agentry recipes new "<prompt>"` to the
  CLI). It calls Claude once (Anthropic API, or `claude -p` one-shot) with a **meta-prompt that emits
  a schema**, which is what "agent-friendly" means concretely:
  ```
  role, repository, allowed_tools, forbidden_paths, success_criteria,
  output_contract (e.g. "write PLAN.md then stop"), escalation_rules (when to notify the human)
  ```
  Output → `<search-path>/<name>/{recipe.toml, CLAUDE.md}`. The generator turns a fuzzy human prompt
  into a *constrained* agent identity — that reframing is step 2.
- **Schema note:** to support this, the `Recipe` struct (`recipe.rs:13`) should probably gain
  optional fields (`allowed_tools`, `output_contract`, etc.) OR keep them purely inside the generated
  `CLAUDE.md` + a provisioned `.claude/settings.json`. Decision needed (see Part XIII).

### X.2 Step 3: user accepts the critical path (approval gate)

- **Exists today:** nothing. `start` is fire-and-forget (`cmd.rs:70` spawns detached and returns).
  There is **no status field** on `Session` (`session.rs:8`); "running vs stale" is computed live,
  never persisted. No pause, no plan state.
- **Two build options:**
  - **(a) Claude Code plan mode.** The agent produces a plan and waits on `ExitPlanMode`. Problem:
    that approval prompt lives inside the container TTY, invisible to Slack. You must forward it out
    via a hook or MCP that blocks on a Slack reply.
  - **(b) Two-phase spawn** (fits the container lifecycle better). Phase 1: the recipe's
    `output_contract` instructs *"produce `/workspace/PLAN.md`, then exit."* The actor detects
    container exit, reads `PLAN.md`, posts it to Slack, and **waits**. On 👍, phase 2 re-runs the
    container with an "execute the approved plan" command. Run-to-completion → inspect → conditionally
    continue is exactly what podman's lifecycle affords.
- **Required new state:** a **persisted `status` enum** on the session
  (`planning → awaiting_approval → running → done | failed`) plus an approval token. This is the one
  place that genuinely needs new state that doesn't exist today.

### X.3 Step 4: write requests notified to the user

- **Exists today:** zero notification surface. The only outbound signal anywhere is
  `theater:simple/runtime.log` (`lib.rs:97`), which merely logs.
- **Right mechanism: a Claude Code `PreToolUse` hook** — not Rust. A hook matching `Write|Edit|Bash`
  receives the tool's input as JSON on stdin *before it runs*, and can:
  - **notify** (async FYI): POST the diff to Slack and let the call proceed; or
  - **gate** (blocking): return a decision of `ask`/`deny` so the write is held pending approval.
- **Provisioning:** agentry already bind-mounts `~/.claude` into the container
  (`scripts/agentry-helpers.sh:46`). Have agentry write a `settings.json` with that hook at spawn — a
  **config injection, not a Rust change**. The hook's payload target is the Slack relay. Use blocking
  for high-risk writes, notify-only for the rest; branch inside the hook on tool/path.

### X.4 The Slack (and/or email) bridge

- **Exists today:** nothing.
- **Design:** a new process — e.g. a **Slack Bolt app** — that:
  1. maps slash commands / thread messages → the actor's HTTP API (`POST /sessions`, `GET /sessions`,
     `DELETE /sessions/<name>`, and the new `POST /recipes`);
  2. receives the hook + plan callbacks and posts write-notifications / approval prompts back into
     the channel.
- **Mapping unit: thread ↔ session.** This is just generalizing `linked_ticket` (`session.rs:26`)
  into `linked_thread`. That field is already the seed of "correlate a session to an external
  entity."
- Email is the same shape with a different transport (inbound parse + outbound send); the actor stays
  transport-agnostic if the relay is a separate process.

---

## Part XI — Cross-referencing agents' work via state history

**Can an agent reference what other agents have done, today? No — and it is a three-way structural
"no," not a config gap:**

1. **State is metadata-only.** `Session` (`session.rs:8`) records config + existence, **nothing about
   what the agent did**. There is no "work" in the state to reference.
2. **State is current-only, not history.** `stop` **deletes** the state file (`cmd.rs:226`,
   `session.rs:40`); in Model B a removed container simply disappears from `podman_list`. A finished
   agent leaves **no trace**. There is no history — only a snapshot of what is alive right now.
3. **Agents are isolated and can't see the store.** Each runs in its own workspace/container, seeing
   only mounted paths. Nothing mounts the state store in, and there is no query API an agent could
   call. The current design is intentionally **ephemeral + isolated** — the *opposite* of shared
   memory.

**What it takes to enable cross-referencing (this is the one genuinely large piece of work):**

1. **Make state durable / append-only.** Stop deleting on `stop`; instead set `status=stopped`, stamp
   `ended_at`, and retain the record (an `archive/` dir, or a real store — SQLite is the natural fit
   for a queryable single-writer store owned by the actor).
2. **Enrich the record with work product.** Add fields the agent (or the orchestrator) writes back:
   produced branch/commit, a summary, artifact paths, and a pointer to the Claude Code **transcript**
   (Claude Code writes per-session `.jsonl` transcripts under `~/.claude/projects/…` — that *is* the
   full action history; index or link it rather than reinventing it).
3. **Expose a query surface.** Add actor endpoints: `GET /sessions?repo=X&ticket=Y`,
   `GET /history`, `GET /sessions/<name>/transcript`. Cross-referencing then = "find sessions sharing
   a repo, ticket, or Slack thread."
4. **Give agents a client.** Either mount a read-only view of the store into each container, or —
   cleaner — **make the actor an MCP server** so `claude` can call `list_sessions` / `get_session` /
   `search_history` as native tools. That is how one agent asks "what did the others do on this repo?"

**Two existing threads to pull on:** `linked_ticket` is the natural join key for correlation; and
because Model A's jj workspaces are all on the same repo, agents can already see each other's commits
through jj if they share branches — a second, code-level cross-reference channel independent of any
state store.

---

## Part XII — Effort summary

| Requirement | Exists today | What it actually is | Effort |
|---|---|---|---|
| 1–2. Prompt → agent-friendly recipe | ✗ | New generator endpoint; recipe layer already pluggable | **Low** |
| 3. Accept the critical path | ✗ (no status; fire-and-forget) | Claude plan mode **or** two-phase spawn + new `status` state | **Medium** |
| 4. Notify on writes | ✗ (no notify surface) | Claude Code `PreToolUse` hook + agentry provisions `settings.json` | **Low–Med** |
| Slack/email front end | ✗ | New relay process ↔ actor HTTP; thread↔session (extend `linked_ticket`) | **Medium** |
| Cross-reference via history | ✗ (metadata-only, deleted on stop, isolated) | Durable+enriched store + query API + MCP client for agents | **High** (redesign) |

---

## Part XIII — Open design questions to resolve during scoping

1. **Model A or B as the base?** (Recommendation: B — it has the single coordination point. Confirm,
   or decide to fold A's jj-workspace isolation into B.)
2. **Where does recipe structure live?** Extend the `Recipe` TOML schema, or keep constraints entirely
   in the generated `CLAUDE.md` + provisioned `.claude/settings.json`? (Affects `recipe.rs`.)
3. **Approval model:** Claude plan mode piped to Slack, vs. two-phase (`PLAN.md`) spawn? The latter
   needs a persisted session state machine; the former needs a blocking hook/MCP round-trip.
4. **State store technology:** keep flat JSON but stop deleting, vs. move the actor to SQLite for
   concurrent-safe, queryable history. (Concurrency: many Slack-driven agents will read/write state
   at once — the actor should be the **single writer**; flat files with no locking will race.)
5. **How much history to retain, and where?** Just metadata + outcome, or full transcript indexing?
   Retention/GC policy?
6. **Security surface:** Model B bind-mounts `~/.ssh`, `~/.config/gh`, `~/.claude*` into every
   container (`agentry-helpers.sh:44-50`). A Slack-triggered, multi-agent deployment widens the blast
   radius considerably. The `:8090` API is currently unauthenticated and localhost-only — that
   assumption breaks the moment a Slack relay can reach it. Threat-model this explicitly.
7. **Attach/observability from Slack:** interactive TTY can't stream through the actor's
   request/response API (today attach goes straight to podman). How does a Slack user "watch" an
   agent — tail the transcript, periodic summaries, or a separate streaming channel?
8. **Idempotency / naming:** Model B keys containers by `agent-<recipe>`, so only one live session per
   recipe. Multi-agent-per-recipe needs a new naming scheme (e.g. `agent-<recipe>-<short>`) and a
   real session id, closing the gap with Model A's uuid/short scheme.

---

## Appendix A — Quick reference

**Env vars:**

| Var | Default | Consumed by |
|---|---|---|
| `AGENTRY_RECIPES` | `~/.config/agentry/recipes` | recipe search path (`recipe.rs:114`) — colon-separated |
| `HOME` | (required) | `worktree_root` (`session.rs:104`) |
| `AGENTRY_ACTOR_URL` | `http://127.0.0.1:8090` | shell helpers |
| `AGENTRY_WORKSPACES` | `~/work/agentry-workspaces` | `agentry-spawn` |
| `AGENTRY_IMAGE` | `localhost/agent-poc:latest` | `agentry-spawn` |
| `INBOX_TOKEN_FILE` | `~/.config/inbox/token` | `agentry-spawn` |

**Naming conventions:** `short` = first 8 alphanumeric chars of a uuid v4; tmux session + jj
workspace + container all named `agent-<short>` (Model A) or `agent-<recipe>` (Model B).

**Actor listen address:** `127.0.0.1:8090` (`lib.rs:131`).

**Build commands (Windows/GNU, as set up here):**

```powershell
cargo build --bin agentry
cargo build -p agentry-actor --target wasm32-unknown-unknown
```

## Appendix B — Glossary

- **Recipe** — a reusable agent identity template (`recipe.toml` + `CLAUDE.md`).
- **Session** — one running agent instance (a jj workspace + tmux session, or a podman container).
- **jj (Jujutsu)** — a git-compatible VCS; `jj workspace add` gives multiple working copies on the
  same commit (unlike `git worktree`, which pins one branch per tree).
- **Theater** — the WASM actor runtime that hosts `agentry-actor` and provides `tcp` + `podman` host
  capabilities.
- **packr-guest** — the crate providing the WASM guest bindings / `GraphValue` derive used by the
  actor.
- **Claude Code (`claude`)** — the AI coding agent that actually does the work inside each session.
- **Hook / plan mode / MCP** — Claude Code features (pre-tool-use interception, plan approval, and a
  tool-provider protocol) that the extension should leverage rather than reimplement.
