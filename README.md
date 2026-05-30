# rote-skills

Distributable [Claude Code](https://claude.com/claude-code) skills for [rote](https://getrote.dev).

## Install

In any Claude Code session (or your terminal), add the marketplace and install the skill:

```bash
claude plugin marketplace add modiqo/rote-skills
claude plugin install rote-setup@rote-skills
```

Or as a single copy-paste line:

```bash
claude plugin marketplace add modiqo/rote-skills && claude plugin install rote-setup@rote-skills
```

Then run it:

```
/rote-setup
```

The skill is installed at **user scope** by default, so `/rote-setup` is available in every
project. Use `--scope project` on the install command to scope it to one repo instead.

## What `/rote-setup` does

A guided, interactive wizard that takes you from zero to a working rote install:

1. **Installs rote if missing** — CLI one-liner (`curl -fsSL https://getrote.dev/install | bash`)
   or a VS Code-family editor extension (VS Code Marketplace / Open VSX for Cursor &
   Antigravity), detecting which editors you actually have.
2. **Signs you in** — detects whether you already have an account, then signs in via Google
   or GitHub, or branches to request-an-invite / claim-an-invite-code.
3. **Menu-driven setup** — install adapters à la carte from the live registry (with their
   flows), wire credentials, connect Google OAuth, install the agent skill, and explore —
   you pick what runs.
4. **Value-proof closer** — ends by running one live flow against your own data (you pick
   it, the wizard reads its parameters and asks you for values), so setup finishes with real
   output, not just "complete."

Every branch is a clear choice; it never silently runs the whole one-liner.

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
