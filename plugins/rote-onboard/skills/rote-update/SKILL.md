---
name: rote-update
description: >
  Update rote to the latest version — both the binary and the skill distribution. Runs
  `rote self-update`, refreshes adapter subagent templates, and updates the rote-skills plugin
  for the current agent (Claude Code or Codex). Use when the user says "update rote", "upgrade
  rote", "is there a new rote version", "/rote-update". Quick and guided — covers the two
  layers (binary + skills) people forget to do together.
---

# rote-update — keep rote current

Two layers most people forget to update together: the **binary** and the **skill/plugin
distribution**. This skill does both. Determine facts from live commands; one command per Bash
call. If rote isn't on PATH, resolve its absolute path.

---

## 1. Check first (optional)

```bash
rote self-update --check
```
Shows whether a newer version is available without installing. If already current, say so and
skip to step 3 (the skills can still drift).

## 2. Update the binary

```bash
rote self-update --yes
```
`--yes` skips the confirmation prompt (the agent shell has no TTY). After it completes, confirm:

```bash
rote --version
```

**After a binary upgrade, refresh the adapter subagent templates** — existing
`~/.rote/adapters/<id>/agent.md` files are not auto-overwritten on upgrade (per rote's own
post-upgrade guidance):

```bash
rote adapter agent generate --all --force
```

## 3. Update the rote-skills plugin (this skill set)

The wizard skills are distributed via the plugin marketplace — update them for the agent in
use:

**Claude Code:**
```bash
claude plugin update rote-setup@rote-skills
```

**Codex:**
```bash
codex plugin marketplace upgrade rote-skills
```
Then reinstall from the in-session `/plugins` browser if prompted. (Codex updates the
marketplace snapshot; the plugin re-installs from it.)

Both note "restart to apply" — tell the user to restart their agent session to load the
updated skills.

---

## Notes

- `rote self-update` updates the `rote` binary in place (typically `~/.local/bin/rote`).
- If `self-update` reports the binary was installed via a package manager (brew) or an editor
  extension, follow its guidance — it may defer to that installer rather than self-replacing.
- The binary and the skills version independently; updating one doesn't update the other —
  that's why this skill does both.
