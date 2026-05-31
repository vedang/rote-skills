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

## Shared operating rules (apply in EVERY skill)

These are load-bearing — a skill that ignores them fumbles the run. Each SKILL.md references
this section; keep them identical across skills.

### 1. Permissions — clear the prompts up front

Every `rote …` / `cd …` / `rote deno …` call goes through the agent's Bash tool, which prompts
for permission unless allowlisted. On a fresh machine that means a prompt on *every step* —
death by a thousand confirmations. **At the very start of any skill, before the first rote
command, tell the user to allowlist these** (or do it via the `update-config` skill if
available):

There are **two independent permission layers** — you need both, or you'll still be prompted:

**(a) Bash-command allowlist** — `permissions.allow`:
- `Bash(rote:*)` — all rote commands (this already covers `rote deno run …`, so a separate
  deno rule is redundant; include it only as an explicit teaching aid if you like)
- `Bash(cd:*)` — the `cd <dir> && rote …` compounds used for workspace/flow execution

**(b) Directory-access allowlist** — `permissions.additionalDirectories` (THIS is the one
most people miss). The skills `cd` into several paths **under `~/.rote/` — all outside the
user's project directory**: `~/.rote/rote/workspaces/<name>` (flow runs / probes),
`~/.rote/flows/<org>/<name>` (deno flow execution), and they read `~/.rote/adapters/…`. Each
`cd` outside the project triggers a *separate* filesystem prompt ("allow reading from <dir> …")
that the Bash allowlist does **not** cover. The simplest cover-all is to add the whole
`~/.rote` tree:
- **`~/.rote`** — covers workspaces, flows, adapters, tokens — everything the skills touch by
  path. (Use a narrower `~/.rote/rote/workspaces` + `~/.rote/flows` pair if you want to scope
  it tighter, but `~/.rote` is the one-entry answer.)

So the complete `~/.claude/settings.json` shape is:

```json
{
  "permissions": {
    "allow": ["Bash(rote:*)", "Bash(cd:*)"],
    "additionalDirectories": ["~/.rote"]
  }
}
```

Offer to add **both** layers once at the start; don't re-ask. Without `additionalDirectories`,
every flow run / probe inside a workspace re-prompts even though the Bash rules are present —
the exact bug seen on a fresh machine. (Codex: the equivalent is its approved-commands /
sandbox-writable-roots config.)

### 1b. Resolving the rote binary — narrow probe, NEVER a deep search

If `command -v rote` is empty, rote may be installed but off this shell's PATH. Check the
**two known install locations directly** — do **NOT** run `find ~` / a recursive search of the
home directory (it's slow, triggers a scary read-everything permission prompt, and is exactly
the overreach to avoid):

```bash
ls -la "$HOME/.local/bin/rote" "$HOME/.cargo/bin/rote" 2>/dev/null
```

- Either path exists → use that **absolute path** for every later `rote` command in the run.
- Neither exists **and** `command -v rote` was empty → the binary is genuinely **not
  installed** (route to install, or tell the user). Do not go hunting for it elsewhere.

### 2. Step-wise, NEVER parallel

Run **one command per Bash call, strictly in sequence** — never batch independent probes into a
single parallel tool block, and never fire the next step before the current one's result is in.
Each step's output gates the next (an empty `whoami`, a 401 on a dry-run, an unset token all
change what comes next). Parallel probing skips that gating and the run fumbles — the exact bug
seen on a fresh machine. If you catch yourself about to send two Bash calls at once: don't.

### 3. Required-state gates block the next step

When a step establishes a precondition the next step needs — a token set, a login succeeded, a
dry-run passed — **verify it actually happened before proceeding.** Do not advance on an
adapter whose required credential is unset, a login that still reports "not logged in", or a
create whose dry-run errored. Re-read the real output; if the precondition isn't met, stop and
resolve it (or ask the user) rather than marching to the next step on a broken foundation.

### 4. Running a flow — explore, read frontmatter, run the right way

There are two execution modes and you must pick by the flow's frontmatter:

1. **Find the flow** for the intent (don't guess the name):
   `rote explore "<intent>"` (ranks installed adapters/flows) and/or `rote flow search "<q>"`.
2. **Read the flow's frontmatter** before running it: look at
   `~/.rote/flows/<org>/<name>/main.ts`. Does it have a `steps:` block?
   - **Has `steps:` (DAG flow)** → `rote flow run <name> key=value …` works (needs a workspace cwd).
   - **No `steps:` (legacy/sequential)** → `rote flow run` may misfire (it can fall back to a
     bash invocation instead of Deno). Run it the reliable way instead:
     ```bash
     cd ~/.rote/flows/<org>/<name> && rote deno run --allow-all main.ts [args…]
     ```
     `rote deno run` uses the bundled Deno at `~/.rote/bin/deno`. Pass the flow's positional
     args (read them from the frontmatter `parameters:`). This `cd && rote deno run` compound is
     one logical step (see the `Bash(cd:*)` allowlist above).

When unsure which mode, prefer the `cd … && rote deno run` form — it works for both and matches
how the flows are authored.

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
