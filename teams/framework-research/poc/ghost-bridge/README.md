# ghost-bridge — team-lead cross-host comms daemon (v1)

Sketch-grade reference implementation of `SPEC.md` (this directory). Generalizes the S31 RFC #66 PoC (`../ghost-member-cli/ghost-chat.py`) into a long-running daemon that bridges Aen (FR team-lead) <-> Schliemann (apex-research team-lead).

One pair per process (v1 scope). Outbound forwards FR-local outbox -> remote inbox via SSH; inbound fetches remote outbox -> FR-local inbox via SSH. Each direction runs once per polling cycle (default 2.0s).

## Files

| File | Purpose |
|---|---|
| `SPEC.md` | The full design. Read before editing code. |
| `ghost-bridge.py` | The daemon. Stdlib only, Python 3.7+. |
| `ghost-bridge.config.example.json` | Config template. Copy and edit. |
| `ghost-bridge.config.json` | Live config. **gitignored.** |
| `start-ghost-bridge.sh` | Launch in background. Refuses if alive PID file exists. |
| `stop-ghost-bridge.sh` | SIGTERM, 5s grace, SIGKILL fallback. |
| `ghost-bridge.pid` | Runtime PID file. **gitignored.** |
| `ghost-bridge.log` | Runtime log file. **gitignored.** Truncated at each daemon start. |

## Prerequisites

1. **`~/bin/rc-deployments.json`** with an `apex-research` deployment entry. Same registry the PoC uses.
2. **SSH key + auth** pre-configured for the user invoking the daemon. The PoC's existing setup is reused.
3. **Python 3.7+** on the FR host (the daemon's host). `python3` preferred; `python` accepted as fallback.
4. **Apex-side prerequisites** per SPEC § Apex-side prerequisites — `fr-lead-ghost` member entry in apex's `roster.json`, then a fresh apex session so apex's harness creates `inboxes/fr-lead-ghost.json`.
5. **FR-side roster entry** for `apex-lead-ghost` per SPEC § Components. Restart FR's session so the harness creates `$HOME/.claude/teams/framework-research/inboxes/apex-lead-ghost.json`.

## Install

Nothing to install. The daemon uses only the Python standard library and standard SSH from the system PATH.

```bash
cd teams/framework-research/poc/ghost-bridge
cp ghost-bridge.config.example.json ghost-bridge.config.json
# Edit ghost-bridge.config.json if your pair/aliases differ from the defaults
chmod +x start-ghost-bridge.sh stop-ghost-bridge.sh
```

## Start / stop

```bash
./start-ghost-bridge.sh   # launches in background, writes ghost-bridge.pid
./stop-ghost-bridge.sh    # SIGTERM, waits 5s, SIGKILL if needed
```

The daemon truncates `ghost-bridge.log` at every start. Tail it during the session:

```bash
tail -f ghost-bridge.log
```

## Test path (team-lead validation)

Run these in order against a real apex session that has `fr-lead-ghost` set up. Substitute your actual paths if not running from the canonical FR repo root.

### Smoke: daemon survives idle

```bash
cd teams/framework-research/poc/ghost-bridge
cp ghost-bridge.config.example.json ghost-bridge.config.json
./start-ghost-bridge.sh
sleep 60
# Daemon should still be alive:
kill -0 "$(cat ghost-bridge.pid)" && echo "ALIVE"
tail -20 ghost-bridge.log
# Should see startup line + no errors.
./stop-ghost-bridge.sh
```

### Outbound: FR -> apex

```bash
# With the daemon running:
./start-ghost-bridge.sh

# Manually drop a test message into FR-local outbox:
INBOX="$HOME/.claude/teams/framework-research/inboxes/apex-lead-ghost.json"
TS="$(python3 -c 'from datetime import datetime,timezone; n=datetime.now(timezone.utc); print(f"{n.strftime(\"%Y-%m-%dT%H:%M:%S\")}.{n.microsecond//1000:03d}Z")')"
python3 -c "
import json, os
p = os.path.expanduser('$INBOX')
arr = json.load(open(p)) if os.path.exists(p) and open(p).read().strip() else []
arr.append({'from':'team-lead','text':'ghost-bridge outbound smoke test','summary':'smoke','timestamp':'$TS','read':False})
json.dump(arr, open(p,'w'), ensure_ascii=False)
print('appended to', p)
"

# Within ~3s:
#   - the FR-local entry's `read` flag should flip to true
#   - apex's ~/.claude/teams/apex-research/inboxes/team-lead.json gains an entry
#     with from=fr-lead-ghost, read:false
# Check FR-local:
python3 -c "import json; print(json.load(open('$INBOX'))[-1])"
# Check apex (via ssh):
ssh -i ~/.ssh/id_ed25519_apex -p 2222 ai-teams@<apex-host> \
  "tail -c 600 ~/.claude/teams/apex-research/inboxes/team-lead.json"

tail -20 ghost-bridge.log
./stop-ghost-bridge.sh
```

### Inbound: apex -> FR

```bash
./start-ghost-bridge.sh

# Drop a test message into apex's outbox (the FR-bound one):
ssh -i ~/.ssh/id_ed25519_apex -p 2222 ai-teams@<apex-host> "
python3 - <<'PY'
import json, os, fcntl
p = os.path.expanduser('~/.claude/teams/apex-research/inboxes/fr-lead-ghost.json')
fd = os.open(p, os.O_RDWR|os.O_CREAT, 0o644)
fcntl.flock(fd, fcntl.LOCK_EX)
raw = os.read(fd, 1<<24).decode('utf-8') or '[]'
arr = json.loads(raw) if raw.strip() else []
arr.append({'from':'team-lead','text':'ghost-bridge inbound smoke','summary':'smoke','timestamp':'2026-05-14T00:00:00.000Z','read':False})
os.lseek(fd,0,0); os.ftruncate(fd,0); os.write(fd, json.dumps(arr, ensure_ascii=False).encode())
fcntl.flock(fd, fcntl.LOCK_UN); os.close(fd)
PY
"

# Within ~3s:
#   - apex-side flag flips to read:true
#   - FR-local ~/.claude/teams/framework-research/inboxes/team-lead.json gains
#     an entry with from=apex-lead-ghost, read:false
python3 -c "import json,os; p=os.path.expanduser('~/.claude/teams/framework-research/inboxes/team-lead.json'); print(json.load(open(p))[-1])"

tail -20 ghost-bridge.log
./stop-ghost-bridge.sh
```

### Clean shutdown

```bash
./start-ghost-bridge.sh
sleep 3
./stop-ghost-bridge.sh
# Expect:
#   - "ghost-bridge stopped cleanly." from stop script
#   - ghost-bridge.pid removed
#   - tail of ghost-bridge.log shows "signal 15 received" + summary line
ls ghost-bridge.pid 2>/dev/null && echo "BAD: pid file lingered" || echo "OK: pid file gone"
```

## Reading the log

Lines are `<ISO-UTC-timestamp> [LEVEL] <msg>`. Levels: `INFO`, `WARN`, `ERROR`.

Useful greps:

```bash
grep "forwarded ->" ghost-bridge.log   # outbound deliveries
grep "received <-"  ghost-bridge.log   # inbound deliveries
grep -E "ERROR|WARN" ghost-bridge.log  # all anomalies
```

## Known limitations (v1)

Inherited from `SPEC.md § Known limitations`:

1. **No supervisor.** Daemon dies -> all comms stop until restart.
2. **Race on FR-local read-flag flip.** No fcntl on FR-side. Rare but possible.
3. **No backpressure.** Unbounded growth if apex unreachable for long.
4. **Polling latency.** Up to 2s reply-arrival latency.
5. **Sender-rewrite drops original sender.** By design for v1.
6. **Single-pair coded.** Config supports list; code handles `pairs[0]`. Additional pairs logged as a WARN at startup and ignored.

## Architecture notes (cross-reference to SPEC)

- Reuses `APPEND_INBOX_SCRIPT` and `FETCH_AND_MARK_READ_SCRIPT` from `../ghost-member-cli/ghost-chat.py` verbatim — SF-4 substrate-validated.
- No `color` field on forwarded envelopes (SF-3 contract).
- FR-local file ops use plain `open()` / `json.load`/`json.dump` (SPEC § Substrate invariants accepts Known limitation #2 for v1).
- Path resolution uses `pathlib.Path.home()` — works on Windows + POSIX.
- Empty / missing remote inbox file: skip with one log line, retry next cycle (SPEC § Open decisions #2).
- Log rotation: truncate at every daemon start (SPEC § Open decisions #1).
- PID file location: script-adjacent (SPEC § Open decisions #3).

---

(*FR:Aen via coding-subagent*)
