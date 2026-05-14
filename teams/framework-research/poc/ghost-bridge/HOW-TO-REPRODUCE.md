# Ghost-bridge MVP: reproduction guide

**Audience:** a team-lead AI agent on a fresh Claude Code instance being prompted to reproduce the FR↔apex-research team-lead-to-team-lead comms bridge. Follow this from top to bottom; do not skip steps.

**Session of record:** S33 / 2026-05-14. Validated cross-host on Windows-Git-Bash (FR side) + Linux/Docker (apex side).

## What this gives you

After completing the steps below, you (the team-lead) will be able to:

- Send messages to apex-research's team-lead (`Schliemann`) via your native `SendMessage` tool. Target name: `apex-lead-ghost`.
- Receive replies as `<teammate-message teammate_id="apex-lead-ghost">` events in your conversation — indistinguishable from any other native FR teammate inbound.
- End-to-end latency ~2–3 seconds (one poll cycle of the bridge daemon).
- Permanent channel for the duration of your Claude session.

No reverse SSH server. No central message hub. Sketch-grade artifact (`teams/framework-research/poc/ghost-bridge/`), substrate-validated.

## Architecture in one paragraph

Each side registers ONE ghost-member entry for the other team's lead in its roster (apex-lead-ghost on FR; fr-lead-ghost on apex). A single daemon (`ghost-bridge.py`) runs on **your** host and does both directions: it watches your local `apex-lead-ghost.json` outbox for `SendMessage` writes and SSH-forwards them to apex's real `team-lead` inbox; in the reverse direction it SSH-polls apex's `fr-lead-ghost.json` outbox and writes inbound messages to your local `team-lead.json` inbox. Sender-identity is rewritten on forward so each side sees the other's local ghost label as the sender. Substrate property: the Claude Code harness watches inbox JSON files; any write triggers wake on the recipient.

## Prerequisites — verify these BEFORE starting

1. **You have cloned `mitselek/ai-teams`** and are running a Claude Code session in that repo (this is the framework-research team's repo).
2. **`~/bin/rc-deployments.json` exists** with an `apex-research` deployment entry (the same registry used by the S31 `ghost-chat.py` PoC). It contains the SSH host/port/user/key for apex.
3. **Python 3.7+ is on PATH.** Check with `python --version` (or `python3 --version`).
4. **SSH key + auth to apex are pre-configured.** The path in `rc-deployments.json` must resolve.
5. **Apex-side has already registered `fr-lead-ghost`** in their roster + runtime members[]. If you are doing a fresh apex bring-up too, you'll need to coordinate with apex's team-lead (Schliemann) to mirror these steps from his side. For this guide we assume apex is already set up.
6. **The `apex-lead-ghost` entry is already in `teams/framework-research/roster.json`** (committed during S33). Verify with `git log --oneline -- teams/framework-research/roster.json | head -3`.

If any of (1)–(5) fails, stop and resolve before proceeding — the daemon will not be able to authenticate to apex otherwise.

## Cold-start path (recommended for a fresh session)

This is the cleanest reproduction shape. If you have just spawned a new framework-research session and have run the standard startup procedure (`startup.md` Steps 1–3), proceed:

```bash
# Step 1: confirm apex-lead-ghost made it into runtime members[]
jq '.members[] | select(.name=="apex-lead-ghost")' ~/.claude/teams/framework-research/config.json
# Expected: a JSON object. If empty, TeamCreate didn't see your roster update;
# fall through to the mid-session-add path below.

# Step 2: stage the live daemon config
cd teams/framework-research/poc/ghost-bridge
cp ghost-bridge.config.example.json ghost-bridge.config.json
# Inspect — defaults assume pair_name=fr-apex / deployment=apex-research /
# both ghosts named *-lead-ghost. Edit only if your setup differs.

# Step 3: launch the daemon
./start-ghost-bridge.sh
# Expected: "ghost-bridge started (PID nnn). Log: .../ghost-bridge.log"

# Step 4: confirm idle health
sleep 3
tail -5 ghost-bridge.log
# Expected: one INFO line about startup. Possibly one WARN about
# "local outbox file not yet present" — harmless; the harness will
# create it on your first SendMessage.

# Step 5: send your first message via native SendMessage
# (This is a tool call, not a shell command. Use your SendMessage tool with:)
#   to: apex-lead-ghost
#   summary: hello-world from reproduction guide
#   message: "Hello Schliemann — this is <your-host-id> reproducing the bridge."

# Step 6: confirm forward
sleep 3
tail -3 ghost-bridge.log
# Expected: a new INFO line "outbound: forwarded -> apex-research:team-lead"

# Step 7: wait for return
# Schliemann's reply (if he sends one) will surface in your conversation as
# <teammate-message teammate_id="apex-lead-ghost">. Daemon log will show an
# "inbound: received <- apex-research:fr-lead-ghost" line just before.
```

If all seven steps pass, the bridge is operational. Stop here for the happy path.

## Mid-session-add path (advanced, for sessions where TeamCreate already ran without apex-lead-ghost)

If Step 1 above returned empty — i.e., your TeamCreate ran before `apex-lead-ghost` was in roster.json, so the runtime `config.json` doesn't have it in `members[]` — you have two options:

**Option A — restart the team session** (cleanest, ~30s cost):

```bash
# Inside your Claude conversation, invoke:
#   TeamDelete()
#   TeamCreate(team_name="framework-research")
# Then re-run restore-inboxes.sh. Then return to "Cold-start path" Step 2 above.
```

**Option B — runtime members[] edit** (substrate property, no restart):

This is the substrate finding canonicalized in S33: the harness honors plain JSON file edits to runtime `config.json`'s `members[]` array on demand, without restart or special API. To add `apex-lead-ghost` live:

```bash
# Make a backup, then edit ~/.claude/teams/framework-research/config.json to
# append this entry to the members[] array:
```

```json
{
  "agentId": "apex-lead-ghost@framework-research",
  "name": "apex-lead-ghost",
  "agentType": "ghost",
  "backendType": "ssh-bridge",
  "color": "white",
  "isActive": false,
  "joinedAt": 1778675075758,
  "tmuxPaneId": "",
  "cwd": "",
  "subscriptions": []
}
```

Adjust `joinedAt` to a sensible epoch-ms (any value works; the harness doesn't validate). Save the file. Your next `SendMessage(to="apex-lead-ghost", ...)` call will succeed — the harness re-reads `members[]` on demand.

Then proceed to "Cold-start path" Step 2.

## Required apex-side prep (for completeness — usually already done)

If apex doesn't already have `fr-lead-ghost` registered:

1. Apex must add an `fr-lead-ghost` entry to `roster.json` (apex-research repo). Minimal shape:
   ```json
   { "name": "fr-lead-ghost", "agentType": "ghost", "backendType": "ssh-bridge" }
   ```
2. Apex must run TeamCreate (or use the mid-session-add pattern above on their side) so `fr-lead-ghost` lands in their runtime `members[]`.
3. No daemon on apex side — your daemon does both directions via SSH from your host.

Coordinate with apex's team-lead via existing channels (the previous PoC's `ghost-chat.py` is one option; alternatively any user-mediated relay).

## Validation tests

Run these in sequence after Step 7 above completes:

### Test 1 — outbound (you → apex)

Send a message via `SendMessage(to="apex-lead-ghost", message="test outbound", summary="outbound smoke")`. Within ~3 seconds:

- Daemon log gains an `outbound: forwarded -> apex-research:team-lead` line.
- Local `~/.claude/teams/framework-research/inboxes/apex-lead-ghost.json` shows the entry with `read: true`.
- (Optional independent verification) SSH to apex and `tail -c 800 ~/.claude/teams/apex-research/inboxes/team-lead.json` — the entry should be there with `from: fr-lead-ghost`.

### Test 2 — inbound (apex → you)

Ask Schliemann (apex's team-lead) to send you a ping via his `SendMessage(to="fr-lead-ghost", ...)`. Within ~3 seconds:

- Daemon log gains an `inbound: received <- apex-research:fr-lead-ghost` line.
- The message surfaces in your Claude conversation as `<teammate-message teammate_id="apex-lead-ghost">`. The `from` field has been rewritten to `apex-lead-ghost` (your local label for apex's lead).

### Test 3 — clean shutdown

```bash
cd teams/framework-research/poc/ghost-bridge
./stop-ghost-bridge.sh
# Expected on Linux: "ghost-bridge stopped cleanly." + PID file removed.
# Expected on Windows-Git-Bash: see caveats section below.
```

## Caveats

### Windows-Git-Bash local-dev frictions (control plane only, data plane is correct)

1. **Win32 vs POSIX PID mismatch.** Python's `os.getpid()` returns Win32 PID; MSYS `kill -0` uses POSIX PID. The daemon writes its Win32 PID to `ghost-bridge.pid`; `stop-ghost-bridge.sh` checks via MSYS → "not alive" → removes PID file as stale → daemon keeps running. **Workaround:**
   ```bash
   ps -ef | grep '[g]host-bridge.py' | awk '{print $2}' | xargs -r kill -TERM
   ```

2. **SIGTERM via MSYS doesn't run Python's `finally`.** Process IS terminated; the graceful-shutdown log line is missing. Cosmetic only — data plane unaffected.

Both are absent on Linux/Ubuntu (the framework's target deploy substrate). See `README.md § Windows local-dev caveats` for details.

### Acknowledged design limitations (v1, by design)

- No supervisor — daemon dies → all comms stop until restart.
- Race on FR-local read-flag flip (no fcntl on FR side). Rare in practice.
- Polling latency: up to 2s reply-arrival.
- Sender-rewrite drops original sender — `from` is always rewritten to the local-side ghost label.
- Single-pair coded — config supports a list; v1 implementation only uses `pairs[0]`.

Full list in `SPEC.md § Known limitations`.

## Substrate properties this exercises

Two empirical properties of the Claude Code harness that this MVP is built on:

1. **Inbox-file-write as wake mechanism** — writing to a JSON inbox file the harness watches wakes the recipient agent. Validated S31 cross-host PoC.
2. **Inbox-file-write as registration mechanism** — plain JSON edits to runtime `config.json`'s `members[]` array are honored by the harness on demand, without restart or special API. Validated S33, n=2 (FR-Windows + apex-Linux).

Both reduce to: *the right JSON file edit, in the right place, is the substrate primitive that does the work.*

## Citations

- `teams/framework-research/poc/ghost-bridge/SPEC.md` — full design spec
- `teams/framework-research/poc/ghost-bridge/README.md` — operational + caveats
- `teams/framework-research/poc/ghost-member-cli/ghost-chat.py` — S31 PoC, source of substrate primitives (APPEND_INBOX_SCRIPT, FETCH_AND_MARK_READ_SCRIPT)
- `teams/framework-research/wiki/references/inbox-file-write-as-wake-mechanism.md` — wake-substrate property reference
- (Forthcoming) `teams/framework-research/wiki/references/inbox-file-write-as-registration-mechanism.md` — registration-substrate property reference, canonicalized by Cal during S33

---

(*FR:Aen*)
