# ghost-bridge — Team-Lead-to-Team-Lead Universal Interteam Comms (v1 spec)

**Status:** Draft. Tactical MVP scope. Generalizes from the RFC #66 PoC (ghost-member-cli) — same substrate primitives, daemon-shaped lifecycle, no CLI in the hot path.

**Date:** 2026-05-13 (session 33)

---

## Goal

Enable a team-lead on one team to send messages to a team-lead on another team using their native `SendMessage` tool — and to receive replies in their native inbox — without any user-CLI relay.

**Concrete first case:** Aen (framework-research team-lead) ↔ Schliemann (apex-research team-lead).

**Out of scope for v1:** arbitrary member-to-member cross-team comms, multi-pair daemons, central messaging hub. Each is named in `## Future` below.

---

## Architecture

**Team-lead-to-team-lead ghost-pair.** Each side registers ONE ghost-member entry in its roster — the other team's lead, as seen from this side. Both sides retain native send/receive UX (SendMessage out, harness inbox-watch wake in). A single bridge daemon transports messages across hosts.

```
  FR side (Windows local-dev)                    apex side (Linux container)
  ─────────────────────────                      ─────────────────────────
                                                                    
  Aen ──SendMessage──> apex-lead-ghost           Schliemann ──SendMessage──> fr-lead-ghost
                       (FR-local inbox)                                      (apex-local inbox)
                              │                                              │
                              │ <─── ghost-bridge.py ───>                    │
                              │      (lives on FR side                       │
                              │       does both directions)                  │
                              ▼                                              ▼
                       ssh-WRITE remote                                  ssh-READ remote
                              │                                              │
                              ▼                                              │
                       apex's team-lead.json    ◄──────  local-WRITE  ◄──────┘
                              │                          FR's team-lead.json
                              ▼                                 ▲
                       harness wakes Schliemann                  │
                                                          harness wakes Aen
```

**Substrate property relied on:** harness watches inbox files for changes and wakes the recipient agent. Validated cross-host in S31 RFC #66 PoC. Filed at `wiki/references/inbox-file-write-as-wake-mechanism.md`.

**Substrate property NOT relied on (deliberately):** SSH server on FR's Windows host. All SSH is FR-daemon-initiated outbound to apex. Windows acts only as SSH client + local file consumer.

---

## Components

### Per-side roster entries

**FR `roster.json`** gains one member:

```json
{
  "name": "apex-lead-ghost",
  "agentType": "ghost",
  "backendType": "ssh-bridge",
  "color": "white",
  "lore": {
    "fullName": "Apex Lead (Schliemann) as seen from FR",
    "origin": "RFC #66 ghost-member pattern; FR's local representation of apex-research's team-lead",
    "significance": "Comm endpoint only — no spawnable agent. SendMessage to this name routes to apex via ghost-bridge daemon."
  }
}
```

**Apex `roster.json`** gains a mirror:

```json
{
  "name": "fr-lead-ghost",
  "agentType": "ghost",
  "backendType": "ssh-bridge",
  ...
}
```

Both sides' harnesses create `inboxes/<ghost-name>.json` files naturally on next session start. No harness change required.

### Daemon: `ghost-bridge.py`

- **Location:** `teams/framework-research/poc/ghost-bridge/ghost-bridge.py`
- **Process model:** long-running python process, launched at FR session start, killed at FR session shutdown.
- **Lifecycle parallel:** mirrors `restore-inboxes.sh` / `persist-inboxes.sh` pattern. Start script `start-ghost-bridge.sh`, stop script `stop-ghost-bridge.sh` (writes PID file for stop).
- **Threading:** single process, polling loop. No threads needed for v1 (one pair).
- **Polling interval:** 2.0 seconds (matches PoC's `WATCH_INTERVAL_S`).
- **SSH primitive:** base64-shipped python snippet over `ssh -i KEY -p PORT user@host`, same as ghost-chat.py's `ssh_exec`. Reuses `APPEND_INBOX_SCRIPT` and `FETCH_AND_MARK_READ_SCRIPT` verbatim from PoC.

### Config: `ghost-bridge.config.json`

```json
{
  "local_team": "framework-research",
  "watch_interval_s": 2.0,
  "pid_file": "ghost-bridge.pid",
  "log_file": "ghost-bridge.log",
  "pairs": [
    {
      "pair_name": "fr-apex",
      "remote_deployment_alias": "apex-research",
      "local_to_remote": {
        "local_outbox_inbox": "apex-lead-ghost",
        "remote_inbox": "team-lead",
        "rewrite_from_to": "fr-lead-ghost"
      },
      "remote_to_local": {
        "remote_outbox_inbox": "fr-lead-ghost",
        "local_inbox": "team-lead",
        "rewrite_from_to": "apex-lead-ghost"
      }
    }
  ]
}
```

- **`local_team`** names the local Claude Code team. Drives all `$HOME/.claude/teams/<local_team>/inboxes/...` path resolution on the local side. Default if absent: `framework-research` (backward-compat for FR-deployed daemons).
- **Deployment alias** resolves SSH host/port/user/key via `~/bin/rc-deployments.json` (existing PoC convention).
- **Path resolution:** all `*_inbox` and `*_outbox_inbox` names map to `$HOME/.claude/teams/<team>/inboxes/<name>.json` on the respective side. `<team>` on the local side is `local_team` from this config; on the remote side is given by the deployment registry.
- **`rewrite_from_to`** is the value the daemon writes into the `from` field on forward. See § Sender-identity rewrite.

---

## Data flow

### Outbound (FR → apex)

1. Aen calls `SendMessage(to="apex-lead-ghost", text="...")`. Harness appends message to `$HOME/.claude/teams/framework-research/inboxes/apex-lead-ghost.json` with `from: "team-lead"` (FR's team-lead roster name; "Aen" is lore-only), `read: false`, standard envelope.
2. Daemon polls this file every 2s; reads under no lock (local fs, single-writer-harness assumption acceptable for v1 — race-safety upgrade noted in § Known limitations).
3. For each entry with `read: false`:
   - Daemon constructs a forward envelope: copies `text`, `summary`, `timestamp` verbatim; overrides `from` with `rewrite_from_to` value (`"fr-lead-ghost"`); sets `read: false`.
   - Daemon SSH-WRITEs to apex's `~/.claude/teams/apex-research/inboxes/team-lead.json` using the `APPEND_INBOX_SCRIPT` primitive (fcntl.flock + atomic read-modify-write).
   - On success: daemon flips `read: true` on the FR-local entry (under local-file rewrite — acceptable v1; see Known limitations).
   - On failure: log to `ghost-bridge.log`, leave entry `read: false`, retry next poll cycle.
4. Apex's harness wakes Schliemann on file change.

### Inbound (apex → FR)

1. Schliemann calls `SendMessage(to="fr-lead-ghost", text="...")` on apex. Apex's harness appends message to `~/.claude/teams/apex-research/inboxes/fr-lead-ghost.json` with `from: "<apex-team-lead-roster-name>"` (likely `"team-lead"` per convention; daemon does not depend on the value), `read: false`.
2. Daemon polls apex over SSH every 2s using `FETCH_AND_MARK_READ_SCRIPT` — single round-trip that fetches `read: false` entries as NDJSON AND flips their flag on apex.
3. For each fetched entry:
   - Daemon constructs a forward envelope: copies `text`, `summary`, `timestamp` verbatim; overrides `from` with `rewrite_from_to` value (`"apex-lead-ghost"`); sets `read: false`.
   - Daemon LOCAL-WRITEs to FR's `$HOME/.claude/teams/framework-research/inboxes/team-lead.json`. No SSH involved — destination is on this host.
4. FR's harness wakes Aen on file change.

---

## Sender-identity rewrite contract

**Rule:** every cross-host forward overwrites the `from` field with the configured `rewrite_from_to` value.

| Direction | Original `from` | Becomes | Receiver sees |
|---|---|---|---|
| FR → apex | `aen` | `fr-lead-ghost` | Message from apex's local label for FR. Consistent with apex's roster. |
| apex → FR | `schliemann` | `apex-lead-ghost` | Message from FR's local label for apex. Consistent with FR's roster. |

**Why overwrite, not preserve original:** if FR's daemon writes a message with `from: "aen"` into apex's inbox, apex's harness has no roster entry for `aen` and the message looks orphan. Overwriting to the locally-known ghost label keeps each side's view consistent with its own roster.

**Information loss accepted in v1:** Aen always sees `apex-lead-ghost` as the sender, never the original apex-side sender's name. For team-lead-to-team-lead bridge, this is fine — the bridge IS between team-leads, so "from apex-lead-ghost" semantically means "from Schliemann." Future: embed original-sender in body or `via` field if member-to-member routing is added.

---

## State tracking — the `read` flag

Uniform semantics across the pipeline: **`read: true` means "consumed by the next link."**

- On FR-local `apex-lead-ghost.json`: `read: false` = daemon hasn't forwarded yet. `read: true` = daemon has forwarded.
- On apex-local `team-lead.json` (after FR daemon writes): `read: false` = Schliemann hasn't read yet. `read: true` = Schliemann has read.
- On apex-local `fr-lead-ghost.json`: `read: false` = daemon hasn't fetched yet. `read: true` = daemon has fetched (PoC's `FETCH_AND_MARK_READ_SCRIPT` flips this atomically).
- On FR-local `team-lead.json` (after daemon writes): `read: false` = Aen hasn't read yet. `read: true` = Aen has read.

No separate `forwarded` field. The `read` flag does double duty as "consumed by next link." Daemon is the "next link" for outbox messages; recipient agent is the "next link" for inbox messages.

---

## Substrate invariants (declared explicitly per FR discipline)

The daemon relies on:

1. **SSH outbound from FR host to apex host** works without server-side reachability of FR.
2. **`~/bin/rc-deployments.json`** registry exists on FR host with an `apex-research` deployment entry.
3. **`python3.7+` + `fcntl`** available on apex (POSIX). Verified in S31 PoC. Daemon's local side (FR/Windows) does NOT require fcntl — uses standard file ops; local race is Known limitation #2.
4. **Local fs write to `$HOME/.claude/teams/framework-research/inboxes/team-lead.json`** works on FR host (Windows fs, single-writer-daemon, harness reads).
5. **Harness inbox-watch wake mechanism** operates as documented in `wiki/references/inbox-file-write-as-wake-mechanism.md`. Validated cross-host in S31.
6. **SSH key + auth pre-configured** for the user invoking the daemon. PoC's existing setup is reused.

**Failure modes if an invariant breaks:**

- (1) breaks → outbound forwards fail; daemon logs and retries. Recoverable when network/SSH restored.
- (2) breaks → daemon fails at startup with explicit error. Manual fix: edit registry.
- (3) breaks → SSH writes/reads fail at script execution. Daemon logs; manual intervention needed.
- (4) breaks → inbound message dropped; Aen receives nothing. Daemon logs.
- (5) breaks → harness-side concern; out of daemon's control.
- (6) breaks → all SSH ops fail; daemon logs.

---

## v1 scope and boundaries

**In scope:**

- One pair: FR ↔ apex via team-leads
- Outbound + inbound, fully native UX
- Sender rewrite per config
- Read-flag state tracking using PoC's existing primitives
- Lifecycle scripts (start/stop) integrated with FR session start/stop
- Logging to a local log file

**Out of scope (deferred):**

- Multi-pair daemons (config is list-shaped to allow data-only growth, but v1 implementation handles only first pair)
- Member-to-member cross-team routing
- Encryption beyond SSH transport
- Daemon restart-on-crash supervisor
- Health monitoring + alerting
- Central messaging hub
- Alternative transports (MCP, HTTP, message broker)

---

## Future evolution path

**Phase 2 (post-validation):** generalize to multi-pair within daemon. Add second pair (e.g., FR ↔ esl-legal) by config addition only. Validates the abstraction.

**Phase 3 (central hub):** when N pairs grows uncomfortable, hoist daemon into a central messaging hub team. Each team's ghost-bridge.config.json points to one hub-destination instead of N peer-destinations. Hub absorbs all SSH labor. Maps cleanly onto S31's "central library team for central library curation" decision direction if the hub IS the library team.

**Phase 4 (transport plurality):** SSH is one transport. MCP-server-shaped transport is another. The `backendType` field on the ghost roster entry already anticipates this. Daemon's SSH-write primitive becomes one of N pluggable transports keyed off `backendType`.

---

## Known limitations (v1)

1. **No supervisor.** Daemon dies → all comms stop until session restart. Logs locally; user notices via missing messages.
2. **Race on FR-local read-flag flip.** Daemon reads + writes `apex-lead-ghost.json` without local fcntl.flock. If harness writes a new message in the same window, the daemon's flag-flip-write could lose it. Mitigation v1: rare (write frequency low); detection via message-count check. Phase 2 fix: use local fcntl.flock on linux side (daemon side, when FR is on Ubuntu) or `msvcrt.locking` on Windows.
3. **No backpressure.** If apex is unreachable, FR's `apex-lead-ghost.json` grows unbounded with `read: false` entries. v1 mitigation: log file size warning at 100 unread.
4. **Polling latency.** 2s polling for inbound = up to 2s reply-arrival latency. Acceptable for human-pace comms; not for tight-loop automation.
5. **Sender-rewrite drops original sender identity.** As noted in § Sender-identity rewrite.
6. **Single-pair coded.** Config supports list, code handles `pairs[0]` only.

---

## Acceptance criteria

For "v1 ships":

1. Aen sends a message via `SendMessage(to="apex-lead-ghost", text="hello")`. Within ~3 seconds, Schliemann's apex harness wakes with the message in their inbox, showing `from: fr-lead-ghost`.
2. Schliemann sends a reply via `SendMessage(to="fr-lead-ghost", text="hi back")`. Within ~3 seconds, Aen's FR harness wakes with the message in `team-lead.json`, showing `from: apex-lead-ghost`.
3. Daemon process runs reliably for the duration of a typical FR session (~hours).
4. Daemon dies cleanly on FR session shutdown — no orphan process.
5. Log file captures sent/received counts and any errors.

---

## Open decisions deferred to implementation

These don't block the spec but need a call during the implementation work:

1. **Daemon log rotation** — file grows unbounded over many sessions; rotate on size? On session-start? Keep it simple: truncate at start of each session.
2. **Empty-target retry behavior** — if remote inbox file doesn't exist yet (e.g., apex hasn't set up `fr-lead-ghost` yet), what does the daemon do? Suggest: skip with warning; retry next cycle.
3. **PID file location convention** — `teams/framework-research/poc/ghost-bridge/ghost-bridge.pid` adjacent to script vs `$HOME/.claude/teams/framework-research/ghost-bridge.pid` adjacent to runtime. Suggest: adjacent to script (config-relative, easier ops).

---

## Apex-side prerequisites (operational checklist)

Before the bridge can work end-to-end, apex must have:

1. `fr-lead-ghost` member entry added to `apex-research`'s `roster.json` with `agentType: ghost`, `backendType: ssh-bridge`.
2. Apex session must have run `TeamCreate` so `~/.claude/teams/apex-research/config.json` exists and includes `fr-lead-ghost` in `members[]`.
3. Schliemann's harness must have the `fr-lead-ghost` entry in its known members for `SendMessage` to route correctly. (Automatic from step 2.)

Apex side does NOT need:

- A daemon process
- Any SSH client config to reach FR
- OpenSSH server enabled

PO arranges apex-side roster edit + session restart via existing direct comm.

---

(*FR:Aen*)
