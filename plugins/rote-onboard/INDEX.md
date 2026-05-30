# rote-onboard — skill map

Five guided slash-command skills that walk a human through getting rote working, sharing what
they build, and keeping it all current. They're a sequence, not a grab-bag — each ends by
naming the next logical step, so the path becomes muscle memory. This file is the canonical
map; each skill links back here.

## The skills, in the order you meet them

| Order | Slash command | Use it when | Hands off to |
|------:|---------------|-------------|--------------|
| 1 | `/rote-onboard:rote-setup` | First run — install the binary, sign in, pull/​build your first adapters, prove value with one live flow. | adapter-create (build more) · main **rote** skill (daily use) |
| 2 | `/rote-onboard:rote-adapter-create` | Add an adapter for any API — dry-run-first, auth researched from the provider's docs. | adapter-config (tune it) · registry (share it) |
| 3 | `/rote-onboard:rote-adapter-config` | Tune an adapter you already have — auth, base URL, write guard, sensitivity, capability index, OAuth re-auth, grouping. | back to itself (menu) · adapter-create (new one) |
| 4 | `/rote-onboard:rote-registry` | Share an artifact — at the moment an adapter is minted or a flow crystallized, check if it's already in your orgs, push at chosen visibility, then surface members + invite. Also `/show usage` (quota across orgs). | rote-org (full admin) |
| 5 | `/rote-onboard:rote-update` | Keep current — update both layers (the `rote` binary **and** these skills), which version independently. | — |

**The day-to-day skill is separate:** once you're set up, the main **rote** skill takes over
(`rote flow search "<intent>"` before any direct adapter call). The onboard skills are for
*changing your setup*; the rote skill is for *using* it.

## How to build muscle memory

- **Discovery:** type `/rote` in the agent — the `rote-onboard:*` skills appear with their
  descriptions. This file gives them an order; the menu alone doesn't.
- **The chain is the teacher:** every skill ends by naming the next step in context ("you just
  made an adapter → `rote-adapter-config` tunes it"). Following the chain a few times is what
  makes it stick — you stop needing this table.
- **One entry point to remember:** `/rote-onboard:rote-setup` is the front door. From a clean
  machine it routes to everything else.

## Shared closing-line convention (dry humor)

All skills end a **clean** run on one dry one-liner, keyed to what actually just happened.
The running joke: rote wired you straight to the provider's own API — no metered middleman
proxy billing per call, no extra network hop forwarding your request for a markup.

Rules (identical across all skills — keep them consistent):
- **Grounded in this run's facts**, never a canned template — the flow name, adapter count, or
  tool count from *this* session.
- **Honest claims only.** "Per-call metering" and "an extra network hop" are the safe, true
  jabs. Never invent a dollar figure or name a competitor's pricing you didn't verify.
- **Read the room.** Skip the quip if the run was rocky (login failed, a step errored, a probe
  401'd, `self-update` deferred). A victory lap after a half-finished run reads as oblivious.
- **One line.** If it needs a second sentence to explain the joke, cut it.

Per-skill hook (so it lands on different facts each time, not a hardcoded tagline):
- **setup** → the live flow that just proved value
- **adapter-create** → the `total_tools` count just indexed
- **adapter-config** → whatever was just tuned (lower-key; config is housekeeping)
- **registry** → the artifact + org it just landed in, now shareable without a registry tax
- **update** → the version just landed on
