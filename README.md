# rote-skills

Distributable [Claude Code](https://claude.com/claude-code) skills for [rote](https://getrote.dev).

## Install

This repo is a **dual-format marketplace** — the same repo installs into both Claude Code
and Codex (the `SKILL.md` is shared; each agent reads its own manifest).

### Claude Code

```bash
claude plugin marketplace add modiqo/rote-skills
claude plugin install rote-setup@rote-skills
```

Or one line: `claude plugin marketplace add modiqo/rote-skills && claude plugin install rote-setup@rote-skills`

Then run it:

```
/rote-setup
```

Installed at **user scope** by default (available in every project). Use `--scope project` to
scope it to one repo.

### Codex

```bash
codex plugin marketplace add modiqo/rote-skills
```

Then, inside a Codex session, open the **`/plugins`** browser, find **rote-setup**, and
install it. (Codex installs plugins through the in-session browser rather than a CLI install
command.) After install, restart Codex and run the wizard.

## What `/rote-setup` does

A guided, interactive wizard that takes you from zero to a working rote install:

1. **Installs rote if missing** — CLI one-liner (`curl -fsSL https://getrote.dev/install | bash`)
   or a VS Code-family editor extension (VS Code Marketplace / Open VSX for Cursor &
   Antigravity), detecting which editors you actually have.
2. **Signs you in** — every experience is identity-gated, so sign-in always runs (Google or
   GitHub; branches to request-an-invite / claim-an-invite-code if you don't have an account).
3. **Forks on how far to go** — stop at just the CLI, pull curated **powerpack** adapters,
   or **build adapters from the 872-API built-in catalog** (`rote adapter catalog search` →
   `rote adapter new`). All branches run under your signed-in identity.
4. **Menu-driven setup** — install adapters à la carte from the live registry (with their
   flows), wire credentials, connect Google OAuth, install the agent skill, and explore —
   you pick what runs.
5. **Value-proof closer** — ends by running one live flow against your own data (you pick
   it, the wizard reads its parameters and asks you for values), so setup finishes with real
   output, not just "complete."

Every branch is a clear choice; it never silently runs the whole one-liner.

## Decision flow

```
                            /rote-setup
                                 │
                                 ▼
                   ┌───────────────────────────┐
                   │  rote binary installed?    │
                   │     command -v rote        │
                   └───────────────────────────┘
                       │                   │
                   no  │                   │  yes
                       ▼                   │
        ┌──────────────────────────┐      │
        │   How to install?        │      │
        ├──────────────────────────┤      │
        │ • CLI one-liner          │      │
        │ • Editor extension       │      │
        │   (VS Code / Cursor /    │      │
        │    Antigravity)          │      │
        │ • I'll do it myself      │      │
        └──────────────────────────┘      │
                       │                   │
                       └─────────┬─────────┘
                                 ▼
                   ╔═══════════════════════════╗
                   ║   SIGN IN  (mandatory)     ║   every path is
                   ║   rote whoami → login      ║   identity-gated
                   ╟───────────────────────────╢
                   ║ Have an account?           ║
                   ║  • yes → Google / GitHub   ║
                   ║  • have invite code → join ║
                   ║  • no account → request    ║
                   ╚═══════════════════════════╝
                                 │
                                 ▼  signed in as you@…
              ┌───────────────────────────┐
              │   How far do you want      │
              │   to go?                   │
              └───────────────────────────┘
          ┌───────────────┬──────────────────┐
          ▼               ▼                  ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
  │ Just the CLI │ │  Powerpack   │ │   API catalog    │
  │   — stop     │ │  (curated)   │ │  (872 specs)     │
  ├──────────────┤ ├──────────────┤ ├──────────────────┤
  │ rote how     │ │ pick from    │ │ catalog search   │
  │              │ │ live registry│ │   ↓              │
  │              │ │ à la carte   │ │ catalog info     │
  │              │ │              │ │   ↓              │
  │              │ │              │ │ adapter new <id> │
  └──────────────┘ └──────────────┘ └──────────────────┘
          │               │                  │
          │               └────────┬─────────┘
          │                        ▼   adapters installed
          │            ┌───────────────────────────┐
          │            │   Post-login menu          │
          │            │   (pick any, any order)    │
          │            ├───────────────────────────┤
          │            │ • credentials (masked)     │
          │            │ • Google OAuth             │
          │            │ • install agent skill      │
          │            │ • explore / learn          │
          │            └───────────────────────────┘
          │                        │
          │                        ▼
          │            ┌───────────────────────────┐
          │            │   Prove value:             │
          │            │   run one live flow        │
          │            │   on your own data         │
          │            └───────────────────────────┘
          │                        │
          └───────────┬────────────┘
                      ▼
                 ✦  done  ✦
```

Legend: `╔═╗` = always runs (identity gate) · `┌─┐` = a choice you make ·
`✦ done ✦` = clean exit. **Just the CLI** ends right away; **Powerpack** and
**API catalog** both flow on to credentials → skill install → a live-flow
finale. Every `•` is an AskUserQuestion option — the wizard pauses for you at
each fork and never runs the whole thing silently.

## Updating

```bash
claude plugin update rote-setup@rote-skills
```

## Skills in this marketplace

| Skill | Command | Description |
|---|---|---|
| `rote-setup` | `/rote-setup` | Guided first-run setup for rote |

## License

MIT
