# Hopper — scratchpad (*FR:Hopper*)

First-spawn 2026-05-20. Maintain under 100 lines; prune stale entries; promote pattern-grade to Callimachus via Protocol A.

## Substrate facts — apex-research

[LEARNED — substrate, apex-research] Host-user SSH: `ssh -T dev@100.96.54.170` (default key `~/.ssh/id_ed25519`, port 22). Source-of-truth: `~/bin/rc-deployments.json` entry `num:"1"`. `-T` is load-bearing for non-interactive ops (suppresses TTY allocation, keeps stdout clean for parsing).

[LEARNED — substrate, apex-research] Container-user SSH: `ssh -i ~/.ssh/id_ed25519_apex -p 2222 ai-teams@100.96.54.170`. Source-of-truth: `~/bin/rc-deployments.json` entry `num:"2"`. Same physical host as the host-user SSH (`num:"1"`); two SSH shapes for host vs container access.

[LEARNED — substrate, apex-research] `$COMPOSE_DIR` = `/home/dev/github/apex-migration-research`. Discovered via P1.2 find-probe (Tier R); confirmed authoritatively via `docker inspect` label `com.docker.compose.project.working_dir`, NOT via FR's `designs/deployed/apex-research/container/` path (that's design-as-shipped, not operational substrate).

[LEARNED — substrate, apex-research] Operational compose-yml diverges from FR design: substrate `docker-compose.yml` exposes `SSH_PUBLIC_KEY_3` slot; FR's `designs/deployed/apex-research/container/docker-compose.yml:47-60` shows only SLOTS 1+2. Per PO via Aen 2026-05-20 17:37: SLOT 3 reserved for future key, never populated — though backup `.env` (frozen 2026-04-29) had `SSH_PUBLIC_KEY_3=...rc-connect` (apparent contradiction; documented in ops-log, not adjudicated).

[LEARNED — substrate, apex-research] Host user `dev`: uid 1000, gid 1000, in groups `sudo + docker + ollama + plugdev + ...`. `docker inspect` and `docker compose config` work unprivileged.

[LEARNED — substrate, apex-research] AS OF 2026-05-21T09:18 (post-Phase-2 recreate success): **original PO ask ACHIEVED**. Container recreated; 3 SSH keys (PO + Aleksandr + rc-connect) installed by entrypoint Step 7 from canonical `.env`; KEY_COUNT=3 confirmed in entrypoint logs; sshd on port 2222 accepting all 3 keys. Operational compose-yml amended (P4.05) to declare `GH_TOKEN=${GH_TOKEN:-}` in apex-research env block — GH_TOKEN preserved through recreate per PO Option B direction. All declared tokens in Config.Env (GITHUB_TOKEN, GH_TOKEN, ATLASSIAN_*, ANTHROPIC_API_KEY-empty). All 3 named volumes preserved across recreate. Container is recreate-safe — next maintenance recreate installs same keys cleanly from same .env. Backup `.env` at $BACKUP_DIR + compose-yml backup `docker-compose.yml.bak.20260521-091347` both untouched as rollback artifacts. **The "any recreate = multi-system credential loss" gotcha is fully RESOLVED post-Phase-2.**

[GOTCHA — substrate, apex-research, historical] Pre-2026-05-20T19:17 the substrate was in degraded state: no `.env` at `$COMPOSE_DIR`; container survived on Config.Env baked pre-2026-04-29-fresh-clone; any recreate = full SSH lockout + credential cluster loss (GITHUB_TOKEN, ATLASSIAN_API_TOKEN, TUNNEL_TOKEN). Original r3 Phase-2 sanction was rescinded by Aen 17:34 due to this. **Fully RESOLVED 2026-05-21T09:18 post-Phase-2 recreate** — substrate now in canonical state with .env + amended compose-yml + recreate-safe Config.Env. Historical reference for the dispatch-arc audit trail; do NOT treat as current.

[LEARNED — substrate, apex-research] Operational compose-yml's apex-research env block (post-2026-05-21T09:18 P4.05 amendment) declares 15 vars: HOME, REPO_URL, SOURCE_REPO_URL, GITHUB_TOKEN, **GH_TOKEN (added P4.05)**, ANTHROPIC_API_KEY, SSH_PUBLIC_KEY, SSH_PUBLIC_KEY_2, SSH_PUBLIC_KEY_3, NODE_EXTRA_CA_CERTS, CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS, TEAM_NAME, ATLASSIAN_EMAIL, ATLASSIAN_API_TOKEN, ATLASSIAN_BASE_URL. FR-design `designs/deployed/apex-research/container/docker-compose.yml:47-60` has fewer slots (no SLOT 3, no GH_TOKEN). Sub-shape E drift still exists between Layer 1 (FR design) and Layer 2 (operational); future FR-design update may want to absorb the operational SLOT 3 + GH_TOKEN additions.

[GOTCHA — substrate, apex-research] Backup `.env` at `$BACKUP_DIR/.env` (frozen 2026-04-29) has `SSH_PUBLIC_KEY_3=ssh-ed25519 ... rc-connect` UNQUOTED (other SSH_PUBLIC_KEY entries are double-quoted; SLOT 3 is the only unquoted multi-word value). With `set -e` enabled, bash `source` of this file fails (parses `SSH_PUBLIC_KEY_3=ssh-ed25519` as assignment then tries to exec the rest). Workaround: use `grep+cut+strip-quotes` per-token instead of `source` — substrate-truthful and robust to malformed backup syntax. Pattern shape from P3.6 amendment-1.

## Generalizable patterns

[LEARNED — discipline, revert-on-invalidated-premise] Mid-conversation 2026-05-20: PO surfaced apparent wrong-host concern via Aen at 17:47-17:58 (his transient mis-read of `~/bin/rc-deployments.json`). I executed the discipline-honoring cleanup amendment when instructed by Aen — relabeled scratchpad substrate entries, appended ops-log correction at 18:05 (append-only, not in-place). PO re-checked registry himself at 18:08 and confirmed the original 17:09 substrate-target was correct. Aen instructed full revert at 18:19. **Lesson: even after discipline-honoring artifact updates (append-only ops-log + scratchpad amendments), be willing to revert when the original premise is invalidated. Discipline serves accuracy, not consistency-with-prior-correction.** The append-only ops-log preserves the full audit-trail of confusion-and-resolution (17:09 original + 18:05 first correction + 18:XX revert-correction); scratchpad reverts to pre-amendment state because scratchpad is working-memory, not append-only audit log. Sub-shape F (registry-entry-choice-from-first-match) was withdrawn from the catalog — no valid instance from this dispatch.

[LEARNED] **Within-dispatch-agency JSON-labels dump pattern:** when a single-label `docker inspect ... --format '{{index .Config.Labels "key"}}'` probe returns empty, run `docker inspect ... --format '{{json .Config.Labels}}'` as a Tier R follow-up. Disambiguates three failure modes in one round-trip: (a) label-key-missing, (b) container-not-running, (c) command-malformed. Substrate-truth-cheap, well within Tier R within-dispatch-agency scope.

[LEARNED] **Substrate-truth probe sequencing for env-source diagnosis:** container Config.Env > docker compose config (rendered output) > filesystem search (`.env*`, `*.env`, `find` in host home). Cheap-first, online-truth-first. Each layer reveals what the next can't: Config.Env shows what running container actually has; compose config shows what NEXT compose-up would resolve; filesystem search catches env-file-via-flag or alt-naming variants.

[LEARNED — generalized discipline] **On any first-ever dispatch against a new substrate**, scratchpad-capture as `[LEARNED — substrate, <team-name>]` entries the durable operational facts the probes surfaced: host-side filesystem paths, SSH connection shapes (cite `~/bin/rc-deployments.json` entry), file-ownership and writability defaults, named-volume layout, entrypoint-specific behaviors, naming conventions. On subsequent dispatches against the same substrate, READ scratchpad BEFORE probing; only re-probe if a fact looks stale or the dispatch explicitly directs.

[LEARNED — discipline gap, candidate Hopper-Amendment-4] My existing `hopper.md` Diagnostic Discipline "Read Deployed Artifacts Before Executing" section reads **Layer 1 only** (FR design-as-shipped). Insufficient for FR-shipped substrates consumer teams operationalize. **Three-layer substrate-truth discipline:** (1) FR design-as-shipped (`designs/deployed/<team>/container/*`), (2) consumer team operational copy (substrate host's compose dir), (3) running container state (Config.Env, mounted volumes). Each can drift from neighbor. First-dispatch against any substrate → run Tier R probe-suite surfacing all three; subsequent dispatches → read scratchpad first, only re-probe if stale. **Surface to Celes at session wrap as Hopper-Amendment-4, cross-link to ops-log-2026-05.md catalyzing-incident entry.**

[LEARNED] **Discriminator anchored on sub-canonical source** (Brunel-phrasing). Filter regex / lookup key derived from a template stub or inferred convention rather than substrate-live state. Two instances in this single dispatch: P1.1 regex anchored on `.env.example` template (`michelek`); P1.2a label-key anchored on inferred underscore form. Recovery: substrate-live state is the canonical discriminator source; templates/inferences are stub-grade. JSON-dump-on-empty is the cheap Tier R recovery diagnostic.

## Local-side gotchas (Windows dev workstation)

[GOTCHA — local dev] PowerShell-on-Windows quoting of nested Go-template format strings via `ssh "remote-cmd"` mangles `\"` escapes. **Workaround used through this dispatch: base64-transit** — build the literal remote command locally, base64-encode, send as `ssh "echo '<b64>' | base64 -d | bash"`. Same pattern Brunel specified for P1.4; generalizes to any remote command with shell metacharacters or nested quotes. (Pattern itself was Brunel-authored for transit safety; deployment lesson here is its second-use generality.)

[GOTCHA — local dev] PowerShell 5.x file-loaded scripts: non-ASCII characters (em-dash `—`, etc.) in `Write-Host` literal strings cause parser failure when file is UTF-8-no-BOM. **Workaround: use ASCII hyphens in script messages.** `(blank-line)` etc. for separators. This is a Windows session-local friction per `feedback_no_windows_substrate_findings.md` — do NOT file as wiki-grade.

[GOTCHA — local dev] PowerShell 5.x lexer mis-parses `[STRING-BRACKET-LITERAL]` inside `Write-Host` strings under some states. **Workaround: parentheses/no-brackets in script messages.** Same windows-substrate-friction class as the em-dash gotcha.

## In-flight dispatches

(none — Phase 1 ABORTED MID-EXECUTION per PO direction 2026-05-20 17:37; close-out written to `docs/operations-log-2026-05.md`)

## Cal-Protocol-A submissions planned (session wrap)

[CHECKPOINT] Two joint-authored Brunel + Hopper submissions to draft at session wrap:

1. **`wiki/patterns/discriminator-anchored-on-sub-canonical-source.md`** — Brunel-phrasing; two instances in this dispatch (P1.1 regex on template stub, P1.2a label-key on inferred underscore form). Recovery: substrate-live state is canonical; templates/inferences are stub-grade. JSON-dump-on-empty recovery pattern.
2. **`wiki/patterns/three-layer-substrate-truth-discipline.md`** — Brunel-authored architectural distinction (3 layers + 2 drift surfaces); Hopper-authored operator-defense pattern (Tier R 3-layer probe-suite mandatory on first-dispatch). Joint citation: this ops-log entry as catalyzing incident.

## Open observations (session-wrap surface candidates)

[DEFERRED] Cross-in-transit n=11 this session. Disambiguation: cadence-driven (tasker↔operator pair-loop density) vs substrate-driven (Windows-session-local). Filed pending Linux-substrate replication observation.

[DEFERRED] Hopper-Amendment-4 candidate (three-layer Diagnostic Discipline) — surface to Celes at session wrap, cross-linked to ops-log-2026-05.md.
