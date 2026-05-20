# Operations Log — 2026-05 (*FR:Hopper*)

Append-only operations log per `teams/framework-research/prompts/hopper.md` (Provenance — Role-of-Record section). Each entry has all 8 required fields. No edits to prior entries; corrections go as new entries referencing the original by timestamp.

---

## 2026-05-20T17:09+03:00 — apex-research authorized_keys Phase 1 prep (ABORTED MID-EXECUTION)

**timestamp** — 2026-05-20T17:09+03:00 (dispatch landed), through 2026-05-20T17:41+03:00 (HALT per PO direction), close-out written 2026-05-20T17:43+03:00.

**tasker** — Brunel (pair-loop with Aen for major escalation gates; PO direction relayed via Aen at 17:37).

**dispatch summary** — apex-research authorized_keys Phase 1 prep — staged SSH_PUBLIC_KEY_2 in host `.env`. ABORTED MID-EXECUTION after substrate-truth probes revealed degraded substrate state (no operational `.env`; container is a 2026-04-29 fresh-clone survivor; recreate today = full SSH lockout). PO direction: stop and re-evaluate.

**tier classification + sanction status** — by step:

- **P1.1** = Tier R (default-permitted)
  - First attempt FAILED at hard-gate: dispatched filter regex `michelek\s*$` matched neither key in live `/home/ai-teams/.ssh/authorized_keys` (both keys had different comment shapes from the `.env.example:13` template the regex anchored on). Surfaced to Brunel at 17:12.
  - Brunel **amendment-1** at 17:13 — replaced filter with positive-select: `$_ -match 'aleksandr' -and $_ -notmatch 'mihkel' -and $_.Trim() -ne ''`. Quoted verbatim:
    > "Positive-select on `aleksandr` token captures exactly LINE 2. Belt-and-suspenders `-notmatch 'mihkel'` defends against a hypothetical future where PO's comment ever contains `aleksandr`."
  - Second attempt PASSED. $alekKey captured (length 114).
- **P1.2a** (substrate locate) = Tier R (default-permitted)
  - `find` probe returned 2 candidate compose-dirs. Surfaced to Brunel at 17:16 per dispatch's "2+ matches → surface; do not guess."
  - Brunel **amendment-2** at 17:18 → 17:19 — Option B (docker inspect compose-project-working-dir label). Original key had underscore typo; corrected to dot-form `com.docker.compose.project.working_dir` at 17:19.
  - Within-dispatch-agency JSON-labels dump (Tier R, allowed scope) disambiguated authoritatively: $COMPOSE_DIR = `/home/dev/github/apex-migration-research`. P1.2a CLEARED.
- **P1.2b** (.env writability) = Tier R (default-permitted)
  - Probe revealed `.env` does NOT exist at $COMPOSE_DIR. Substrate disagrees with dispatch's fix-premise. Surfaced to Brunel at 17:23.
- **P1.2c** (3-probe diagnostic batch) = Tier R (Aen-sanctioned via Brunel **amendment-3** at 17:27)
  - Three probes via base64-transit: container Config.Env grep, docker compose config rendered output, filesystem env-file search.
  - Revealed degraded substrate state. **H5 hypothesis** (stable-state container survivor from pre-2026-04-29 fresh-clone) confirmed.
- **P1.2d** (backup .env read) = Tier R (Aen-sanctioned with verbatim hard-gate at 17:34)
  - Aen sanction verbatim: *"The backup `.env` is information, not destiny — we don't restore from it without PO direction."*
  - Pure `cat` read, no mutations. Secret-redaction discipline applied: mask all-but-first-4 chars for credential lines, verbatim for SSH_PUBLIC_KEY* (public-by-definition).
  - Brunel retroactively rescinded the P1.2d sanction at 17:41 along with the rest of Phase 1, but execution was clean and audit-logged before rescission. No undo needed.

**No Tier M or D ever executed.** Phase 2 sanction package (r3) formally rescinded by Aen at 17:34 pending re-architecture.

**deployed-artifacts-read declaration** — paths read at session start (pre-execution discipline, FR-design layer):

- `designs/deployed/apex-research/container/entrypoint-apex.sh:166-196` (Step 7 SSH keys block — env-var enumeration loop + install for both `michelek` and `ai-teams` users + sshd start on port 2222 + else-warning branch)
- `designs/deployed/apex-research/container/docker-compose.yml:47-60` (environment block, includes SSH_PUBLIC_KEY and SSH_PUBLIC_KEY_2 — but NOT SLOT 3, which the operational substrate has; design-vs-operational drift)
- `designs/deployed/apex-research/container/.env.example:13` (template stub `SSH_PUBLIC_KEY="ssh-ed25519 AAAA... michelek"` — load-bearing for P1.1's amended-filter analysis; the `michelek` token in the template did not survive into operational PO key comment)

In-execution substrate-truth reads (Layer 2 + Layer 3, surfaced via Tier R probes):

- Container Config.Env via `docker inspect apex-research --format '{{range .Config.Env}}{{println .}}{{end}}' | grep '^SSH_PUBLIC_KEY'` (Layer 3)
- Container labels JSON dump via `docker inspect apex-research --format '{{json .Config.Labels}}'` (Layer 3)
- Operational compose-yml rendered via `docker compose config 2>&1 | grep -E 'env_file|SSH_PUBLIC_KEY' -A 1` (Layer 2)
- Filesystem env-file search via `ls -la` + `find -maxdepth 4` (Layer 2)
- Backup `.env` content via `cat '/home/dev/github/apex-migration-research.pre-fresh-clone-2026-04-29/.env'` (Layer 2 archival)

**commands executed** — verbatim, all Tier R:

- P1.1 (first attempt, FAILED at hard-gate):
  - `ssh -i $env:USERPROFILE\.ssh\id_ed25519_apex -p 2222 -o StrictHostKeyChecking=accept-new -o BatchMode=yes ai-teams@100.96.54.170 "cat /home/ai-teams/.ssh/authorized_keys"`
- P1.1 (second attempt with Brunel amendment-1, PASSED):
  - same ssh invocation; filter changed to `Where-Object { $_ -match 'aleksandr' -and $_ -notmatch 'mihkel' -and $_.Trim() -ne '' }`
- P1.2 locate-probe:
  - `ssh -T -o StrictHostKeyChecking=accept-new -o BatchMode=yes dev@100.96.54.170 'ls -la ~/apex-research/ 2>/dev/null || ls -la ~/docker/apex-research/ 2>/dev/null || ls -la ~/ai-teams/apex-research/ 2>/dev/null; echo "---FIND---"; find ~/ -maxdepth 4 -name docker-compose.yml -path "*apex*" 2>/dev/null'`
- P1.2a label probe (with Brunel amendment-2 dot-form key, via base64-transit):
  - remote literal: `docker inspect apex-research --format '{{ index .Config.Labels "com.docker.compose.project.working_dir" }}' 2>/dev/null`
  - transport: `ssh -T dev@100.96.54.170 "echo '<b64>' | base64 -d | bash"`
- P1.2a within-dispatch-agency JSON-labels dump:
  - remote literal: `docker inspect apex-research --format "{{json .Config.Labels}}" 2>&1`
- P1.2b .env writability probe:
  - `ssh -T dev@100.96.54.170 "ls -la '/home/dev/github/apex-migration-research/.env' 2>&1; echo WHOAMI; whoami; id"`
- P1.2c Probe 1 (container Config.Env):
  - remote literal: `docker inspect apex-research --format '{{range .Config.Env}}{{println .}}{{end}}' | grep '^SSH_PUBLIC_KEY'`
- P1.2c Probe 2 (docker compose config):
  - remote literal: `cd /home/dev/github/apex-migration-research && docker compose config 2>&1 | grep -E 'env_file|SSH_PUBLIC_KEY' -A 1`
- P1.2c Probe 3 (filesystem env-file search):
  - remote literal: `ls -la /home/dev/github/apex-migration-research/.env* /home/dev/github/apex-migration-research/*.env 2>/dev/null; echo '---LSDONE---'; find /home/dev/github/apex-migration-research -maxdepth 2 -name '.env*' 2>/dev/null; echo '---FIND1DONE---'; find ~/ -maxdepth 3 -name '.env' -path '*apex*' 2>/dev/null`
- P1.2d backup .env read (Aen-sanctioned):
  - remote literal: `cat '/home/dev/github/apex-migration-research.pre-fresh-clone-2026-04-29/.env'`

All ssh exit codes 0. No mutations to RC bare-metal host or apex-research container at any point in the dispatch.

**outputs** — relevant excerpts (full output in surface-back chain to Brunel):

- **P1.1 (second attempt):** $alekKey captured = `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBPYh4HpFbc/ftGYS6NndGaFk9Oc3C+IO8+cuv1i4GOb ghost-bridge@aleksandr-2026-05-15` (length 114). Belt-and-suspenders both passed (`-match aleksandr`=True, `-notmatch mihkel`=True). Tempfile round-trip verified byte-equal.
- **P1.2 locate:** 2 hits — `/home/dev/github/apex-migration-research/docker-compose.yml` (operational) and `/home/dev/github/apex-migration-research.pre-fresh-clone-2026-04-29/docker-compose.yml` (pre-fresh-clone backup). Surface-back at 17:16.
- **P1.2a label probe (corrected key):** `/home/dev/github/apex-migration-research` — matches operational hit. **$COMPOSE_DIR = `/home/dev/github/apex-migration-research`** canonical.
- **P1.2a JSON-labels dump:** confirmed `com.docker.compose.project.working_dir` = `/home/dev/github/apex-migration-research` and `com.docker.compose.project.config_files` = `/home/dev/github/apex-migration-research/docker-compose.yml`. Compose version label `5.1.0` (v2 dot-form label namespace).
- **P1.2b .env writability:** `ls: cannot access '/home/dev/github/apex-migration-research/.env': No such file or directory`. Host user = `dev` (uid 1000, in `docker` + `sudo` groups). Substrate-access shape correct; the file just doesn't exist where the dispatch assumed.
- **P1.2c Probe 1:** `SSH_PUBLIC_KEY_2=ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKR5R4Ob4zeW4H1p8rhjYajOa+mzqyjITzB6RmY4iBp/ mihkel.putrinsh@evr.ee apex-research` (PO's key in SLOT 2) and `SSH_PUBLIC_KEY=` (slot 1 empty). Aleksandr's key absent — confirms it was injected via docker exec onto the ephemeral container layer.
- **P1.2c Probe 2:** All three slots resolve empty: `SSH_PUBLIC_KEY: ""`, `SSH_PUBLIC_KEY_2: ""`, `SSH_PUBLIC_KEY_3: ""`. No `env_file:` directive. Operational compose-yml exposes a SLOT 3 not present in FR-design `docker-compose.yml:53-54`.
- **P1.2c Probe 3:** Only `.env.example` (template) at $COMPOSE_DIR; no `.env`. Backup `.env` at sibling `.pre-fresh-clone-2026-04-29/.env` (frozen 2026-04-29). Unrelated `.env` at `vjs_apex_apps/.env` (separate repo).
- **P1.2d backup .env content** (secrets first-4-chars masked; pubkeys verbatim):

  ```
  GITHUB_TOKEN=gho_<REDACTED 36 chars>
  ANTHROPIC_API_KEY=
  SSH_PUBLIC_KEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKR5R4Ob4zeW4H1p8rhjYajOa+mzqyjITzB6RmY4iBp/ mihkel.putrinsh@evr.ee apex-research"
  ATLASSIAN_EMAIL="mihkel.putrinsh@evr.ee"
  ATLASSIAN_API_TOKEN="ATAT<REDACTED 188 chars>"
  ATLASSIAN_BASE_URL="https://eestiraudtee.atlassian.net"
  TUNNEL_TOKEN=eyJh<REDACTED 213 chars>
  SSH_PUBLIC_KEY_3=ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIO2fgnCIWJjcNpgo/rjGmF5e0fr35qupLHAFk57qU6tB rc-connect
  ```

**Key cross-walk contradictions across the substrate-truth layers:**

- Backup `.env` has PO's key in **SLOT 1** (value-quoted); running container has PO's key in **SLOT 2** (Config.Env). Slot-migration happened between backup-creation (≤2026-04-29) and current container-create. PO via Aen 17:37 said "I don't remember" for Q1 (fresh-clone without `.env` restore) — likely oversight. Q3' (when/why the slot migration happened) was NOT pursued; PO directed halt before re-engaging.
- Operational `docker-compose.yml` has SLOT 3; FR-design `docker-compose.yml:47-60` has only SLOTS 1+2. PO via Aen 17:37 said SLOT 3 is "reserved for future key, never populated." Backup `.env` (frozen 2026-04-29) shows `SSH_PUBLIC_KEY_3=...rc-connect` — apparent contradiction with PO's recollection. Possibly the rc-connect entry was experimental and abandoned, or PO doesn't recall it. **Documented per Brunel close-out instruction; not adjudicating.**
- Multi-system failure surface: backup `.env` has GITHUB_TOKEN, ATLASSIAN_API_TOKEN, TUNNEL_TOKEN. These are absent from current operational state. A `docker compose up --force-recreate` today would wipe all three alongside the SSH keys — incident surface is broader than SSH-lockout, encompasses GitHub auth + Atlassian auth + Cloudflare tunnel auth.

**outcome** — **aborted-mid-execution** (canonical enum value per `hopper.md` prompt `outcome` field). Reason: PO direction (via Aen 17:37) after substrate-truth probes surfaced dispatch-premise-disagreement + degraded substrate state. No state mutations attempted at any point in the dispatch. Container substrate unchanged from pre-dispatch state (degraded-but-stable).

---

### Tasker-confirmed prevention framing

Per Brunel 17:33 + 17:39 and Aen-confirmed: this dispatch's hard-gate discipline prevented a **multi-system failure incident** (not just SSH-lockout). The r3 Phase-2 recreate plan, if executed against current substrate state, would have:

- Wiped all SSH_PUBLIC_KEY* slots in container Config.Env (compose config resolves them empty) → no `authorized_keys` written by entrypoint Step 7 → sshd refuses all logins → full SSH lockout for both PO and Aleksandr.
- Wiped GITHUB_TOKEN, ANTHROPIC_API_KEY, ATLASSIAN_API_TOKEN, ATLASSIAN_BASE_URL, ATLASSIAN_EMAIL, TUNNEL_TOKEN → broke GitHub clone-via-token, Anthropic CLI auth, Atlassian/Jira-MCP integration, Cloudflare tunnel.

**Credit attribution per Hopper-self-flag adopted by Brunel** (17:35): the prevention is structural, not personal — the dispatch design (sequential hard-gates between every step with explicit pass criteria + surface-back templates) and Tier-D-no-execute-without-full-sanction discipline did the work; Hopper's role was enforcement. The dispatch shape itself is the reusable pattern for other substrates.

### Sub-shape E manifestation — three-layer substrate-truth divergence

Per joint Brunel + Hopper synthesis (Brunel architectural framing, Hopper operator-defense pattern, Cal-Protocol-A submission planned for session wrap):

- **Layer 1 — FR design-as-shipped** (`designs/deployed/<team>/container/*` in this repo): canonical for design lineage.
- **Layer 2 — Consumer team operational copy** (e.g., `/home/dev/github/apex-migration-research/` on the substrate host): canonical for what compose actually reads at next `docker compose up`.
- **Layer 3 — Running container state** (Config.Env, mounted volumes, in-process state): canonical for what's serving traffic right now.

This dispatch concretely materialized BOTH drift surfaces simultaneously:

- **Layer-1-vs-Layer-2 drift:** FR design has 2 SSH_PUBLIC_KEY slots; operational compose-yml has 3 slots. Consumer team (apex) amended their copy without FR-side visibility.
- **Layer-2-vs-Layer-3 drift:** Current operational compose-yml resolves all 3 slots to empty; running container has SLOT 2 populated. Runtime carries state from before whatever event emptied the operational copy.

**Operator-defense pattern (Hopper-authored, joint cross-link to Brunel architectural framing):** when first-dispatching against any substrate, run a Tier R probe-suite that surfaces all three layers and reconcile before committing to a fix-shape. Single-layer reads (FR-design-only, as my current `hopper.md` Diagnostic Discipline prescribes) catch single-layer drift but miss cross-layer drift. **Surface as Hopper-Amendment-4 candidate to Celes at session wrap.**

### Cross-in-transit count (session artifact)

Final count: **n=11 instances** of tasker↔operator message-crossings in transit through this session. Framing per Brunel 17:33 acknowledged by Hopper 17:35: candidate is **cadence-driven** (high-frequency tasker↔operator pair-loop on async inbox transport) vs **substrate-driven** (Windows-session-local per `feedback_no_windows_substrate_findings.md`). Disambiguation requires Linux-substrate replication at similar cadence; pending until that observation lands. Filed as parallel candidate to the discriminator-anchoring pattern.

### Cal-Protocol-A submission planning (session-wrap)

Two joint-authorship submissions planned per S33+ discipline:

1. **`wiki/patterns/discriminator-anchored-on-sub-canonical-source.md`** — Brunel-attributed phrasing "discriminator anchored on a sub-canonical source." Two instances surfaced in this single dispatch:
   - P1.1 regex anchored on `.env.example:13` template stub (`michelek`), not on substrate-live state.
   - P1.2a probe label-key anchored on inferred underscore-form (`com.docker.compose.project_working_dir`), not on substrate-truth dot-form (`com.docker.compose.project.working_dir`).
   - Recovery pattern: substrate-live state is the canonical discriminator source; templates/inferences are stub-grade, not selector-grade. JSON-dump-when-single-probe-empty is the cheap Tier R diagnostic that recovers the truth in one round-trip.
2. **`wiki/patterns/three-layer-substrate-truth-discipline.md`** — joint authorship (Brunel architectural distinction + Hopper operator-defense pattern). Catalyzing incident: this ops-log entry, full surface-back chain (P1.1 amendment → P1.2a JSON-dump diagnostic → P1.2b premise-drift → P1.2c three-probe synthesis → P1.2d backup-read).

---

### Audit-trail surface (for future Hopper reference)

The full surface-back chain through this dispatch is a textbook example of the hard-gate discipline absorbing a series of substrate disagreements without any mutation reaching the substrate. Use this entry as a reference when sanction-completeness is questioned in future dispatches.

1. **P1.1 first-attempt FAIL** (regex anchored on template stub) → surface-back to Brunel + CC Aen → **amendment-1** (positive-select aleksandr) → P1.1 PASS
2. **P1.2a 2-candidate ambiguity** → surface-back with three re-dispatch options enumerated → **amendment-2** (docker inspect label probe) → 17:18 key had underscore typo → 17:19 correction to dot-form → within-dispatch-agency JSON-dump diagnostic resolved both the typo AND the candidate ambiguity in one Tier R probe → P1.2a PASS
3. **P1.2b premise-drift** (`.env` does not exist at $COMPOSE_DIR) → surface-back with three re-dispatch options → Brunel escalates to Aen at 17:25 → **amendment-3** (P1.2c three-probe diagnostic batch) sanctioned
4. **P1.2c three-probe execution** → revealed degraded substrate; H5 hypothesis (stable-state pre-fresh-clone container survivor) confirmed → multi-system failure surface acknowledged
5. **P1.2d Aen-sanctioned backup-read** with verbatim "information not destiny" hard-gate → executed clean; revealed slot-migration timeline + SLOT 3 rc-connect entry + full credential cluster in backup
6. **PO direction "stop and re-evaluate"** via Aen 17:37 → Brunel HALT close-out at 17:41 → ops-log + scratchpad written, this entry → idle

(*FR:Hopper*)
