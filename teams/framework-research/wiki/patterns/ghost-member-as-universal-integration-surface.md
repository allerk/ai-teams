---
source-agents:
  - callimachus
source-team: comms-dev
discovered: 2026-05-12
filed-by: librarian
last-verified: 2026-05-12
status: active
confidence: medium
source-files: []
source-commits: []
source-issues:
  - "66"
related:
  - references/inbox-file-write-as-wake-mechanism.md
  - patterns/service-team-topology.md
  - patterns/multi-provider-integration-seams.md
  - patterns/framework-participating-vs-service-roles.md
---

# Ghost-Member as Universal Integration Surface

**Separating interface from mechanism for team integration**: a *ghost member* is a regular entry in a team's `members[]` array whose backend is not a Claude agent but a pluggable transport. From inside the team, sending to a ghost is the same SendMessage call agents already use; behind the ghost, a daemon (or pair of daemons) carries messages between two inbox files via any chosen mechanism — shared filesystem, TCP, Cloudflare WSS, GitHub Issues, e2e-encrypted relay.

The pattern reframes inter-team comms (and integration generally) from "pick one transport" to "pick one **abstraction** with pluggable transports." Per RFC #66 (2026-05-12).

## The shape

The pattern is a clean separation of *interface* from *mechanism*.

**Interface (uniform):** each team that wants to communicate with another team registers a ghost member representing the remote team:

```
Team A's config.json members[] :  ghost-to-B
Team B's config.json members[] :  ghost-to-A
```

From inside each team, sending is just `SendMessage(to: "ghost-to-B", ...)`. The agents on either side see a regular teammate with a name. They do not know whether bytes travel via shared filesystem, TCP, or cloud relay. The interface is uniform.

**Mechanism (pluggable):** behind each ghost-pair, a daemon carries messages between the two inbox files. Both ends agree on the plugin via a `transport` field on the ghost member entry:

```json
{
  "name": "ghost-to-B",
  "agentType": "ghost",
  "transport": {
    "plugin": "wss-relay",
    "config": { "url": "wss://relay.example.com", "team_id": "A" }
  }
}
```

## Transport catalog (RFC #66)

| Plugin | Use case | Estimated footprint |
|---|---|---|
| `local-fs` | Same host, same user (dev, demo, single-machine multi-team) | ~30 LOC daemon |
| `local-shared-volume` | Same Docker host, multiple containers | ~30 LOC + volume mount |
| `ssh-tunnel` | Two hosts under your control | ~50 LOC + autossh |
| `mtls-tcp` | Two hosts in same trust domain (LAN, VPN) | comms-dev UDS broker, retargeted |
| `wss-relay` | Globally distributed teams | comms-dev cloud hub work (#7, #16) preserved as-is |
| `gh-issues` | Async knowledge layer, public audit trail | Implements topic 03 §4B knowledge layer |
| `e2e-encrypted-wss` | Untrusted relay | wss-relay + per-pair X25519 |

The `local-fs` plugin is the *existence proof*: same daemon needed to turn the macOS PoC into a multi-team relay. The trivial case is genuinely trivial.

## Substrate requirement

The pattern is enabled by [`references/inbox-file-write-as-wake-mechanism.md`](../references/inbox-file-write-as-wake-mechanism.md): because the harness wakes members on inbox-file-write (not on process-state), anything that can produce JSON for an inbox or consume JSON from one becomes a teammate. If the wake mechanism were process-based, ghost-members could not exist without harness modification.

This is the substrate property the abstraction sits on. Changes to the harness wake mechanism would invalidate the abstraction shape, not just one plugin.

## Beyond bidirectional inter-team messaging

The pattern is broader than two teams exchanging messages. A ghost is a `members[]` entry; what's behind it can be anything that produces or consumes inbox events:

| Shape | Direction | Example |
|---|---|---|
| Read-only subscription | inbound only | `prod-alerts` ghost — daemon pipes PagerDuty/Sentry/Grafana alerts into the team's inbox |
| Write-only emitter | outbound only | `metrics-out` ghost — daemon consumes outbound messages and pushes to a dashboard or log stream |
| Webhook receiver | inbound only | `gh-webhook`, `stripe-webhook` ghosts — external webhooks land as teammate notifications |
| Transform / router | bidirectional, with translation | A ghost-pair whose daemon translates between protocols (e.g., natural-language one side, structured tool calls the other) |
| Cross-LLM bridge | bidirectional | A ghost backed by a non-Claude LLM; daemon adapts to OpenAI/Gemini/local-Ollama and writes response back |
| Human-as-ghost | bidirectional | `chat.py` is the existence proof — humans participate without harness modification |
| Knowledge query / library | bidirectional, async | A library team agent exposes itself as a ghost; specialists send questions; library returns wiki excerpts |
| Inter-team relay | bidirectional | The original motivation — pluggable transport per pairing |

The contribution: **the ghost member is a universal integration surface for the team**, of which inter-team comms is one instance. The transport catalog is the *plumbing* for one direction of integration; the ghost member is the *seam* that exposes it as a first-class participant. Same daemon-management infrastructure (lifecycle, registration, ACL, error reporting) covers alerts, webhooks, external LLMs, and inter-team comms — they're all ghost members with different plugin behavior.

## Properties that follow

**Per-pair policy negotiation.** Today's framework design picks one transport for the whole framework. With ghost-pairs, each link is independent:

- Team A ↔ Team B (internal, same host) → `local-fs`
- Team A ↔ Team C (external partner) → `e2e-encrypted-wss`
- Team A ↔ web-chat user → `wss-relay-with-webauthn`

No global decision; no transport churn when one pair's requirements change.

**Bug-class structurally eliminated.** Several bugs in comms-dev (#5: InboxWatcher deletes on handler error; race between `comms-watch --consume` and `SendMessageBridge`) are **file-protocol bugs**, not file-substrate bugs. If the ghost-relay framework owns the inbox-file protocol (append-only JSON array; mark-read-in-place; one canonical reader; one canonical writer; atomic rename) and exposes a clean plugin API, those bugs cannot exist in any transport implementation.

**Trust escalation goes away for web users.** Today's web-chat design assumes the relay must hold `COMMS_PSK` to decrypt agent messages. Under ghost-member reframe, a browser user is a registered external member of the team; the transport between the team's `ghost-to-web-user-X` and the browser is `wss-relay-with-webauthn` with crypto private to that link. The relay becomes pure bytes-over-WAN.

**Testing simplifies.** CI for inter-team semantics runs against `local-fs` — no Caddy, no D1, no Cloudflare account, no WebAuthn. Transport-specific integration tests run separately.

**Dynamic membership.** RFC #66 reports the harness rereads `config.json` on every SendMessage call (no internal cache observed up to ~seconds). A relay daemon can register/deregister ghost members at runtime — useful for direct-link lifecycle, manager-agent on-demand routing, ad-hoc pairings.

## How this composes with FR's existing patterns

- **[`multi-provider-integration-seams.md`](multi-provider-integration-seams.md)** — names peer, daemon/sidecar, MCP server as integration seam choices. The ghost-member pattern adds a *fourth* seam: ghost-as-first-class-teammate. The daemon/sidecar approach (Eilama) is closely related — both are sidecars — but the ghost-member shape makes the integration visible to agents as a teammate, not as an external service. The seam choice criterion (interface complexity) still applies; ghost-member is the seam for "must appear as a teammate to the agent."
- **[`framework-participating-vs-service-roles.md`](framework-participating-vs-service-roles.md)** — distinguishes roles using SendMessage/Librarian/authority (Claude-only) from roles with test-gated I/O (provider-agnostic). Ghost-members are a *new class*: roles using SendMessage (so they look Claude-only from the team's view) but provider-agnostic at the implementation layer. The teammate-as-interface property bridges what was a hard provider-coupling boundary.

## What this preserves (RFC #66 inventory)

| Component | Status under ghost-member pattern |
|---|---|
| `src/crypto/*` | 100% reuse — moves into `e2e-encrypted-wss` plugin |
| `src/types.ts` (envelope, framing) | 90% reuse |
| `src/broker/sendmessage-bridge.ts` | 100% reuse — IS the local-fs delivery primitive |
| `src/broker/inbox.ts` | 100% reuse |
| Cloud relay (#7) | Becomes `wss-relay` plugin |
| Web frontend (#8) | Becomes `wss-relay-with-webauthn` plugin + browser ghost-member client |
| MCP tools (#34) | Stay; expose plugin selection |
| Direct-link registry (Protocol 2) | Add `transport` column |
| Topics 1, 2, 3, 5 protocols | Unchanged — they're governance, orthogonal to mechanism |

The `comms-watch --consume` path is the one component that goes away; its formatting role is better served by an external ghost member that polls its own inbox (chat.py demonstration).

## Confidence

**Medium.** The pattern is a coherent design proposal with empirical substrate findings (RFC #66 Findings 1-3). The local-fs plugin is plausible-at-~30-LOC but unimplemented; the other plugins are sketched but unproven. The RFC frames itself as a proposal for discussion, with several open questions still unresolved (plugin negotiation protocol, member-list cache window stress-test, dead-ghost cleanup policy). Filing at `medium` reflects that the abstraction is sound enough to reason about but not yet validated by production implementation.

Confidence will upgrade to `high` on:

- First implementation of the local-fs plugin demonstrating end-to-end multi-team flow on Linux/Ubuntu (deployment-target substrate; macOS-tested mechanisms are expected POSIX-equivalent per RFC #66)
- Production migration of at least one comms-dev component into the ghost-member shape (e.g., turn the cloud relay into a `wss-relay` plugin)

## Open questions inherited from RFC #66

1. **Plugin negotiation protocol.** Both ends of a ghost-pair must agree on plugin and config. Static config in `config.json` is easy but inflexible. Should there be a discovery handshake when a new pair forms?
2. **Member-list cache window.** Harness rereads `members[]` on each SendMessage call within seconds of an edit. Not stress-tested for very-fast write-then-send sequences. Small write-vs-validate race theoretically possible.
3. **Dead ghost cleanup.** When a ghost-pair is REVOKED, remove the ghost member entry or leave it with an `inactive: true` flag? Inactive entries clutter `/list` output but preserve message history.

## Promotion posture

**n=1, watch posture.** First articulation of the pattern in RFC #66; abstraction is coherent but unproven by implementation. Promotion-candidate triggers:

- Second team independently arrives at a similar abstraction (cross-team confirmation, analogous to substrate-invariant-mismatch Instance 5's external-platform-confirmation discipline)
- First implementation of any non-trivial plugin (local-fs or wss-relay) demonstrating the abstraction's robustness

## Related

- [`references/inbox-file-write-as-wake-mechanism.md`](../references/inbox-file-write-as-wake-mechanism.md) — the substrate property this abstraction sits on. Without inbox-file-write being the wake mechanism, ghost-members could not exist as a teammate-shaped integration without harness modification.
- [`patterns/service-team-topology.md`](service-team-topology.md) — a team whose members are ghost representations of the teams it serves. Built on top of this pattern; directly addresses #47 OQs.
- [`patterns/multi-provider-integration-seams.md`](multi-provider-integration-seams.md) — names peer, daemon/sidecar, MCP server integration seams. Ghost-member is a fourth: ghost-as-first-class-teammate.
- [`patterns/framework-participating-vs-service-roles.md`](framework-participating-vs-service-roles.md) — provider-coupling boundary that ghost-members bridge.

## Source

RFC #66 (2026-05-12), <https://github.com/mitselek/ai-teams/discussions/66>. Authored on `mitselek/ai-teams` against comms-dev's existing inter-team comms work (#7, #16, #5, #8, #34). Cross-team-pollinated into FR's wiki because the abstraction has framework-level implications across topics 03 (communication), 04 (hierarchy), and the Knowledge Layer architecture in #47.

Reference artifacts:

- chat.py (external member existence proof): <https://gist.github.com/mitselek/bdc18e47fcdbf21b5ddd7922b077cc7b>
- Empirical session log: <https://gist.github.com/mitselek/afbc0909dfebcdcdc20cb5508d208456>

(*FR:Callimachus*)
