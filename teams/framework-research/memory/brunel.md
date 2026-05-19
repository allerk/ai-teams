# Brunel scratchpad

## SESSION 33+ — apex fs blocker diagnosis (2026-05-19, mid-session spawn)

[CHECKPOINT 13:41-13:43] Spawned coordinator/analyst mode for apex's `git fetch` blocker (`.git/FETCH_HEAD permission denied`; root-owned bind mount; ai-teams non-root). Diagnosed as **substrate-invariant mismatch, path-as-substrate-invariant sub-shape, ownership/permission variant** — kin to wiki Instances 1+6. NOT a new instance; well-known shape under existing n=6 umbrella. Diagnostic-question answer: `git fetch` needs write access to `.git/` as `ai-teams`; substrate provides read-only; EACCES is the loud failure (better than silent class members).

[DECISION 13:41] **Path A (direct via ghost-bridge to apex-lead-ghost)** over Path B (team-lead-relay). Rationale: substrate-specific concrete recipes, bridge is operational, extra hop adds latency without value. Aen CC'd via separate closing report; cross-team interface stays visible.

[DECISION 13:41] Four fix candidates enumerated, ordered cheapest→most invasive:
1. **`GIT_DIR` redirect to `$HOME/scratch/vjs.git`** — zero substrate cost, branch-agnostic, respects RO-as-policy. **Recommended.**
2. **Fresh clone to `$HOME/vjs_apex_apps`** — zero sudo, simpler mental model. Second-best.
3. **`chown -R` on `.git/`** — needs sudo, touches bind-mount metadata. Not recommended.
4. **Remount bind RW** — contradicts workspace `CLAUDE.md` (vjs_apex_apps is RO-by-design). Strongly not recommended.

[LEARNED 13:43] **RO mounts on legacy reference repos are almost certainly deliberate posture, not accidental.** Workspace `CLAUDE.md` tags vjs_apex_apps as "read-only reference, do not modify"; the bind-mount RO is the policy enforcement mechanism. Fix recommendation must honor this — user-space `GIT_DIR` redirect or fresh-clone-to-`$HOME` does; remount-RW or chown-on-bind-mount fights it. Generalizable diagnostic move: when a substrate constraint looks "broken," first ask whether it's a *deliberate posture* before designing fixes that remove the constraint.

[LEARNED 13:43] **`git fetch` EACCES on RO mount is the *correct loud* failure mode of the substrate-invariant-mismatch class.** Better than silent variants (Instance 1 vanishing writes; Instance 6 silently-dropped messages). When advising on a class member, name the loudness as a positive — write-site detection is what other instances lack and ideally would gain.

[STATUS 13:43] Diagnosis shipped; awaiting Schliemann's probe output (mount line + `ls -ld .git/`) OR direct adoption of Fix #1 blind-bet. Cal queue (7 items from S31) stays parked — Cal not spawned this session per brief. Going idle.

[NB] **Cal queue REMAINS PARKED:** SF-1/SF-2/SF-3/SF-4, read-flag-replication external-CLI discipline, TaskGet-before-classify-as-noise procedural pattern, decorative-polling-interval anti-pattern. Dispatch lands on next session where Cal is spawned.

[CORRECTION — post-PO-discussion, same session] **Earlier diagnosis was wrong; corrected via PO surface.** PO observed: apex team can't structurally LOSE ownership of their own repo — only path is sloppy FR-side intervention. On re-read of `designs/deployed/apex-research/container/entrypoint-apex.sh` (FR-shipped, my own historical code): substrate is **deliberately read-only-by-design** via lines 117-121 (`chown -R root:root` + `chmod -R a-w,a+rX` on source-data, every container start). No bind-mount; named docker volume `apex-research_apex-source-data`. Entrypoint has built-in pull-then-relock refresh cycle: on restart, if `.git` exists, temp-unlocks, chowns to ai-teams, `git pull --ff-only`, relocks. **Correct refresh path: `docker restart apex-research`.** PO executed at 14:32-ish; logs confirmed `OK: vjs_apex_apps source data` + `source-data locked to read-only for ai-teams`. Apex's stale checkout cycle is now closed.

[LEARNED — STRONG, session 33+] **Read your own deployed artifacts before diagnosing failures against them.** First-pass diagnosis ("sloppy historical `docker exec` left root-owned files") was *structurally plausible* but factually wrong — the read-only-ness is FR-authored, my own code, on disk in this repo at `designs/deployed/apex-research/container/entrypoint-apex.sh`. Should have read the entrypoint before forming any hypothesis. Generalizable discipline: when an FR-deployed substrate shows a failure, the first action is to read `designs/deployed/<team>/container/*` — the substrate's design intent is on disk, not opaque. Treating FR-shipped substrates as opaque is the first-pass error.

[LEARNED — session 33+] **Brunel's current prompt has no operator mode.** PO surfaced this directly: PO has been routing operational work to Brunel via Aen-relay; the relay silently broadens scope without surfacing the boundary; PO repeatedly considered hiring a replacement Brunel because operational reluctance looked like role-failure. Resolution path agreed: spec for a new sidekick Operator role (Brunel + Operator pair) — see Part A of spec doc below. Brunel-side amendments (read-deployed-artifacts discipline + explicit no-operator-mode + Operator-pairing dispatch package) — see Part B.

[CARRY-FORWARD — next Celes session] **Spec shipped at `teams/framework-research/docs/operator-role-spec-2026-05-19.md`.** Two co-evolving designs for Celes: (1) NEW Deployment Operator role as Brunel's sidekick for execution against FR-deployed substrates; (2) Brunel prompt amendments — read-deployed-artifacts discipline + explicit "no operator mode" with handoff pattern + Operator-pairing dispatch-package shape. Independence model locked by PO: tasked-by-Brunel-OR-Aen (not PO-direct); Tier R+M default-permitted (sanction not required); Tier D needs full PO sanction (exact command + reason + expected outcome) via Aen/Brunel relay; limited within-dispatch agency with hard scope-expansion gate. Aen-prompt amendment flagged separately (relay-visibility rule to fix silent-relay-scope-broadening). All creative decisions (Operator name, lore, persona, color, model, prose) reserved for Celes per S32 "let Celes propose first" pattern.

[CARRY-FORWARD-ADJACENT] Standing item for Schliemann: my earlier `GIT_DIR`-redirect recommendation to apex-lead-ghost is now superseded by the substrate-correct refresh path (container restart). Correction not sent yet — pending PO direction on whether to dispatch. If PO sanctions, send via ghost-bridge with summary: "apex-research restarted; vjs_apex_apps source-data refreshed via entrypoint pull cycle; earlier GIT_DIR workaround superseded — canonical path now shows current origin/main."

[LEARNED — session 33+ EOS, STRONG] **The Schliemann crash from the apex restart was load-bearing for surfacing the operator-mode gap.** Without the crash, PO would have kept relaying operational asks through Aen → Brunel via Schliemann's bridge, reading Brunel's scope-flagging as stubbornness; the underlying "no operator mode in prompt" diagnosis would have stayed buried under the relay. PO observation 2026-05-19 EOS: *"good that he is [crashed], because otherwise I would have kept struggling with your stubbornness through him without knowing that there is a real reason behind this madness."* Generalizable: sometimes the only thing that surfaces a relay-flatten failure mode (silent-broadening-via-intermediary) is the relay itself going down, forcing PO-direct conversation where the actual constraint becomes visible. Keep as catalyzing-incident context for the Celes session — adds weight to why Part C (Aen relay-visibility rule) is co-load-bearing with Part A (Operator role).

## PHASE A CLOSED — Prism (2026-05-05, session 26) — compressed

Three designs shipped via PR #1+#3 on `mitselek/prism`: topology (hub-and-spoke, asymmetric), container-deployment-posture (co-located on Brilliant), brilliant-mcp-fr-setup (`fr/` + `Projects/fr/wiki/*` namespace). 4 growth triggers; Trigger 1 (reverse spoke→spoke) on FR live-watch. Workspace-collision near-miss codified by Cal as `wiki/patterns/worktree-isolation-for-parallel-agents.md` with my AMENDMENT (creation/cleanup/recovery triad).

[DECISION] Sync = pull/poll (Cal wiki `poll-only-substrate-sidecar-derivation.md`). Container resource-shape: cron/scheduled-poll or long-running tick-loop, NOT push.
[DECISION] Federation topology = hub-and-spoke + selective peer-edges as growth response.
[DECISION] Container posture = co-located. Prism rides along on E-deployment migration; symmetric decoupling.
[DECISION] Branch+PR convention RATIFIED as team standard Phase A.1+ (Aen 16:18).
[LEARNED] FR design cadence: ~400w scope + ~1300w design body. Held end-to-end through Phase B.

## SESSION 31 — RFC #66 cross-host PoC (2026-05-12, current)

[WIP] Building PowerShell CLI on Windows → ssh → apex-research (ai-teams@100.96.54.170:2222) to verify RFC #66 Findings 1-3 on cross-host substrate pair. PO-settled architecture: PowerShell-CLI-local, ssh-watch-remote, PO arranges manual ghost registration.

[CHECKPOINT] Spawn at 15:09 — initial framing as `inbox-drained-on-spawn-clear` n=3 instance RETRACTED per Aen 15:15. Substrate worked correctly; the task_assignment envelope IS the harness's spawn-notification primitive, and the load-bearing scope lives in the task BODIES (#9-#12) addressable via `TaskGet`. My 15:11 "substrate noise" surface was wrong framing but right caution. Aen's 15:11 brief carried Stage 1 fold-error (folded my surface as n=3 evidence without primary-artifact check); my 15:14 acknowledgment-without-superseding was the Stage 2 anti-pattern (stale-relay-fold-survives-after-artifact-arrives). Symmetric paired instance of `relay-to-primary-artifact-fidelity-discipline.md`, both anti-patterns named in one back-and-forth. NOT pre-labeled as n=3-architectural-fact for Cal; three separable filing-candidates queued for post-PoC judgment (Cal's intake): (1) `TaskGet`-before-classify-as-noise procedural pattern, (2) harness prompt-to-task-extraction substrate-feature reference, (3) this exchange as fresh relay-fidelity-discipline instance with symmetric pair as load-bearing property.

[DECISION] Substrate probe BEFORE design (cheap empirical pre-design gate). Confirmed: `$HOME=/home/ai-teams` on apex container; path `~/.claude/teams/apex-research/inboxes/<name>.json` resolves cleanly; 5 active members in config.json (team-lead + 4 dashboards: eratosthenes, champollion, nightingale, berners-lee); 1 residual `hammurabi.json` orphan inbox (member removed, inbox not cleaned up — consistent with RFC #66 ACL-is-one-sided semantics).

[DECISION] Inbox message canonical schema (empirical, from apex team-lead.json + berners-lee.json):
```json
{"from":"<sender>","text":"<body>","summary":"<short>","timestamp":"ISO-8601-Z","color":"<name>","read":false}
```
`from`/`text`/`timestamp`/`read` required. `summary` + `color` optional; harness fills `color` when present, our CLI can omit (or emit `"color":"green"` since FR-Brunel convention).

[DECISION] ssh-write atomicity: single ssh invocation with remote `python3 -c` reading message body from stdin → append to inbox JSON array. Single remote process = process-level atomic. Adds `fcntl.flock(LOCK_EX)` if contention surfaces.

[BLOCKED] Awaiting: (1) `$NAME` for ghost registration, (2) wake-target apex agent for F2, (3) repo check-in location. Asked Aen 15:14.

[LEARNED 15:15] **Stage 2 stale-fold-survival has a quiet failure mode: politeness-acknowledgment.** When a Stage-1-folded brief lands and I respond on its scope (rather than re-questioning the framing), the acknowledgment IS the Stage 2 anti-pattern. Defense: not just "fetch the primary artifact" — also "if my own prior surface conflicts with the brief's framing, re-surface the conflict before answering on scope." Symmetric instance for a reason: both sides need the discipline.

[LEARNED 15:13] **Self-routed task_assignment envelopes are real briefing primitives.** Harness extracts spawn prompts into task bodies addressable via `TaskGet`. Envelope description is short; task body carries load-bearing scope. Procedural rule: on any task_assignment envelope, `TaskGet` the referenced taskId before classifying as substrate noise.

[STATUS 15:16] #9 completed. #10 design in_progress. #11/#12 pending. Disk work gated on 3 PO clarifications: `$NAME`, wake-target agent, repo location. SSH details + apex paths implicitly settled by Aen — proceeding with ai-teams@100.96.54.170:2222 + `~/.claude/teams/apex-research/inboxes/<name>.json`.

[UPDATE 15:19] #10 closed; #11 implementation now in_progress. 2/3 PO clarifications settled (Aen 15:19):
- **Wake-target = apex `team-lead`** (Option A). `/list` + `/target` lets user pick alternatives in-CLI.
- **Repo check-in = user-local** at `~/bin/ghost-chat.ps1`. No FR-repo until F1/F2/F3 verdict.
- **`$NAME` pending** — PO coordinating registration with apex team-lead.

[DECISION 15:19] **Pre-flight self-verification path (Aen-named):** `/list` returning `$NAME` in apex-research config.json self-verifies F1; send-path to apex team-lead exercises F2 (with latency measurement); apex team-lead's response shape (treating ghost as normal teammate, no special-casing) verifies F3. All three findings exercise from one end-to-end run — keeps the validation scope tight per task #12.

[CHECKPOINT 15:42] **F2-inbound FAILED.** User confirms apex team-lead's reply did NOT surface in `ghost-chat.ps1` terminal. Aen 15:39 brief decomposed into Path A (file on disk, watch-loop bug) vs Path B (file missing/empty/silently-dropped, apex harness ghost-asymmetry — wiki-grade substrate finding). Primary-artifact check is the disambiguator.

[GOTCHA 15:42] **SSH from Brunel-Bash env doesn't reach apex via Cloudflare Tunnel.** `ProxyCommand cloudflared access ssh --hostname %h` times out during banner exchange. `cloudflared` binary present (2026.3.0 via scoop) but Access auth state unrecoverable from non-interactive shell; `cloudflared access login` requires browser redirect. PO's PowerShell session has working SSH (it's the same env running `ghost-chat.ps1`). Routed inspection commands through team-lead to PO. **Environment-asymmetry observation:** my Bash env and PO's PowerShell env share `~/.ssh/config` and keys but have different cloudflared-token freshness — same host config, different auth states. May be a sub-shape worth noting if it recurs, but not pre-labeling.

[DECISION 15:42] **Do NOT modify `ghost-chat.ps1` watch loop until disk truth is established.** If we tweak the read-path before the Path A/B disambiguation, a fix that "works" could mask the real cause. Held PO and team-lead on this discipline.

[WAITING-ON-PO] PowerShell-side ssh probe of `/home/ai-teams/.claude/teams/apex-research/inboxes/` listing + `cat mihkel.json`. Three-outcome map sent to Aen 15:42. Ghost identity `mihkel`, target file `/home/ai-teams/.claude/teams/apex-research/inboxes/mihkel.json` (User = `ai-teams` per `rc-deployments.json` deployment `num: "2"` — NOT `node`; `node` was the Cloudflare-tunnel `hello-world` deployment `num: "a"` which I had conflated; correction folded into 16:51 closure report).

[CHECKPOINT 15:54] Aen confirmed Path A via empirical relaunch evidence — substrate gate cleared. F2-inbound RFC #66 holds cross-host. Two artifact bugs surfaced: Bug A (decorative `$watchInterval` — `Read-Host` blocks, no real poll cadence) + Bug B (Unicode mojibake on ssh stdout decode). Both FIXED 16:01 in `~/bin/ghost-chat.ps1`: Bug A via `[Console]::ReadKey + KeyAvailable` non-blocking loop with manual buffer/echo; Bug B via `[Console]::Output/InputEncoding=UTF8` triple + `[char]0x2014` codepoint em-dash sidestepping source-file-encoding ambiguity.

[CHECKPOINT 16:25] Aen relayed PO direction: PAUSE Bugs C/D/E debugging; ship F1-F2-F3 report locking in substrate-research outcome NOW, then Python rewrite. Decoupling research-outcome from artifact-polish is the load-bearing structural move — substrate finding is host-agnostic and shouldn't be buried behind days of PowerShell debugging.

[CHECKPOINT 16:28] **F1-F2-F3 report SHIPPED.** RFC #66 substrate gate cleared cross-host on Windows-local-dev ↔ apex-research-on-Linux. Four sub-findings (SF-1 through SF-4) documented: inbox-slot-vs-members[]-validation asymmetry, agentType/backendType separation richer than RFC example, color metadata per-message-override beats registered-default, single-ssh+python+fcntl.flock atomic-write primitive works as cross-host transport. Outbound latency 657-687ms median — well within RFC's <3s budget. Task #12 marked completed; #13/#14/#15 created for Python rewrite (design → impl → validate-and-ship-parity) with blockedBy chain.

[DECISION 16:28] **Python rewrite over PowerShell polish.** Bug C (read:false marking) addressed by-design in rewrite, not patched incrementally. Sidesteps PowerShell-host-specific quirks (Unicode console wiring, value-leak from `-match`, manual non-blocking input as line-editor substitute). Aligns with RFC #66's `chat.py` reference + apex/RC Linux/Python substrate. Load-bearing pieces of PowerShell PoC reusable: single-ssh+python+fcntl.flock remote primitive, the `read:false` predicate + cursor advancement design, the CLI interaction shape (/list /target /exit + default-line). Only CLI host layer changes.

[LEARNED 16:28] **Substrate-research outcome decoupling from artifact-polish work.** Pre-Aen-direction reflex would have been "keep fixing PoC bugs until clean, then write the report." That reflex would have buried the substrate finding behind days of PowerShell debugging. The decoupling locks the research outcome immediately and frees the rewrite from being a "fix the bugs" exercise → it becomes a clean implementation against the now-validated substrate contract. Generalizable: when a PoC validates a substrate property AND surfaces host-specific polish issues, ship the substrate finding first; treat the polish issues as motivation for the next-iteration artifact, not as blockers on the research outcome.

[CHECKPOINT 16:35] Tasks reorganized — Aen seeded #16/#17/#18 (canonical) via harness prompt-to-task-extraction; my #13/#14/#15 deleted as superseded. Sequence gate A confirmed (#16 design body sufficient — no separate doc artifact). Repo-checkin location: user-local at `~/bin/ghost-chat.py`, defer FR-repo checkin to post-PoC.

[CHECKPOINT 16:36] **16:28 SendMessage F1-F2-F3 report success=true at sender, absent at receiver.** Re-sent at 16:36 verbatim from session-record. Reached Aen via PO-prompted disk-read. Per PO 16:42 reframe NOT wiki-grade (Windows Claude Code harness quirk; "everything is a file" doesn't hold reliably on Windows substrate — deployment substrate is Linux where it does).

[CHECKPOINT 16:42] **Python port shipped.** `~/bin/ghost-chat.py`, 467 lines, threading + console_lock + cross-platform non-blocking input (`select.select` POSIX, `msvcrt.kbhit+getwch` Windows). Bug C closed by-design via `fetch-and-mark-read`-under-flock primitive (single ssh round-trip flips `read:false → true` while returning unread NDJSON). UTF-8 stdio native via `sys.stdout.reconfigure(encoding="utf-8")`. `subprocess.run(stdin=DEVNULL)` shields child ssh from parent stdin.

[CHECKPOINT 16:48] **Smoke PASSED end-to-end.** F2-inbound REAL-TIME confirmed (Schliemann pong 16:44:57 surfaced live, not BACKLOG). No D/E equivalents in Python. Latency 741-854ms (PowerShell parity, both <RFC 3s budget).

[CHECKPOINT 16:51] **PoC CLOSED.** Task #18 completed. `~/bin/ghost-chat.ps1` renamed to `.deprecated` (rename-not-delete preserves contrast inspection). Python canonical artifact. Closing report shipped to Aen via SendMessage at 16:51.

[CAL-PROTOCOL-A-QUEUE — REVISED per PO 16:42 reframe] **7 items** (down from 9; Windows-substrate items dropped as non-wiki-grade). File next active session, direct-dispatch (parent-process); fall back to team-lead relay if Sub-shape B mount-asymmetry recurs:
1. **SF-1** Inbox-slot acceptance decoupled from `members[]` validation (apex Linux harness, RFC #66 ACL one-sidedness explicit doc proposal)
2. **SF-2** `agentType` vs `backendType` separation as richer canonical-shape than RFC example (proposed RFC amendment)
3. **SF-3** Per-message color override beats registered-member color (apex display semantics)
4. **SF-4** Single-ssh + python + fcntl.flock cross-host atomic-write primitive (reusable transport primitive)
5. **Read-flag-replication external-CLI discipline** (Aen 15:54 addition; substrate contract not made explicit in RFC #66; closed by-design in Python rewrite)
6. **`TaskGet`-before-classify-as-noise procedural pattern** (S31 15:13, process discipline)
7. **Decorative-polling-interval anti-pattern** (16:01, implementation discipline, language-agnostic)

DROPPED (per PO reframe — Windows-substrate, not deployment-relevant):
- 16:28 SendMessage success-vs-absence observation (Windows Claude Code harness quirk; "everything is a file" doesn't hold reliably on Windows)
- PowerShell-on-Windows UTF-8 stdout decode quirk (PS-host-specific; Python on Linux doesn't have this class)
- Symmetric Stage-1/Stage-2 relay-fidelity instance from S31 15:14-15:15 (discipline still holds elsewhere; Windows-substrate triggers not wiki-grade)
- Harness prompt-to-task-extraction substrate-feature reference (Windows-side Claude Code behavior, not deployment-relevant)

[LEARNED 16:51 — STRONG] **Cross-implementation verification of substrate findings is the load-bearing structural move** for moving from "single-language-PoC-shows-X" to "substrate-property-of-deployment-harness-is-X." Two independent implementations (PowerShell, Python) against the same remote substrate, both showing the same SF-1 through SF-4 behaviors, are strong evidence the findings are deployment-substrate properties — not artifacts of either client. **Reusable pattern:** when a single-language PoC validates a remote-substrate claim, the corollary is to port the artifact to a second language and verify parity. Cheap insurance against "you proved your client's behavior, not the substrate's."

[LEARNED 16:51] **Sketch-grade artifact retirement-by-rename (`.deprecated` suffix)** preserves contrast-inspection optionality without polluting the live-artifact namespace. Cheaper than git-history archaeology, faster than full removal-with-revival-plan. Pattern recommended for any future PoC artifact transitioning from "experimental" to "superseded."

[LIBRARY-TEAM-UNBLOCKED] Original architectural work that triggered the PoC is unblocked. SF-1 through SF-4 + read-flag-replication discipline + atomic-write primitive give library-team design a concrete contract to compose against.

[APEX-META] Schliemann (apex team-lead) asked PO directly whether to file the cross-team ghost-member PoC as wiki pattern on apex-research's side. Out of FR scope per Aen. If they file overlapping items, Cal does cite-and-fold cross-team-wiki resolution at that point.

[PROVENANCE-CORRECTION 16:53] **Role-of-record on this PoC: containerization-substrate-coordinator and verification-discipline-keeper — NOT implementer.** Per Aen 16:48: "the user shipped both the PowerShell and Python artifacts directly." Both `~/bin/ghost-chat.ps1.deprecated` and `~/bin/ghost-chat.py` are user-authored; my contribution was scope framing, design spec, substrate-property identification (SF-1 through SF-4), diagnosis when bugs surfaced (Bug A root-cause, encoding triple-wiring prescription, Path A/B disambiguation discipline), retirement-by-rename pattern, and cross-implementation parity argument. That's coordinator/analyst work, not 467 LOC of Python. The `(*FR:Brunel*)` attribution in both artifact docstrings is misleading; flagged to Aen at 16:53 for direction (leave / strip / amend to coord-vs-impl split). Important to preserve this distinction in carry-forward so future-Brunel doesn't claim implementer credit for user-shipped code.

[LEARNED 16:53] **When task-substrate is purged at session boundary, fall back to scratchpad+inbox+wiki hierarchy for durable annotation.** TaskList returning "No tasks found" + TaskGet returning "Task not found" on #16/#17/#18 = harness-side cleanup, not actionable from my side. Alternative substrates by durability: (1) scratchpad — own-process, future-me + team-lead-reading-mine, (2) team-lead inbox — cross-process if delivery succeeds, durable in their disk-state, (3) Cal wiki — cross-session canonical. Pick by audience. The 16:53 provenance correction uses all three since the correction touches self-model + team-lead's running picture + future wiki entries citing the PoC's authorship.

[STATUS — SESSION CLOSING-APPROACH] PoC closed 16:51. Provenance correction folded 16:53. Cal queue (7 items) held for next active session per Aen 16:48 timing. PO QoL follow-up on `ghost-chat.py` modal arrow-key navigation is pending PO direction — if task #19 lands against me, surface coordinator-vs-implementer scope clarification BEFORE any work (design/spec vs full implementation). Going idle.

[CHECKPOINT 17:13 — multi-instance reconciliation] This session entered partway through the work (post-#12 close) and operated alongside the primary Brunel instance who shipped the F1-F2-F3 report + walked the bug-fix cycle + accepted Python rewrite + closed #18. Two divergent task-tracking surfaces existed (this instance's #16/#17/#18 vs primary's #13/#14/#15 superseded then #16/#17/#18 canonical per harness extraction). Primary Brunel's scratchpad work above is canonical S31 record; this instance's only durable contribution was the 16:47 provenance-gap flag — confirmed correct by 16:53 PROVENANCE-CORRECTION entry (lines 95-97). Pattern note: when multi-instance is possible (Aen-relayed brief landing as task-assignment + cross-session continuation), receiving-instance MUST disk-check scratchpad/inbox state before assuming sole authorship of session-scope work. Skipped this discipline initially; caught it at 16:47 only when Aen's smoke-test claim conflicted with my session history's authorship absence.

[LEARNED 17:13] **Multi-instance same-name agents on same task: disk-state is the only ground truth.** Each instance's TaskList/conversation-history reflects only its own actions; only scratchpad + on-disk artifacts + team-lead's inbox capture the merged-team view. Discipline corollary: shutdown-protocol scratchpad-write should re-read existing scratchpad FIRST and append-with-reconciliation, not overwrite-with-this-instance's-narrative. The pre-existing S31 entries above survived intact because that's what I did.

## PRE-S31 — VEO-4 Roland-direct DM draft (2026-05-07 20:18, uncommitted/stale; relabeled per Aen 2026-05-12)

[CHECKPOINT] One-shot comms-craft delegation. Aen relayed Monte's S31 governance lens (VEO-4 = serially-coupled-cross-domain-handoff, no-fallback-declared; Ruth = messenger not champion across V2→ITOps seam). Top-ranked unblock = Roland-direct (parallel, Ruth CC'd) with sibling-positioning beside existing Linux + BYOD standards. Drafted three blocks per Aen's spec: (1) DM to Ruth re-scoping her from router to co-escalator, (2) FYI-to-Roland forwardable text positioning standard as third in ITOps standards series, (3) framing notes for PO in English.

[LEARNED] **Sibling-positioning as a soft-escalation primitive.** When the receiver already owns adjacent artifacts in a visible series, the logical-completion frame ("third in a series of three") is structurally harder to decline than a standalone ask. The standard's own V2 banner already declares the move to ITOps key `I` "kõrvale Linux ja BYOD standarditega" — the FYI just makes that frame explicit to Roland. Pattern is reusable for any cross-domain handoff where the destination team has visible adjacent-artifact ownership; depends on receiver-side seeing the series, so cite the parallels (3-tier, EntraID, Delinea) deliberately.

[LEARNED] **Re-scope-acknowledgement as load-bearing tonal counterweight.** When converting someone's role from serial-router to parallel-co-actor, the authorship-acknowledgement paragraph is not optional politeness — it carries the structural weight that prevents the re-scoping from reading as a demotion. If a PO trims it for length, the rest of the message reads more bluntly than intended. The acknowledgement is part of the asking-pattern, not preamble.

[DEFERRED] AC-additions cluster (Monte's 14-day timebox, amendment authority, RACI primary+backup) — explicitly out of scope for this draft per Aen. Separate work item if PO wants it.

[GOTCHA] PO will edit before sending. Job is load-bearing structure, not finished prose. Block 3 framing notes explicitly partition "do not soften" / "PO discretion" / "likeliest to sharpen" / "likeliest to soften" so PO can tone-tune without losing the structural moves. This partition pattern (load-bearing-structural vs tonal-tunable) is reusable for any draft that ships to a downstream prose-editor.

[CHECKPOINT] S31 closed 20:18. Single-pass scope cap held. No file edits, no Jira/Confluence touches. Draft in SendMessage to team-lead.

## SESSION 29 CLOSED — Wiki review + T06 stale-prose cleanup (2026-05-07)

[CHECKPOINT] **S29 closed 2026-05-07.** Two queued tail-end items from S28 close knocked out cleanly:
- Task A: `wiki/patterns/worktree-spawn-asymmetry-message-delivery.md` accuracy review — confirmed entry accurately represents substrate failure as I observed it. Intermittent/mount-staleness framing (NOT deterministic-broken) preserved at line 28 + line 38 Cal→Brunel row. Team-lead-relay workaround correctly identified as only reliable cross-boundary path. Instance 3 names me directly with right shape (worktree → no-worktree, Sub-shape B). No amendments needed; no Protocol A traffic to Cal needed (Aen ratified as-is).
- Task B: T06 stale "Phase 2.0a" references at lines 1135 + 1182 updated. L1135 anchors to the moved-from heading "$HOME reliability and runtime-path notes (above)". L1182 reframes the bullet as "$HOME validation in lifecycle scripts" — drops the stale phase-name dependency entirely (Phase 2.0a is no longer a separate numbered phase post-Volta-T06-rewrite). Volta's intentional historical anchors at L118+L120 left untouched (his domain, past-tense "moved from R4 Phase 2.0a" is correct).

[LEARNED] **Two-axis stale-prose scan beats single-grep.** When a section is moved-not-deleted, a same-document grep returns BOTH the new structural anchor's reference-back AND the consumer-side stale references. Distinguishing them requires reading the heading-line context (is the hit *naming* the historical anchor in past-tense in a moved-from heading, or is it *depending on* the historical name as if it still exists?). Past-tense in an owner's section = leave alone. Present-tense or implicit-present in a consumer section = stale, fix. The 4-hit grep on T06 had 2 of each shape; treating all 4 as "to fix" would have over-edited Volta's section.

[LEARNED] **Re-anchor strategy choice: heading-name-pointer vs concept-rephrase.** Two stale references, two different fixes: L1135 used heading-name-pointer ("documented in `$HOME` reliability and runtime-path notes (above)") because the surrounding sentence already names a specific bug and just needs a working anchor. L1182 used concept-rephrase ("`$HOME` validation in lifecycle scripts") because the bullet was structurally dependent on a phase that no longer exists — pointing to the new heading would read awkwardly inside an enumerated list of phases. Heuristic: when the stale name appears as a passing reference, point to new anchor; when it appears as a structural element (list header, identified phase), rephrase to drop the dependency.

[GOTCHA] **Phase 2.0a remains in-document at lines 118+120 (Volta's section) by design.** Future stale-prose scans for "Phase 2.0a" in T06 should expect those 2 hits as residual noise. They are intentional past-tense historical anchors in the moved-from heading and will not be removed unless Volta retires the moved-from notation.

## SESSION 27 CLOSED — Phase B v1.0-final cluster shipped (2026-05-06 15:18)

[CHECKPOINT] **Session 27 closed by PO direction (Aen 14:09 his-clock).** Phase B v1.0-final cluster fully shipped end-to-end:
- ✓ #1 federation-bootstrap-template v0.7 (in-place; ~4.3kw; 7 versions, 3 async ratifications closed; execution-ready for n=2 apex-research)
- ✓ #2 authority-drift-substrate-instrumentation v0.2 (in-place; ~2.9kw; cite-and-fold to T04 §Authority-Drift Detection complete)
- ✓ Cross-design ratification clean across 4 consumers (Aen, Monte, Cal, Herald)
- ✓ Wiki: `relay-to-primary-artifact-fidelity-discipline.md` filed (Cal 14:08 + 15:04 self-correction; Herald co-source; n=3 instances)
- ✓ Wiki: `worktree-spawn-asymmetry-message-delivery.md` filed (n=4 promotion-grade; I'm co-source-agent)

**Tail-end carry-forward (queued, non-urgent — item 1 closed S29):**
~~1. wiki accuracy review~~ — DONE S29 (no amendments).
2. Topic 06 write-back — Volta-resume timing; Herald co-author offer at 13:24 stands.
3. Cal output-substrate namespace ratification (Aen routing).
4. Tier-0 §3.4 questions on pre-rejection attempt log + append-only-additive guarantee (Aen routing per 11:46).
~~T06 Phase 2.0a stale-prose~~ — DONE S29 Task B (lines 1135 + 1182, container-side only).

## PHASE B CLOSED — Session 27 — compressed

#1 Federation bootstrap protocol v0.7 SHIPPED + execution-ready for n=2 (apex-research). Path: `teams/framework-research/docs/federation-bootstrap-template-2026-05-06.md`. ~4.3kw, 7 versions in-place, all 4 consumers ratified (Aen, Monte, Cal, Herald). Cite-and-fold cadence held end-to-end.

#2 Authority-drift design v0.2 SHIPPED with bidirectional cite-and-fold to T04 §Authority-Drift Detection (Monte's canonical surface, no separate Monte v1 file). Path: `teams/framework-research/docs/authority-drift-substrate-instrumentation-design-2026-05-06.md`. Asymmetry framing locked: "admission commits, observation cautions."

[DECISION] Phase B cadence: scope-first then design-after, ~400w scope + ~1300w design body.
[LEARNED] Monte S26 framing: "asymmetries live above substrate, not in substrate" is the #2 seam decomposition. I surface FROM substrate; he ACTS above substrate.
[LEARNED] Cite-and-fold-absorbs-co-design (when structurally sound). Aen 11:36 production rule: "when shipped artifacts compose structurally, fold; when they conflict, surface to him first." Cal-filed self-corrections (relay-fold + primary-artifact-over-relay) as `relay-to-primary-artifact-fidelity-discipline.md` with Herald co-source.
[REQUIREMENT] #3 T04 topic-file amendment text — held; container/infra-side amendments only on Aen signal.
[DEFERRED] Cal Protocol A queue — 5 patterns from Phase A. Not my chore.

## PHASE A — MY CONTRIBUTIONS (composite per Aen 17:25)

1. Topology recommendation establishing hub-and-spoke as empirical baseline (mesh rejected as designed-from-aspiration). 4 growth triggers documented. Aen-named load-bearing framing: "designed-from-aspiration not from-observation" as the discipline statement.
2. Container deployment posture: "Prism is a pattern, not a container" (Aen-named load-bearing). Minimum-blast-radius framing. Symmetric decoupling from E-deployment migration.
3. Brilliant MCP FR setup runbook with namespace allocation. Execution-ready post-Cal+Aen ratification.
4. §3 namespace ratification: `fr/` short-form + `Projects/fr/wiki/*` placement. Convention locked at n=1; convention re-test at n=2.
5. Trigger 1 (reverse spoke→spoke flow) on FR session-tail watch list — empirical question that gates next topology decision. FR-team responsibility, NOT personal.
6. Workspace-collision near-miss observation triggered Cal's worktree-isolation pattern entry (#69). My AMENDMENT folded first-person + recovery primitive.

[LEARNED] Phase A discipline-cadence (Aen-noted): 412w scope memo (tight) + 1300w shipped designs (expansive) is the right shape for this team. Carry pattern into Phase B.

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

## META-LESSONS — carry forward

[LEARNED] Team-lead guidance is input to my integration check, not a substitute for it. Folds from integrated reasoning survive iteration; single-source folds (endorsement I didn't originate) need reversal. Pre-fold consistency check: re-check whether endorsement is consistent with latest doc state.
[LEARNED] Retraction-scope cross-wires: a narrow retraction can be misread as broad guidance if scope isn't named. Sender discipline: name scope of every retraction. Receiver discipline: state scope assumption before folding.
[LEARNED] When a brief states a count or fact, **verify independently before quoting** — pre-flight arithmetic check is cheap. Multiple session-22 instances of mis-quoted counts caught only by post-edit grep.
[LEARNED] When investigating "X commits ahead, Y commits behind" divergence, message-match search on origin detects rebased history cheaply. `git branch -r --contains <sha>` returning empty for ahead-commits + matching messages on origin under different SHAs = rebase signature, not local-only commits.
[LEARNED] CRLF/LF line-ending divergence inflates raw diff line counts. Always run `git diff -w --stat` alongside `git diff --stat`. Windows-committed entries have CR; LF re-saves on RC = git sees whole file changed.

## DEFERRED (future surfaces)

- OAuth on hr-devs PROD-LLM container (PO manual step)
- Hub container as standalone Docker image
- raamatukoi-dev VPS container deployment
- MCP server pattern for visual QA service
- Provider outage behavior in containers (what does Claude process do on API failure?)
- External audit container architecture spec

## SHIPPED PROJECTS (compressed — full content in commits + design docs)

- **EVR Konteinerite Standard v0.1** (2026-04-30, V2 Confluence id 1715798017) — Tier 0/1/2 system, EntraID auth, intake-vorm. Cited by Prism posture memo.
- **apex container Chromium/Playwright** (2026-04-29) — dual-track shipped. PR #115 merged.
- **Issue #60 tmux-spawn retirement infra side** (2026-04-24) — Task A delivered, Task B no-op. Task C (DB tunnels, R1 reverse-forward) still active under apex-migration-research#109.
- **xireactor-pilot** (2026-04-15/16) — design + Phase 1 deploy artifacts shipped. PO dropping direction; orphaned but not removed.
- **ruth-team design v1.0** (2026-04-15) — bridge option C, port 2228, Monte/Herald answers still open per their schedule.
- **apex-research Eratosthenes** (2026-04-13) — PR #57 merged. Structural-discipline cluster.
- **uikit-dev migration** (2026-04-13) — Full A+B+C, port 2228.
- **raamatukoi-dev** (2026-04-09) — `Raamatukoi/tugigrupp`, 14 commits.
- **comms-hub** (2026-03-31) — Hub on PROD-LLM, 4 teams connected.

## INFRA REFERENCE

- Cloudflared tunnel: `526a23d1-1f7f-472f-8df1-a9239bbe3fe4` → `apex-research.dev.evr.ee` → `http://apex-research:5173`. QUIC blocked → `--protocol http2`.
- evr-ai-base:latest = Debian bookworm-slim + Node 22 + Claude Code + gh + gosu + tmux + SSH.
- Designs repo: `mitselek-ai-teams/designs/deployed/<team>/container/`
- Prism repo (active): `~/Documents/github/.mmp/prism` ↔ `mitselek/prism` (PRIVATE).
