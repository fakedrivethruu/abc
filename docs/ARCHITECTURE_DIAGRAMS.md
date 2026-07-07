# agentry — Architecture & Flow Diagrams

> Companion to [`PROJECT_CONTEXT.md`](./PROJECT_CONTEXT.md). Diagrams are Mermaid; render on GitHub,
> in Obsidian, or any Mermaid-aware viewer.
>
> **Legend for status:** solid boxes / lines = **exists today**. Dashed boxes / lines and the
> `«proposed»` tag = **the extension to be scoped** (not yet built). This split matches Parts IX+ of
> the context doc.

---

## 1. Current architecture — the two models (as-built)

```mermaid
flowchart TB
    subgraph human[" "]
        U([Human at laptop])
    end

    subgraph modelA["MODEL A — CLI (src/), host-native"]
        CLI["agentry binary<br/>(runs, exits)"]
        RA["recipe.rs<br/>resolve / search_path"]
        SA["session.rs<br/>JSON state files"]
        JJ["jj workspace add"]
        TM["tmux new-session -d"]
        CLA1["claude (in tmux)"]
        WT["~/work/agentry-sessions/&lt;uuid&gt;/"]
        ST["~/.local/state/agentry/&lt;short&gt;.json"]
    end

    subgraph modelB["MODEL B — Theater actor (actor/) + podman"]
        SH["scripts/agentry-helpers.sh<br/>(curl client)"]
        AC["agentry-actor.wasm<br/>HTTP server on 127.0.0.1:8090<br/>(long-lived, via Theater)"]
        TH["Theater runtime<br/>host fns: tcp + podman + runtime.log"]
        POD["podman"]
        CON["container agent-&lt;recipe&gt;<br/>claude begin (tty)"]
    end

    RC[("Recipes<br/>~/.config/agentry/recipes/&lt;name&gt;/<br/>recipe.toml + CLAUDE.md")]

    U -->|"agentry start &lt;recipe&gt;"| CLI
    CLI --> RA --> RC
    CLI --> JJ --> WT
    CLI --> TM --> CLA1
    CLA1 -.runs in.-> WT
    CLI --> SA --> ST

    U -->|"source helpers;<br/>agentry-spawn &lt;recipe&gt;"| SH
    SH -->|"POST /sessions (JSON spec)"| AC
    AC -->|"import podman.run"| TH --> POD --> CON
    SH -.->|"agentry-attach = podman attach (bypasses actor)"| CON
    SH --> RC

    classDef exists fill:#e8f0fe,stroke:#4285f4,color:#111;
    class CLI,RA,SA,JJ,TM,CLA1,WT,ST,SH,AC,TH,POD,CON,RC exists;
```

**Read this as:** two independent ways to launch the same agent (`claude`). Model A isolates with a
jj workspace + tmux and tracks state in JSON files it deletes on stop. Model B isolates with a podman
container and treats podman's live container list as the only source of truth. The actor is the only
long-lived process in either model.

---

## 2. Target topology — Slack/email multi-agent («proposed»)

```mermaid
flowchart LR
    subgraph chat["Chat surfaces"]
        SL([Slack])
        EM([Email])
    end

    RELAY["Relay process<br/>(Bolt app / mail bridge)<br/>«proposed»"]

    subgraph hub["The actor = coordination hub (extended)"]
        AC["agentry-actor<br/>HTTP on :8090"]
        GEN["POST /recipes<br/>recipe generator «proposed»"]
        HIST["GET /history, /sessions?filter<br/>query surface «proposed»"]
        MCP["MCP server face<br/>list/get/search sessions «proposed»"]
        SM["session state machine<br/>planning→approve→run→done «proposed»"]
    end

    STORE[("Durable history store<br/>(SQLite / append-only)<br/>«proposed»")]
    POD["podman"]

    subgraph agents["Agent containers (many)"]
        C1["agent-&lt;recipe&gt;-&lt;short&gt;<br/>claude + provisioned<br/>.claude/settings.json hooks «proposed»"]
        C2["agent-…"]
    end

    SL <--> RELAY
    EM <--> RELAY
    RELAY <-->|HTTP| AC
    AC --- GEN
    AC --- HIST
    AC --- MCP
    AC --- SM
    HIST <--> STORE
    SM <--> STORE
    AC -->|podman.run| POD --> C1
    POD --> C2
    C1 -.->|"PreToolUse hook<br/>+ plan callback"| RELAY
    C1 -.->|"MCP: what did others do?"| MCP
    MCP <--> STORE

    classDef exists fill:#e8f0fe,stroke:#4285f4,color:#111;
    classDef prop fill:#fef7e0,stroke:#f9ab00,color:#111,stroke-dasharray:5 4;
    class AC,POD exists;
    class RELAY,GEN,HIST,MCP,SM,STORE,C1,C2 prop;
```

**Key idea:** the existing actor becomes the hub. Everything new (recipe generation, an approval
state machine, a durable+queryable history store, an MCP face for agents, and a chat relay) clusters
around that single coordination point. Agents talk *back* to the human through the relay (via hooks)
and to *each other* through the MCP/history face.

---

## 3. Sequence — Steps 1→4 end to end («proposed»)

Prompt → generated recipe → approval → guarded execution with write notifications.

```mermaid
sequenceDiagram
    autonumber
    actor U as User (Slack)
    participant R as Relay (Bolt)
    participant A as agentry-actor (:8090)
    participant L as Claude (recipe generator)
    participant P as podman
    participant C as Agent container (claude)
    participant S as History store

    Note over U,S: STEP 1–2 · prompt → agent-friendly recipe
    U->>R: "/agent new: migrate auth to OAuth in repo X"
    R->>A: POST /recipes {prompt}
    A->>L: meta-prompt (emit schema: role, allowed_tools,<br/>output_contract="write PLAN.md then stop", escalation_rules)
    L-->>A: recipe.toml + CLAUDE.md
    A->>A: write to search path; status=planning
    A-->>R: recipe id + summary
    R-->>U: "Recipe 'oauth-migrate' ready. Generate a plan?"

    Note over U,S: STEP 3 · plan, then human accepts the critical path
    U->>R: 👍 generate plan
    R->>A: POST /sessions {recipe, phase:plan}
    A->>P: podman.run (mount .claude + settings.json hooks)
    P->>C: claude begin
    C->>C: produce /workspace/PLAN.md, then exit
    C-->>A: container exit (phase 1 done)
    A->>S: persist session + PLAN.md ; status=awaiting_approval
    A-->>R: PLAN.md
    R-->>U: render plan + [Approve] [Reject]
    U->>R: Approve
    R->>A: POST /sessions/<id>/approve
    A->>P: podman.run (phase:execute) ; status=running

    Note over U,S: STEP 4 · writes are notified/gated as they happen
    C->>C: about to Write/Edit/Bash
    C->>R: PreToolUse hook → diff (via relay webhook)
    R-->>U: "Agent wants to write auth.rs — [Allow] [Deny]"
    U->>R: Allow
    R-->>C: hook decision = allow
    C->>C: perform write
    C-->>A: phase done (branch/commit + summary)
    A->>S: status=done ; record outcome + transcript pointer
    A-->>R: done
    R-->>U: "✅ oauth-migrate finished — branch oauth-migrate, PR #123"
```

**Where each requirement lives:**
- **1–2** → `POST /recipes` + a meta-prompt that emits a *constrained schema* (that constraint is the
  "fitting").
- **3** → two-phase spawn: phase 1 writes `PLAN.md` and exits; the actor gates on human approval
  before phase 2. Requires the **new persisted `status`** field.
- **4** → a **Claude Code `PreToolUse` hook** (provisioned by agentry into the mounted
  `.claude/settings.json`) that round-trips through the relay. Blocking = gate; non-blocking =
  notify-only.

---

## 4. Sequence — one agent cross-references another's work («proposed»)

```mermaid
sequenceDiagram
    autonumber
    participant C2 as Agent B (claude)
    participant M as actor MCP face
    participant S as History store
    participant TX as Claude transcripts<br/>(~/.claude/projects/*.jsonl)

    Note over C2,TX: Agent B starts work that overlaps Agent A's earlier session
    C2->>M: MCP tool: search_history(repo="X", ticket="AUTH-1")
    M->>S: query sessions where repo=X or ticket=AUTH-1 (incl. stopped)
    S-->>M: [session A: status=done, branch=oauth-migrate,<br/>summary, transcript_ptr]
    M-->>C2: prior work summary + branch + transcript pointer
    opt needs detail
        C2->>M: MCP tool: get_transcript(session=A)
        M->>TX: read indexed transcript
        TX-->>M: action log
        M-->>C2: what Agent A actually did
    end
    C2->>C2: builds on A's branch instead of duplicating it
```

**Prerequisites this depends on (all «proposed», see PROJECT_CONTEXT Part XI):**
1. State becomes **durable / append-only** — `stop` must stop *deleting* records (today: `cmd.rs:226`
   deletes; `podman rm` makes containers vanish).
2. Records **enriched with work product** — branch/commit, summary, transcript pointer.
3. A **query surface** on the actor (`/history`, filters).
4. Agents get a **client** — the actor exposed as an **MCP server** so `claude` can call it as tools.

The join keys are `repo`, and `linked_ticket` (already a field, `session.rs:26`) generalized to also
cover `linked_thread`.

---

## 5. State machine — a session's lifecycle (today vs proposed)

```mermaid
stateDiagram-v2
    direction LR
    [*] --> running_today: agentry start / spawn
    running_today --> gone: stop (state file DELETED)
    gone --> [*]
    note right of running_today
        TODAY: no persisted status.
        "running vs stale" is COMPUTED
        live from tmux/podman.
        Stop erases all trace.
    end note

    state "PROPOSED persisted lifecycle" as prop {
        [*] --> planning
        planning --> awaiting_approval: PLAN.md produced
        awaiting_approval --> running: human approves
        awaiting_approval --> rejected: human rejects
        running --> awaiting_write: PreToolUse hook (gated write)
        awaiting_write --> running: allow
        awaiting_write --> running: deny (skip write)
        running --> done: success
        running --> failed: error
        done --> archived: retained for history
        failed --> archived
        rejected --> archived
    }
```

**The gap in one picture:** today a session is a two-state, trace-erasing thing (exists → gone). The
extension needs a **persisted, multi-state lifecycle** whose terminal states are *archived, not
deleted* — that retention is exactly what makes cross-referencing (§4) possible.

---

## 6. Trust / blast-radius view (security, «to threat-model»)

```mermaid
flowchart TB
    NET([Slack / Internet]) -->|"widens reach «new risk»"| RELAY[Relay process]
    RELAY -->|HTTP, currently UNAUTHENTICATED| AC["actor :8090<br/>(localhost-only assumption breaks)"]
    AC --> POD[podman] --> CON["each agent container"]
    subgraph mounts["Host secrets bind-mounted into EVERY container (agentry-helpers.sh:44-50)"]
        SSH["~/.ssh (ro)"]
        GH["~/.config/gh"]
        CLj["~/.claude.json / ~/.claude"]
        TOK["INBOX_TOKEN"]
    end
    CON --- SSH
    CON --- GH
    CON --- CLj
    CON --- TOK

    classDef risk fill:#fce8e6,stroke:#d93025,color:#111;
    class NET,RELAY,AC,SSH,GH,CLj,TOK risk;
```

**Why this matters:** today the `:8090` API is unauthenticated because it is localhost-only and
single-user. A Slack relay that can reach it — combined with every container mounting `~/.ssh`,
`gh`, and Claude credentials — means a prompt-injected or misbehaving agent has a wide blast radius.
This is Part XIII #6 in the context doc and should be threat-modeled before build: authenticate the
API, scope per-agent credentials, and reconsider the ambient secret mounts.

---

## How to keep these in sync

These diagrams encode the design in [`PROJECT_CONTEXT.md`](./PROJECT_CONTEXT.md). When a proposed
piece gets built, move its node/edge from the dashed `«proposed»` style to the solid `exists` style,
and update the corresponding Part in the context doc.
