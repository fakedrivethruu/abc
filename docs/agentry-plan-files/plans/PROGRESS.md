# PROGRESS — agent-maintained state (the only file in plans/ the agent edits)

Update at every session end and at any mid-session checkpoint. Trust this file over memory.
Statuses: `not_started` | `in_progress` | `blocked` | `done` (done = acceptance criteria all
passed AND `make check` green, both evidenced in the log below).

| Milestone | Status | Notes |
|---|---|---|
| M0.1 scaffold | not_started | |
| M0.2 store and schema | not_started | |
| M0.3 spawner and provisioning | not_started | |
| M0.4 container spawner | not_started | parallelizable after M0.3 |
| M1.1 relay socket mode | not_started | |
| M1.2 session commands | not_started | |
| M1.3 recipe generation | not_started | |
| M1.4 email bridge | deferred | do not start (D13) |
| M2.1 plan approval | not_started | |
| M2.2 file write gate | not_started | |
| M2.3 mcp gateway + hubspot | not_started | |
| M3.1 history and transcripts | not_started | |
| M3.2 steering mailbox | not_started | |
| M3.3 manager control plane | not_started | |

## Current blockers

(none)

## Log — newest first

Format per entry:

```
### YYYY-MM-DD — <milestone> — <status after this session>
- Done: …
- make check: PASS | FAIL (paste failing item)
- Acceptance criteria: n/m passing (list any failing, verbatim command + output summary)
- Decisions needing human input: …
- Next step: …
```
