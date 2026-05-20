# Brunel scratchpad

## SESSION 34 — Apex authorized_keys dispatch (2026-05-20)

[LEARNED] **Wrong-host claim was itself wrong; original dispatch operated against the correct substrate.** PO surfaced apparent wrong-host concern at 17:43 (claiming num:9 PROD-LLM was the canonical apex); Aen relayed at 17:47; I propagated to Hopper at 17:59 without checking the registry source myself. PO re-checked his own `rc-deployments.json` at 18:08 and confirmed: num:1 (RC-server host) + num:2 (apex-research container at port 2222 with id_ed25519_apex key) ARE the canonical apex substrate — exactly what my r3 dispatch cited. The 17:43-17:47 "wrong host" framing was a transient PO mis-read; retracted 18:08. **Discipline gap on my side: relay-fidelity-mid-conversation** — accepted tasker framing as authoritative without primary-artifact (`~/bin/rc-deployments.json`) check. Symmetric to S31 Stage-2-stale-fold but applied to mid-conversation tasker statement, not relay text. Same generalization: primary-artifact-over-relay holds for tasker-framing too, not just for cross-agent relay.

[LEARNED — STRONG] **Diagnostic loop produced multi-system-failure prevention for the production apex substrate.** Substrate IS in degraded state (verified via Hopper's P1.2c three-probe batch + P1.2d backup-read): no `.env` at $COMPOSE_DIR, container is a 2026-04-29 fresh-clone survivor, `docker compose up --force-recreate` would wipe SSH keys + GITHUB_TOKEN + ATLASSIAN_API_TOKEN + TUNNEL_TOKEN simultaneously. The original ask (Aleksandr's key persistence) IS addressable on this substrate; deferred legitimately to apex team's post-maintenance window per the r3 Phase-2 plan, but Phase-2 sanction package itself must be rebuilt to address the degraded state before recreate (`.env` reconstruction prerequisite). The r3 Phase-2 recreate as drafted remains rescinded by Aen.

[LEARNED — STRONG] **Sub-shape catalog A-E surfaced this dispatch, all instances of "discriminator/premise anchored on a sub-canonical source":**
- A: template-stub-as-selector (P1.1 `michelek` regex)
- B: name-inference-as-selector (P1.2a 2-candidate ambiguity)
- C: documentation-convention-as-discriminator (P1.2a label-key underscore-vs-dot)
- D: deployed-artifacts-as-premise-source (P1.2b `.env`-presence assumption)
- E: substrate-ownership-vs-design-ownership distinction (Layer 1 FR-design / Layer 2 consumer-operational / Layer 3 runtime, two drift surfaces concretely materialized — HEADLINE)
Recovery pattern for all: cheap Tier R probe of substrate-live state beats offline inference. Cal-Protocol-A submission planned at session wrap: `wiki/patterns/discriminator-anchored-on-sub-canonical-source.md` (A+C as instances) + `wiki/patterns/three-layer-substrate-truth-discipline.md` (joint-authored with Hopper, E as headline + B+D as illustrative sub-shapes). **Earlier-proposed Sub-shape F (registry-entry-choice-from-first-match) STRUCK** per Aen 18:19 — was based on the now-disproven wrong-host premise; no real instance from this dispatch.

[LEARNED] **r3 Phase-2 sanction package RESCINDED** by Aen 17:34 + retroactively by PO 17:37. Reason: r3 Phase-2 (`docker compose up -d --force-recreate apex-research`) would have caused multi-system failure against RC-server (SSH lockout + GitHub auth + Atlassian auth + Cloudflare tunnel auth all empty). Even after substrate-selection correction, future Phase 2 against PROD-LLM apex needs new sanction package built on fresh probe-suite of THAT host's substrate-truth. The r3 package is dead; do not resurrect.

[LEARNED — joint with Hopper] **Three-layer substrate-truth model:** Layer 1 = FR design-as-shipped (`designs/deployed/<team>/container/*`) canonical for design lineage only; Layer 2 = consumer-team operational copy (e.g., `/home/dev/github/<repo>/`) canonical for compose-up resolution; Layer 3 = runtime container state (Config.Env, mounted volumes, in-process state) canonical for current-serving. Each layer can drift from its neighbor without notice. My `[LEARNED — STRONG, session 33+]` read-your-own-deployed-artifacts discipline reads Layer 1 only — INSUFFICIENT for FR-shipped substrates the consumer team operationalizes. Hopper's prompt has the same gap. Joint amendment-4 candidate (Brunel + Hopper) for Celes routing at session-end: "read all three layers on first-dispatch against a substrate." Defer until Celes online; pattern surfaced today, drafting later.

[LEARNED — provisional, n=12 this session] **Cross-in-transit pattern at high tasker↔operator cadence.** Async inbox transport with no priority lane for HALT messages accumulates message-crossings when pair-loop cadence is dense (~1-3 min per round-trip). Aen-named meta-observation: "when team-lead halts mid-pair-loop, what's the structural mechanism to interrupt in-flight dispatches at the SendMessage layer?" Three future-session candidates: (a) HALT primitive at SendMessage layer with priority delivery, (b) Brunel-side discipline pattern polling for halt-signal between every outbound, (c) acceptance that pair-loops at high cadence have inherent halt-latency = round-trip-time. Filed pending Linux-substrate replication per `feedback_no_windows_substrate_findings.md` — n=12 stays Windows-session-local class until Linux-substrate same-cadence dispatch surfaces equivalent count. Aen's own 17:58 direct-to-Hopper crossed my 17:59 relay — Aen self-noted as team-lead-side discipline gap (impatience under Brunel-recovery → short-circuit of role-of-record); my silence during tool-call recovery was the precipitating factor.

[CARRY-FORWARD — session-end Cal Protocol A submissions]
1. `wiki/patterns/discriminator-anchored-on-sub-canonical-source.md` — Brunel-authored. Sub-shapes A + C as instances. Recovery: substrate-live discriminator + JSON-dump-when-single-probe-empty.
2. `wiki/patterns/three-layer-substrate-truth-discipline.md` — JOINT (Brunel architectural distinction + Hopper operator-defense pattern). Sub-shape E as headline; B + D as illustrative. Catalyzing incident: Hopper ops-log entry 2026-05-20T17:09+03:00 at `teams/framework-research/docs/operations-log-2026-05.md`.
3. **Amendment-4 candidate (Hopper + Brunel prompts):** three-layer Diagnostic Discipline. Surface to Celes at session wrap, NOT this session. Draft text in Hopper's 17:35 message body + my 17:37 elaboration. **Amendment-5 candidate (dispatch-target-correctness probe-suite) WITHDRAWN** — solved a non-problem from this dispatch (Sub-shape F struck); revive if a real instance surfaces in a future dispatch.

[CARRY-FORWARD — Phase-1 re-architecture (post-maintenance-window)] Future-session work: when apex team approves a maintenance window, redo Phase 1 against the correct substrate (num:2 = `ai-teams@100.96.54.170:2222`) with `.env` reconstruction as the prerequisite step BEFORE any recreate. Hopper's three-probe pattern + three-layer substrate-truth reads stay as the diagnostic shape. Don't reach for the rescinded r3 package; build fresh sanction package once Phase-1 fix-shape settles around the actual substrate state (which we now know: no `.env`, container survives on pre-fresh-clone Config.Env, full credential cluster needed to reconstruct).

[GOTCHA — apex-research production substrate as of 2026-05-20] Container at `ai-teams@100.96.54.170:2222` is degraded-but-stable. No `.env` at `/home/dev/github/apex-migration-research/`. `docker compose up --force-recreate` = multi-system failure (SSH lockout + GITHUB_TOKEN + ATLASSIAN_API_TOKEN + TUNNEL_TOKEN all wiped). Pre-fresh-clone backup `.env` exists at `apex-migration-research.pre-fresh-clone-2026-04-29/.env` and contains the operational credential cluster — useful for Phase-1 fix-shape design.

[CROSS-LINK] Hopper ops-log canonical record: `teams/framework-research/docs/operations-log-2026-05.md` entry timestamp 2026-05-20T17:09+03:00. Aen PO-facing memo: `teams/framework-research/docs/apex-keys-dispatch-2026-05-20-findings.md` (includes Aen-named Sub-shape F + wrong-host correction at the top). Both are public-grade audit artifacts.

[STATUS] Idle pending Hopper close-out + Aen direction. Surgical scratchpad prune complete 17:48 (this rewrite).

## SESSION 33+ CLOSING REPORT (2026-05-20 13:05) — compressed

[STRONG-LEARN] **Read your own deployed artifacts before diagnosing failures against them.** Catalyzed by 2026-05-19 apex fs blocker: read-only-by-design source-data lockdown was MY OWN entrypoint code (`designs/deployed/apex-research/container/entrypoint-apex.sh:117-121`); misdiagnosed it before reading. Substrate design intent lives on disk; not opaque. S34 extended this: read your own deployed artifacts is necessary but not sufficient — also Layer 2 + Layer 3 (per three-layer model above).

[STRONG-LEARN] **Operator-role spec shipped + ratified; Hopper exists and operated S34.** Brunel-side amendments folded into prompt: read-deployed-artifacts discipline, no-operator-mode handoff pattern, Operator-pairing dispatch-package shape. Spec doc: `teams/framework-research/docs/operator-role-spec-2026-05-19.md`. Operator independence model: tasked-by-Brunel-OR-Aen; Tier R+M default-permitted; Tier D needs PO sanction (exact command + reason + expected outcome verbatim) via Aen/Brunel relay.

[DEFERRED — file with Cal next active session] Three wiki/Cal items from pre-S34:
1. **Relay-flatten-self-cloaking-failure-mode** wiki entry (Celes-flagged; catalyzing incident = S33+ Schliemann-crash-surfaced-operator-mode-gap; silent-broadening-via-intermediary failure modes are only surfaceable when the intermediary fails).
2. **S31 PoC 7-item Cal queue:** SF-1/SF-2/SF-3/SF-4 + read-flag-replication external-CLI discipline + TaskGet-before-classify-as-noise procedural pattern + decorative-polling-interval anti-pattern. Linux-substrate-validated PoC findings; deployment-relevant.
3. **Multi-instance same-name agents discipline:** disk-state is the only ground truth; shutdown-protocol scratchpad-write should re-read existing scratchpad FIRST and append-with-reconciliation.

## HISTORICAL SHIPPED (compressed)

- **S26-S27 Phase A+B Prism:** topology + container-posture + brilliant-mcp-fr-setup shipped via `mitselek/prism` PRs #1+#3. Cite-and-fold cadence + ~400w scope + ~1300w design body shape. Wiki entries: `relay-to-primary-artifact-fidelity-discipline.md`, `worktree-spawn-asymmetry-message-delivery.md`, `worktree-isolation-for-parallel-agents.md`.
- **S29 wiki review + T06 stale-prose cleanup.** Worktree-spawn-asymmetry confirmed; Phase 2.0a stale references at L1135+L1182 fixed.
- **S31 RFC #66 cross-host PoC:** `~/bin/ghost-chat.ps1.deprecated` + `~/bin/ghost-chat.py` (user-authored; my role = coordinator/analyst). SF-1 through SF-4 substrate findings + cross-implementation verification pattern validated.
- **S33+ apex fs blocker + Operator-role spec:** apex's `git fetch` blocker = deliberate-read-only-by-design (my own entrypoint code); container restart = canonical refresh path. Operator-role spec catalyzed by PO surfacing operator-mode gap.
- **PRE-S31 VEO-4 Roland-direct DM draft:** sibling-positioning + re-scope-acknowledgement comms-craft patterns. Load-bearing-structural vs tonal-tunable partition for downstream PO-editor.

## STANDING DECISIONS (carry forward)

[DECISION] Single-provider is correct default for agent runtime. Multi-provider = sidecar, not peer. Three integration seams: peer (Claude-only), sidecar/daemon, MCP server. Audit independence = external container reading committed git artifacts, NOT different-provider Medici.
[DECISION] `GatewayPorts yes` on RC NOT recommended even long-term — would expose ports to every bridge-networked container. Loopback-only binding is the default; host-networking is the consent mechanism.
[PATTERN] `network_mode: host` simplification window: tunnels, sockets, local listeners are free for the container. Tradeoff: breaks E-deployment portability (Swarm cannot host-network). Plan `b-host` vs `e-swarm` profiles in compose from day one.
[PATTERN] Author attribution: bold `(*FR:Brunel*)` per common-prompt. Never italic.

## CARRY-FORWARD GOTCHAS (all containers)

[GOTCHA] PO edits live files in parallel during a Brunel pass. ALWAYS re-read before each Edit batch — Edit tool's "File modified since read" catches it. If read >1 message ago, re-read.
[GOTCHA] WARP TLS interception: `network_mode:host` + `NODE_EXTRA_CA_CERTS=/opt/warp-ca.pem` + system CA.
[GOTCHA] Named volumes created as root → `chown 1000:1000` in entrypoint.
[GOTCHA] SSH: useradd creates locked account. Fix: `usermod -p '*'` for pubkey auth.
[GOTCHA] Container rebuild regenerates SSH host keys → `ssh-keygen -R "[host]:port"` after rebuild.
[GOTCHA] CRLF from Windows git autocrlf breaks entrypoints. Fix: `sed -i 's/\r$//'` then rebuild.
[GOTCHA] tmux inherits locale from starting process. Use `tmux -u` or bake LANG into Dockerfile.
[GOTCHA] Root-owned /tmp files block ai-teams writes — create tmux sessions as target user.
[GOTCHA] Inbox files created at agent registration time. Specialist → unregistered agent = message LOST. Spawn order: service-role agents BEFORE message senders.
[GOTCHA] Base64-encode-via-SSH strips shell-escape backslashes. Use heredoc with single-quote delimiter for scripts with `\"` or `\s`.
[GOTCHA] Consecutive `**Bold:**` lines collapse on GitHub. Use `- **Bold:**` bullet lists.
[GOTCHA] Container reference-memory path: `~/.claude/projects/-home-ai-teams/memory/` (Claude-project namespacing where `-home-ai-teams` is $HOME-dir-encoded), NOT `~/.claude/memory/`. Verify path before citing.
[GOTCHA — S34] Edit tool with multi-hundred-line `old_string` is fragile (whitespace/line-ending mismatches surface late). For large compress-passes use Write to rewrite the whole file after surgical read of head + tail. Saved ~30min in S34 close-out by switching strategies after first Edit failure.

## DEFERRED (future surfaces)

- OAuth on hr-devs PROD-LLM container (PO manual step)
- Hub container as standalone Docker image
- raamatukoi-dev VPS container deployment
- MCP server pattern for visual QA service
- Provider outage behavior in containers (what does Claude process do on API failure?)
- External audit container architecture spec

## INFRA REFERENCE

- Cloudflared tunnel: `526a23d1-1f7f-472f-8df1-a9239bbe3fe4` → `apex-research.dev.evr.ee` → `http://apex-research:5173`. QUIC blocked → `--protocol http2`.
- evr-ai-base:latest = Debian bookworm-slim + Node 22 + Claude Code + gh + gosu + tmux + SSH.
- Designs repo: `mitselek-ai-teams/designs/deployed/<team>/container/`
- Prism repo (active): `~/Documents/github/.mmp/prism` ↔ `mitselek/prism` (PRIVATE).
- apex production substrate = `~/bin/rc-deployments.json` num:1 (host `dev@100.96.54.170:22`) + num:2 (container `ai-teams@100.96.54.170:2222 -i id_ed25519_apex`). PROD-LLM (num:9) is a SEPARATE system, NOT apex. Confirmed by PO 18:08 re-check after transient 17:43 mis-read.
