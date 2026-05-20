# Hopper — scratchpad (*FR:Hopper*)

First-spawn 2026-05-20. Maintain under 100 lines; prune stale entries; promote pattern-grade to Callimachus via Protocol A.

## Substrate facts

> Post-dispatch correction (Aen 2026-05-20 17:58): the 2026-05-20 dispatch operated against **RC-server (100.96.54.170)**, NOT the canonical production apex deployment. The substrate facts below are scoped to RC-server. Production apex lives on a different host — see `[LEARNED — substrate, apex-PROD-LLM]` entry. The Brunel-side registry-entry-choice-from-first-match error that caused this is now catalogued as Sub-shape F.

[LEARNED — substrate, apex-PROD-LLM] Production apex is at `michelek@10.100.136.162:22` (`~/bin/rc-deployments.json` entry `num:"9"` — "PROD-LLM"), key `id_ed25519_apex`. **NOT TOUCHED in 2026-05-20 dispatch.** State unknown. Future dispatches against the apex **production** substrate must operate against this host, not RC-server.

[LEARNED — substrate, RC-server (100.96.54.170)] Host-user SSH: `ssh -T dev@100.96.54.170` (default key `~/.ssh/id_ed25519`, port 22). Source-of-truth: `~/bin/rc-deployments.json` entry `num:"1"`. `-T` is load-bearing for non-interactive ops (suppresses TTY allocation, keeps stdout clean for parsing). **This is RC-server, NOT production apex — see apex-PROD-LLM entry above.**

[LEARNED — substrate, RC-server (100.96.54.170)] Container-user SSH (for the RC-server-hosted apex container, not production): `ssh -i ~/.ssh/id_ed25519_apex -p 2222 ai-teams@100.96.54.170`. Source-of-truth: `~/bin/rc-deployments.json` entry `num:"2"`.

[LEARNED — substrate, RC-server (100.96.54.170)] `$COMPOSE_DIR` for the RC-server-hosted apex container = `/home/dev/github/apex-migration-research`. Discovered via P1.2 find-probe (Tier R); confirmed authoritatively via `docker inspect` label `com.docker.compose.project.working_dir`, NOT via FR's `designs/deployed/apex-research/container/` path (that's design-as-shipped, not operational substrate).

[LEARNED — substrate, RC-server (100.96.54.170)] Operational compose-yml diverges from FR design on RC-server's apex copy: substrate `docker-compose.yml` exposes `SSH_PUBLIC_KEY_3` slot; FR's `designs/deployed/apex-research/container/docker-compose.yml:47-60` shows only SLOTS 1+2. Per PO via Aen 2026-05-20 17:37: SLOT 3 reserved for future key, never populated — though backup `.env` (frozen 2026-04-29) had `SSH_PUBLIC_KEY_3=...rc-connect` (apparent contradiction; documented in ops-log, not adjudicated). Whether this drift also exists on the PROD-LLM apex deployment is UNKNOWN.

[LEARNED — substrate, RC-server (100.96.54.170)] Host user `dev`: uid 1000, gid 1000, in groups `sudo + docker + ollama + plugdev + ...`. `docker inspect` and `docker compose config` work unprivileged.

[GOTCHA — substrate, RC-server (100.96.54.170)] AS OF 2026-05-20: the RC-server-hosted apex substrate is in **degraded state**. No `.env` at `$COMPOSE_DIR`. Container survives on Config.Env baked in pre-2026-04-29-fresh-clone. ANY container recreate today (`docker compose up --force-recreate`) = full SSH lockout (all `SSH_PUBLIC_KEY*` slots resolve empty at next compose-up) + all credentials lost (GITHUB_TOKEN, ATLASSIAN_API_TOKEN, TUNNEL_TOKEN empty too). **Phase 2 sanction package r3 is UNSAFE under current RC-server substrate state — formally rescinded by Aen 2026-05-20 17:34. Do NOT execute against RC-server. Future Phase 2 requires `.env` reconstruction prerequisite + new sanction package. Production apex (PROD-LLM) was NOT touched and its state is unknown.**

## Generalizable patterns

[LEARNED — discipline, substrate-selection] On first-ever dispatch against any substrate-name (e.g., "apex"), do not assume the first registry entry matching the name is canonical production. Multiple entries may exist with overlapping name-roots (RC-server hosts apex, but production apex is on PROD-LLM). Discipline: if dispatch directs you to a specific host AND your registry-read reveals additional entries matching the substrate-name root, surface the ambiguity as a hard-gate before executing. The 2026-05-20 dispatch retroactively earns Sub-shape F into the pattern catalog: **registry-entry-choice-from-first-match** (Brunel-side authoring; my discipline-side defense). Cross-link: ops-log-2026-05.md entry for the 2026-05-20 dispatch; Aen 17:58 informational amendment.

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
