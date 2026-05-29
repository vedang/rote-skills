---
name: rote-setup
description: >
  Guided, interactive first-run setup for rote. Walks a user from zero to a working rote
  install — detects whether they already have an account, signs them in (Google/GitHub),
  or branches to request-an-invite / claim-an-invite-code, then offers a menu of the
  remaining onboarding steps (powerpack adapters, credentials, OAuth, install skill,
  explore). Use when the user says "set up rote", "rote setup", "onboard me to rote",
  "get rote working", "install and configure rote", "/rote-setup", or is a first-timer
  who needs hand-holding through `rote join` / `rote login` / `rote pull powerpack`.
  This skill drives the human through choices with AskUserQuestion at each branch —
  it does NOT silently run the whole one-liner.
---

# rote-setup — Guided onboarding wizard

Make installation feel like a breeze. You are the wizard: detect state, present clear
choices, run one step at a time, confirm, move on. Lead with the happy path (sign in),
fall back gracefully (request invite / claim code) only when needed.

Use **AskUserQuestion** for every branch below. Run **one command per Bash call** (the user
mandates no chained/piped commands). After each step, give a one-line "what just happened"
and surface the next choice.

All commands are non-destructive to the filesystem; they touch `~/.rote/` config and the
registry.

---

## Step -1 — Is rote installed? (pre-flight gate)

Before anything else, confirm the binary exists:

```bash
command -v rote
```

- **Prints a path** (e.g. `/Users/.../.local/bin/rote`) → installed. Continue to **Step 0**.
- **Empty / exit code 1** → not installed. Offer the install choice via AskUserQuestion,
  header `Install`:

  - **CLI one-liner** (recommended) — installs the `rote` command directly:
    ```bash
    curl -fsSL https://getrote.dev/install | bash
    ```
    This is a piped command — the user mandates no chaining, but this is the canonical
    vendor installer and runs as a single logical step, so it's the one allowed exception.
    Tell the user exactly what it does (downloads + runs the rote install script) and get
    explicit confirmation before running it, since piping curl to bash executes remote code.
  - **Editor extension** — installs the rote extension into a VS Code-family editor
    (bundles the CLI + panels). Go to **Step -1a (detect editors)** to pick which one.
  - **I'll install it myself** — pause and tell the user to install, then re-run `/rote-setup`.

### Step -1a — Detect installed editors and install the extension

The rote extension ships to two registries: the **VS Code Marketplace** (VS Code only)
and **Open VSX** (the open registry that Cursor, Antigravity, and other VS Code forks pull
from). Detect which editors are actually installed before offering them — never present an
editor the user doesn't have.

Probe each editor's CLI (and fall back to the macOS app bundle):

```bash
command -v code
```
```bash
command -v cursor
```
```bash
command -v antigravity
```

If a `command -v` is empty, fall back to checking the app bundle on macOS, e.g.:

```bash
ls -d "/Applications/Visual Studio Code.app" "/Applications/Cursor.app" "/Applications/Antigravity.app" 2>/dev/null
```

Build the AskUserQuestion option list from **only the editors that resolved** (header
`Editor`). For each:

| Editor | CLI | Install command | Registry |
|---|---|---|---|
| **VS Code** | `code` | `code --install-extension Modiqo.rote` | VS Code Marketplace |
| **Cursor** | `cursor` | `cursor --install-extension Modiqo.rote` | Open VSX |
| **Antigravity** | `antigravity` | `antigravity --install-extension Modiqo.rote` | Open VSX |

Run the chosen editor's install command as its own Bash call, e.g.:

```bash
code --install-extension Modiqo.rote
```

If the editor's CLI isn't on PATH but the app bundle exists, point the user at the
marketplace page to install manually instead:
- VS Code → **https://marketplace.visualstudio.com/items?itemName=Modiqo.rote**
- Cursor / Antigravity (Open VSX) → **https://open-vsx.org/extension/Modiqo/rote**

If **none** of the three editors are detected, say so and steer the user to the **CLI
one-liner** instead (or installing an editor first). Don't offer an editor that isn't there.

After install, the extension's setup wizard provisions the CLI on first launch — tell the
user to reload/open the editor, then re-verify the binary with `command -v rote` before
continuing to Step 0.

  After the CLI one-liner finishes, the binary may not be on the current shell's PATH yet
  (the installer typically drops it in `~/.local/bin`). Re-verify before continuing:

  ```bash
  command -v rote
  ```

  If still not found, check `~/.local/bin/rote` directly and tell the user to open a new
  terminal / `source` their shell profile (or add `~/.local/bin` to PATH). Do not proceed
  to Step 0 until `command -v rote` resolves.

---

## Step 0 — Detect current state (always run first, silently)

```bash
rote whoami
```

Read the output:
- `ok: <email>` → **already logged in.** Skip straight to **Step 3 (post-login menu)**.
  Open with: "You're already signed in as `<email>`." Then present the menu.
- Anything else (error, `not logged in`, empty) → **not logged in.** Go to **Step 1**.

Do not assume — branch on the actual output of `rote whoami`.

---

## Step 1 — Do you already have an account?

Ask up front (per design): **"Do you already have a rote account?"**

Present with AskUserQuestion, header `Account`:
- **Yes, sign me in** — proceed to **Step 2 (sign-in)**.
- **I have an invite code** — proceed to **Step 1a (claim code)**.
- **No, I need an invite** — proceed to **Step 1b (request invite)**.

### Step 1a — Claim an invite code

Ask for the code (AskUserQuestion is for choices, so just ask in prose: "Paste your invite
code"). When you have it:

```bash
rote join <invite-code>
```

On success → continue to **Step 2 (sign-in)** with the email tied to that invite.
On failure → show the error verbatim, offer to retry or fall back to **Step 1b**.

### Step 1b — Request an invite

rote is invite-gated. Point the user at the signup/invite request flow:

> rote access is invite-based. Request one at **https://getrote.dev** (or ask whoever
> invited you for a code). Once you have a code, come back and run `/rote-setup` again —
> I'll pick up at the claim-code step.

Confirm whether they want to (a) open the link / request now and pause, or (b) already
have a code after all (loop back to **Step 1a**). Do not fabricate an invite URL beyond
the canonical `getrote.dev`; if unsure, tell them to check the email/Slack where they
first heard about rote.

---

## Step 2 — Sign in (detect configured providers)

Present the sign-in options. The CLI supports `--provider google`, `--provider github`,
and `--email <addr>`. Show provider choices via AskUserQuestion, header `Sign in`:

- **Google** → `rote login --provider google`
- **GitHub** → `rote login --provider github`
- **Email link** → ask for the address, then `rote login --email <addr>`

Run the chosen one as its own Bash call:

```bash
rote login --provider google
```

This opens a browser for OAuth. Tell the user to complete it in the browser, then
re-verify:

```bash
rote whoami
```

When `whoami` returns `ok: <email>`, say "Signed in as `<email>`." and go to **Step 3**.
If it still fails, show the login output and offer to retry with a different provider.

---

## Step 3 — Post-login menu (menu-driven)

Now logged in. Present a **menu** of remaining setup steps (per design — let the user
pick, don't force the full sequence). Use AskUserQuestion with `multiSelect: true`,
header `Setup`, so they can queue several:

| Option | What it does | Command |
|---|---|---|
| **Pull powerpack** (recommended) | Curated starter adapters (github, gmail, calendar, linear, notion). | `rote pull powerpack` (interactive) — or with a preset, see 3a |
| **Set up credentials** | Interactive token wizard for installed adapters. | `rote powerpack credentials` |
| **Connect Google (OAuth)** | Browser OAuth for Google adapters (gmail/calendar). | `rote oauth setup google` |
| **Install the agent skill** | Lets your coding agent (Claude/Cursor/Codex) use rote. | `rote install skill` |
| **Explore / learn** | Onboarding tree + human-friendly help. | `rote how`, then `rote human` |

Recommend the order **powerpack → credentials → OAuth → install skill → explore**, but run
only what they pick, one at a time, confirming after each.

### Step 3a — Powerpack preset choice

If they pick "Pull powerpack", first offer a preset (AskUserQuestion, header `Powerpack`):

- **Default** (recommended) — github, gmail, calendar, linear, notion (5 adapters):
  `rote pull powerpack --with-flows`
- **Dev** — dev-focused adapters: `rote pull powerpack --preset dev --with-flows`
- **Google** — google-only: `rote pull powerpack --preset google --with-flows`
- **AI** — gemini-api: `rote pull powerpack --preset ai --with-flows`
- **All** — everything: `rote pull powerpack --all --with-flows`

Include `--with-flows` so they get crystallized flows too. Add `--yes` only if the user
asks to skip the interactive picker; otherwise let powerpack prompt for adapter selection.

### Step 3b — Install skill provider

If they pick "Install the agent skill", ask which agent (AskUserQuestion, header
`Agent`): **Claude** (default), **Cursor**, **Codex**, or **All**. Then:

```bash
rote install skill --provider claude
```

(Use `--provider all` for "All".) `--force` overwrites an existing SKILL.md without
prompting — only add it if the user confirms an overwrite.

---

## Step 4 — Wrap up

After the menu items they chose are done, confirm and orient:

```bash
rote how
```

Summarize what's now set up (signed-in email, adapters pulled, credentials wired, skill
installed) and point at `rote human` for friendly help and `rote how` for the agent
onboarding tree. Offer to crystallize a first flow or run `rote flow search "<intent>"`.

---

## Behavior notes

- **One command per Bash call.** No `&&`, `|`, or `;` chains. The user wants each
  success/failure visible on its own. The single exception is the canonical vendor
  installer `curl -fsSL https://getrote.dev/install | bash`, which is one logical step —
  and only after explicit user confirmation, since it pipes remote code into bash.
- **Detect before offering.** Confirm the rote binary (`command -v rote`) before any rote
  command, and detect installed editors (`command -v code|cursor|antigravity`) before
  offering the extension path. Never present an install target that isn't there.
- **Detect, don't assume.** Once rote is present, start from `rote whoami`. Branch on real
  output.
- **Lead with the happy path.** Sign-in first; invite/claim flows are the fallback.
- **Browser steps are async.** After `rote login --provider ...` or `rote oauth setup
  google`, tell the user to finish in the browser, then re-run `rote whoami` /
  re-check before declaring success.
- **Show errors verbatim.** Never paper over a failed step; offer a retry or alternate
  branch.
- **Don't fabricate URLs or codes.** Canonical signup is `getrote.dev`; invite codes come
  from the user.
- After a successful first run, suggest the main **rote** skill takes over for day-to-day
  use (`rote flow search` before any direct adapter call).
