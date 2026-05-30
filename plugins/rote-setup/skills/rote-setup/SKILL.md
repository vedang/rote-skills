---
name: rote-setup
description: >
  Guided, interactive first-run setup for rote. Installs the binary, signs the user in via
  Google/GitHub (every experience is identity-gated; branches to request-an-invite /
  claim-an-invite-code if needed), then forks: stop at just the CLI, pull curated powerpack
  adapters, or build adapters from the 872-API built-in catalog. Then offers a menu of
  remaining onboarding steps (adapters, credentials, OAuth, install skill, explore) and ends
  by running one live flow to prove value. Use when the user says "set up rote", "rote
  setup", "onboard me to rote",
  "get rote working", "install and configure rote", "/rote-setup", or is a first-timer
  who needs hand-holding through `rote join` / `rote login` / installing adapters.
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

**Self-contained — never rely on memory or prior session state.** This skill must work on a
fresh machine for a user the agent has never seen. Do **not** consult project memory,
recalled facts, CLAUDE.md, or any "rote is active here" assumption to decide whether rote is
installed, where it lives, or whether the user is logged in. Determine every fact by running
the probe commands in the steps below and branching on their **actual output** — memory may
be stale, machine-specific, or simply wrong. If a recalled note contradicts a live probe,
the live probe wins.

---

## Step -1 — Is rote installed? (pre-flight gate)

Determine this **only** from live probes — never from memory or a "rote is active here"
note. Two probes, in order:

1. Is `rote` on PATH?
   ```bash
   command -v rote
   ```
   If it prints a path, rote is installed and invocable as `rote`. **Use that name for all
   later commands.**

2. If step 1 is empty (exit 1), it may be installed but **not on this shell's PATH** (the
   installer commonly drops it in `~/.local/bin`, which a non-interactive shell often omits).
   Probe the known install locations directly before concluding it's missing:
   ```bash
   ls -la "$HOME/.local/bin/rote" "$HOME/.cargo/bin/rote" 2>/dev/null
   ```
   - If either path exists → rote **is** installed, just off PATH. Use the **absolute path**
     (e.g. `$HOME/.local/bin/rote`) for every later `rote` command in this run, and tell the
     user their shell PATH is missing `~/.local/bin` (they can add it / open a new terminal).
   - If neither exists **and** `command -v rote` was empty → genuinely **not installed**.

Branch on the combined result:

- **Installed** (on PATH or found at a known path) → go to **Step 0 (login)**. Every
  experience is identity-gated, so sign-in always runs before the fork.
- **Not installed** (both probes empty) → offer the install choice via AskUserQuestion,
  header `Install`:

**Command convention for the rest of this run:** every step below writes `rote …`. If Step
-1 resolved rote on PATH, run it verbatim. If it resolved only at an absolute path (off
PATH), substitute that path for the leading `rote` in **every** command (e.g.
`$HOME/.local/bin/rote whoami`). Pick this once at Step -1 and stay consistent.

  - **CLI one-liner** (recommended) — installs the `rote` command directly. **Always run
    it non-interactively** — the installer prompts ("Install Deno runtime? [Y/n]") and a
    plain `curl … | bash` has no TTY in an agent shell, so it dies on `/dev/tty: Device not
    configured`. Set `ROTE_YES=1` (the installer's documented non-interactive switch) and
    invoke via `bash -c` so the env var is in scope:
    ```bash
    ROTE_YES=1 bash -c "$(curl -fsSL https://getrote.dev/install)"
    ```
    This is the one allowed piped/compound command — the canonical vendor installer as a
    single logical step. Tell the user exactly what it does (downloads + runs the rote
    install script, auto-accepting the runtime prompts) and get explicit confirmation
    before running it, since it executes remote code. Other useful env vars: `ROTE_RESET=1`
    (re-run all steps from scratch), `ROTE_FULL=1` (also install the browser runtime).
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
continuing to **Step 0 (login)**.

  After the CLI one-liner finishes, the binary may not be on the current shell's PATH yet
  (the installer typically drops it in `~/.local/bin`). Re-verify before continuing:

  ```bash
  command -v rote
  ```

  If still not found, check `~/.local/bin/rote` directly and tell the user to open a new
  terminal / `source` their shell profile (or add `~/.local/bin` to PATH). Do not proceed
  to Step 0 (login) until `command -v rote` resolves.

---

## Step 0 — Detect login state (always — login is mandatory)

Login runs for **every** user, right after the binary is confirmed installed — the whole
experience is identity-gated so usage is attributable. Detect current login state silently:

```bash
rote whoami
```

Read the output:
- `ok: <email>` → **already logged in.** Say "You're signed in as `<email>`." and go to the
  fork (**Step 2.5**).
- Anything else (error, `not logged in`, empty) → **not logged in.** Go to **Step 1** to
  sign in. Sign-in is required before the fork — do not offer to skip it.

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

When `whoami` returns `ok: <email>`, say "Signed in as `<email>`." and go to the fork
(**Step 2.5**). If it still fails, show the login output and offer to retry with a different
provider.

---

## Step 2.5 — How far do you want to go? (fork, after login)

The binary is installed **and the user is signed in** (login in Steps 0–2 always runs first —
every experience is identity-gated, so usage is attributable). Now ask how far they want to
take setup. AskUserQuestion, header `Scope`:

- **Just the CLI — stop here** — confirm rote is installed and signed in, print `rote how`
  for next steps, and **end the wizard cleanly**. No adapters. They can re-run `/rote-setup`
  anytime to go further.
- **Pull the powerpack** (curated starter adapters) — github, gmail, calendar, linear from
  the registry. Go to the powerpack picker in **Step 3a**.
- **Build from the API catalog** (search 872 built-in API specs) — pick any API (notion,
  stripe, datadog, …) and create an adapter from its spec. Go to **Step 2.6 (catalog flow)**.

Login is **not** part of this fork — it has already happened. All three branches run under
the signed-in identity; do not skip or defer sign-in for any of them.

### Step 2.6 — Build an adapter from the API catalog

The catalog is a binary-embedded set of ~872 validated API specs. Search → inspect → create.

1. **Ask what API they want** (prose — free text): "Which API? (e.g. notion, stripe,
   datadog, or a keyword like 'email')". Then search:
   ```bash
   rote adapter catalog search "notion"
   ```
   This lists matches as `ID · Category · Provider`. If zero matches, suggest
   `rote adapter catalog list` (browse categories) or a broader keyword.

2. **Show details for the picked entry** so the user sees auth type, token page, and notes
   (some entries carry install hints, e.g. DCR for Notion):
   ```bash
   rote adapter catalog info notion-mcp
   ```
   Surface the `Auth`, `Token Page`, and `Notes` lines — they tell the user what credential
   they'll need and how it's obtained.

3. **Confirm, then create — pick the right command + non-interactive flag.** Two creation
   commands exist, and they take **different** non-interactive flags. Read the catalog
   `info` output (step 2): the **Spec Type** / **Notes** tell you which one.

   - **REST / OpenAPI / GraphQL / Discovery spec** → `rote adapter new <id>`. Its
     non-interactive flag is **`--yes`**:
     ```bash
     rote adapter new stripe --yes
     ```
   - **MCP server** (Spec Type "MCP" — e.g. Notion, PostHog, Supabase; the Notes often say
     "install via: rote adapter new-from-mcp …") → `rote adapter new-from-mcp <id> <url>`.
     Its non-interactive flag is **`--headless`**, **not `--yes`** (`new-from-mcp` rejects
     `--yes`):
     ```bash
     rote adapter new-from-mcp notion https://mcp.notion.com/mcp --headless
     ```

   Notes: `rote adapter new <id>` resolves the spec from the catalog; a provider-side
   `HTTP 401` means the API needs auth even to read its spec (rare). MCP creation runs the
   server's auth flow during creation — for OAuth-DCR servers (Notion) it **opens a browser**
   right then; tell the user to complete it. `--headless` only skips the interactive
   *tool-selection* wizard (includes all tools); it does not skip the browser auth.

   **After creating an OAuth-DCR MCP adapter**, re-authorization later is a browser flow:
   `rote adapter reauth <id>` (add `--force-reregister` only if the provider pruned the
   client). There is no pasted token for these.

4. **Optional — prove the new adapter works.** A probe call needs a **workspace cwd** (see
   "Running a workspace-scoped command" in Behavior notes). Create a workspace and probe in
   the same Bash call — never run `rote <adapter>_probe` from a scratch dir (it fails with
   `not in a workspace directory`):
   ```bash
   rote init proof --seq --force
   ```
   ```bash
   cd ~/.rote/rote/workspaces/proof && rote <adapter>_probe "<query>"
   ```
   For an OAuth-DCR adapter the browser auth from creation must have completed first.

5. **Loop or move on.** Offer to add another (`search` again) or proceed to credentials
   (Step 3c) for the adapter(s) just created, then the install-skill step and the value
   closer (Step 5).

After the catalog branch, jump to **Step 3c** (credentials) for the new adapters, then the
install-skill step and the value closer.

> **Future direction (not yet wired into this skill):** an adapter created from the catalog
> should be **pushed to the user's private org hub** so the team shares it —
> `rote registry adapter push <adapter-path> <org-slug>`. When that lands, this step will
> offer "share this adapter with your org" after a successful `rote adapter new`, gated on
> the signed-in identity and org membership. For now, created adapters stay local.

---

## Step 3 — Post-login menu (menu-driven)

Now logged in. Present a **menu** of remaining setup steps (per design — let the user
pick, don't force the full sequence). Use AskUserQuestion with `multiSelect: true`,
header `Setup`, so they can queue several:

| Option | What it does | Command |
|---|---|---|
| **Pull powerpack adapters** (recommended) | Curated registry adapters, à la carte. **See 3a.** | `rote registry adapter pull bootstrap/<name> --yes` |
| **Build from API catalog** | Search 872 built-in API specs, create adapters. **See -0.6.** | `rote adapter catalog search …` → `rote adapter new <id> --yes` |
| **Set up credentials** | Token wizard for installed adapters. **Secrets — see 3c.** | `rote powerpack credentials` (run in user's own terminal) |
| **Connect Google (OAuth)** | Browser OAuth for Google adapters (gmail/calendar). **Non-interactive — see 3d.** | `rote oauth setup google --scopes <list>` |
| **Install the agent skill** | Lets your coding agent (Claude/Cursor/Codex) use rote. | `rote install skill` |
| **Explore / learn** | Onboarding tree + human-friendly help. | `rote how`, then `rote human` |

Recommend the order **adapters → credentials → OAuth → install skill → explore**, but run
only what they pick, one at a time, confirming after each. Once their picks are done, offer
the **value-proof closer (Step 5)** — run one live flow so setup ends with real output.

### Step 3a — Install adapters (à la carte, from the live registry)

**Do not use `rote pull powerpack` presets.** The presets list adapters that aren't
published to the reachable registry (`notion`, `gemini-api`, `googledrive`, `googledocs`),
so they partially or fully fail (`ai` fails entirely; `default`/`dev` choke on notion;
`google` loses drive+docs). Install à la carte instead — every pick either installs or
gives its own clear error, with no phantom adapters.

**Query the registry at runtime** — never hardcode the list, it grows:

```bash
rote registry adapter list --community bootstrap
```

Build the AskUserQuestion checklist (`multiSelect: true`, header `Adapters`) from **only
the adapters that command returns**. As of writing the bootstrap community publishes:
`calendar, cloudflare-api, elevenlabs, exasearch, github, gmail, linear, parallelweb,
polymarket-data, polymarket-gamma, stripe` — but trust the live output, not this list.

**Cap the selection at 2 adapters per run** (fewer than 3). If the user wants more, they
re-run this step. Keep the first pass small so credential setup stays manageable.

Install each picked adapter as its own Bash call:

```bash
rote registry adapter pull bootstrap/github --yes
```

`--yes` keeps it non-interactive. After each, note whether it needs a credential and carry
that into Step 3c. Then reindex once at the end if needed (`rote adapter list` to confirm
ready state). Adapters that need an API token will surface their env var name; Google
adapters (gmail/calendar) use OAuth via Step 3d, not a static token.

**Pull each adapter's flows right after installing it.** The adapter pull installs only the
adapter — its curated flows are separate, and the value-proof closer (Step 5) needs at
least one flow present. For each adapter just installed, find and pull its flows:

```bash
rote registry flow find-by-adapter github
```

That lists `<org>/<name>` flows associated with the adapter (e.g.
`modiqo/list-top-committers`, `modiqo/retrieve-recent-emails`,
`modiqo/check-calendar-meetings`). Pull them non-interactively — `--yes` is **required**,
the pull confirm-prompt has no TTY in an agent shell:

```bash
rote registry flow pull modiqo/list-top-committers --yes
```

Pull a couple of the most useful per adapter (don't bulk-pull all of them). These land in
`~/.rote/flows/<org>/<name>/` and become the menu for Step 5.

### Step 3b — Install skill provider

If they pick "Install the agent skill", ask which agent (AskUserQuestion, header
`Agent`): **Claude** (default), **Cursor**, **Codex**, or **All**. Then:

```bash
rote install skill --provider claude
```

(Use `--provider all` for "All".) `--force` overwrites an existing SKILL.md without
prompting — only add it if the user confirms an overwrite.

### Step 3c — Credentials (static tokens — hand off, never capture in chat)

**State this constraint up front, before offering any option.** A Claude skill cannot
render a masked input field — the only interactive surface is AskUserQuestion (fixed
buttons, no text/password field), and anything the user types into the conversation is
recorded in the transcript in clear text. So **there is no way to enter a static token
through the agent without exposing it.** The terminal wizard is the only unexposed path.
Say this plainly so the user understands the handoff isn't friction, it's the secure choice.

Two kinds of credential, handled very differently — **classify first**:

- **OAuth adapters (Google: gmail, calendar)** → no static token at all. Skip to **Step 3d**
  — the browser redirect keeps the secret off the terminal *and* the transcript. Always
  prefer this when the adapter supports it.
- **Static-token adapters (github `GITHUB_TOKEN`, linear `LINEAR_API_TOKEN`, stripe, etc.)**
  → the user holds a bearer string that has to be conveyed. Hand off to the masked wizard:

  **Default — hand off to the terminal wizard.** Tell the user to run, **in their own
  terminal** (not via the agent):
  ```
  rote powerpack credentials
  ```
  It prompts per-adapter and **masks** input (rote uses `rpassword`), storing to
  `~/.rote/secrets/` (perms 600). The secret never passes through the agent or the
  transcript. This is the recommended path — frame the handoff as the secure one. The user
  runs it, returns, and you verify (below). Note: the wizard may also list `GSUITE_TOKEN` —
  tell the user to **skip it**, since Google is wired via OAuth in Step 3d, not a token.

  **`rote token set` is a last-resort opt-in only.** Offer it via AskUserQuestion *only* if
  the user explicitly refuses the terminal handoff and insists on setting a token through
  the agent. State plainly: "this puts the token in the transcript and shell history —
  rotate it afterward if it's long-lived." Only on explicit go-ahead:
  ```bash
  rote token set GITHUB_TOKEN "<value>"
  ```
  (This is the exposure tracked for a future `--stdin` fix — see the secrets behavior note.)

- **Never print, echo, or re-quote a token back** to the user once set. Don't `cat` the
  secrets dir.

**Verify with cwd-independent checks only.** Do NOT run a flow or `rote ready` to test a
token — those require being *inside a rote workspace* and otherwise fail with
`not in a workspace directory` / `Permission denied (os error 13)` (that error from a
scratch dir like `/tmp` is the workspace requirement, not a bad token). Verify with:

```bash
rote powerpack tokens
```

(or `rote token get <name>` for one) — both work from any directory and show
configured/not-configured without exercising the API.

### Step 3d — Google OAuth (non-interactive)

The bare `rote oauth setup google` opens an interactive multi-select ("Which Google APIs
do you need?") that has no TTY in an agent shell. Drive it non-interactively by
**pre-selecting scopes with `--scopes`** (this is exactly what the rote-vscode extension
does):

```bash
rote oauth setup google --scopes gmail.readonly,calendar.calendarlist.readonly,calendar.calendars.readonly,calendar.events.readonly,calendar.events.freebusy,drive.file
```

That's the standard discovery scope set (Gmail read, Calendar read + free/busy, Drive
app-files). If the user only wants a subset, offer a scope choice via AskUserQuestion
(Gmail-only `gmail.readonly`; Calendar-only the four `calendar.*`; add `drive.file` for
Drive) and pass just those — comma-separated, no spaces. This still opens a **browser** for
the Google consent screen; tell the user to complete it there. The token is stored as
`GSUITE_TOKEN`; confirm with `rote powerpack tokens` afterward.

### Step 3e — OAuth DCR adapters (reference)

Some MCP adapters authenticate via **OAuth Dynamic Client Registration (DCR)** rather than
a static token — for those, after `rote registry adapter pull bootstrap/<name> --yes`, run:

```bash
rote adapter reauth <name>
```

The first run registers a DCR client (RFC 7591) and opens a PKCE browser flow; tell the
user to complete it in the browser. If a later reauth fails because the provider pruned the
client (rare), add `--force-reregister`. Note: **Notion is not currently in the reachable
registry** (`bootstrap` / `modiqo`), so don't offer a Notion install — the bootstrap
community list in Step 3a is the source of truth for what's actually installable.

---

## Step 4 — Wrap up

After the menu items they chose are done, confirm and orient:

```bash
rote how
```

Summarize what's now set up (signed-in email, adapters pulled, credentials wired, skill
installed) and point at `rote human` for friendly help and `rote how` for the agent
onboarding tree. Then offer the value-proof closer (Step 5).

---

## Step 5 — Prove value: run one live flow (optional, recommended)

End on a win — run a real flow against the user's own data so setup ends with *output*,
not just "complete." Only offer this if at least one credential is wired (a flow with no
working credential will just error). Ask first (AskUserQuestion, header `Try it`): "Want me
to run a quick flow to see it work?" — yes / skip.

**1. Find adapter-matched flows.** Use the adapters they installed (and credentialed) so the
flow can actually run. For each, list its flows:

```bash
rote registry flow find-by-adapter github
```

Prefer flows for an adapter whose credential is configured (check `rote powerpack tokens` /
the Google OAuth result). A read-only flow is the safest first run (e.g.
`list-top-committers`, `retrieve-recent-emails`, `check-calendar-meetings`) — avoid
write/create flows (`create-github-issue`) for the value-proof.

**2. Let the user pick one** (AskUserQuestion, header `Flow`), built from the matched list.

**3. Ensure it's pulled.** If not already local from Step 3a, pull it (`--yes` required):

```bash
rote registry flow pull modiqo/list-top-committers --yes
```

**4. Read its parameters from frontmatter — don't guess.** Each flow declares params in a
YAML `parameters:` block in its `main.ts` (name + description + required). Read them:

```bash
rote flow run list-top-committers --help
```

(or read the `parameters:` block in `~/.rote/flows/<org>/<name>/main.ts`). For **each**
declared param, ask the user for a value — use the param's `description` as the hint. Ask in
prose (AskUserQuestion is for fixed choices; param values are free text), e.g. "This flow
needs `repository` (Repository in format 'owner/repo') — what should I use?" Offer a sensible
default where obvious (e.g. `modiqo/rote`).

**5. Preview, then run — inside a workspace.** `flow run` needs a workspace cwd. Create one
and run **in the same Bash call** (see "Running a workspace-scoped command" in Behavior
notes — `eval $(rote cd …)` / `--enter` won't carry across the agent's separate Bash calls).
Params are `key=value` pairs; do a `--dry-run` first to preview without making calls:

```bash
rote init proof --seq --force
```
```bash
cd ~/.rote/rote/workspaces/proof && rote flow run list-top-committers repository=modiqo/rote --dry-run
```

Then run it for real:

```bash
cd ~/.rote/rote/workspaces/proof && rote flow run list-top-committers repository=modiqo/rote
```

If it still fails with `not in a workspace directory` / `Permission denied (os error 13)`,
the `cd` didn't take — make sure the workspace path is right (`~/.rote/rote/workspaces/<name>`)
and that the `cd` and `rote` share one Bash call. That error is the cwd requirement, **not** a
bad credential or flow.

Show the flow's output to the user — that's the payoff. Then suggest the main **rote** skill
for day-to-day use (`rote flow search "<intent>"` before any direct adapter call).

---

## Behavior notes

- **One command per Bash call.** No `&&`, `|`, or `;` chains. The user wants each
  success/failure visible on its own. Two allowed compounds, both single logical steps: the
  vendor installer `ROTE_YES=1 bash -c "$(curl -fsSL https://getrote.dev/install)"` (run only
  after explicit confirmation — it executes remote code), and `cd <workspace> && rote …` for
  workspace-scoped commands (the cwd must hold for the command; see below).
- **Everything must run non-interactively** — the agent shell has no TTY, so any command
  that prompts dies on `/dev/tty: Device not configured`. The non-interactive switch differs
  per command: installer → `ROTE_YES=1`; powerpack picker → `--yes`; `rote adapter new` →
  `--yes`; `rote adapter new-from-mcp` → **`--headless`** (it rejects `--yes`); Google OAuth →
  `--scopes …`. Don't guess the flag — if unsure, check `<command> --help` first. The one
  prompt you must NOT automate away is credential entry — see secrets below.
- **Never handle secrets in the agent — you cannot mask them.** A skill has no masked
  input field; anything typed in chat is in the transcript. So static tokens always hand
  off to `rote powerpack credentials` in the *user's own* terminal (masked via rpassword) —
  frame the handoff as the secure path, not friction. Prefer OAuth (browser redirect) over
  any static token where the adapter supports it. `rote token set <name> <value>` is a
  last-resort opt-in that puts the token in the transcript — only on explicit insistence,
  with a rotate-afterward warning. Never echo a token back or read the secrets dir.
- **Verify without a workspace where possible.** To check a credential, use
  `rote powerpack tokens` / `rote token get` — these work anywhere. But **anything that runs
  a flow, probes an adapter, or calls an API needs a workspace cwd** (`rote flow run`,
  `rote <adapter>_probe`, `rote ready`, `rote POST`) — outside one they fail with
  `not in a workspace directory` / `Permission denied (os error 13)`.
- **Running a workspace-scoped command (the correct pattern).** The agent shell does not
  persist cwd between Bash calls, and `rote cd` / `--enter` only affect an interactive shell —
  so `eval $(rote cd …)` won't carry over. Instead, create the workspace, then `cd` into its
  real directory **in the same Bash call** as the command:
  ```bash
  rote init proof --seq --force
  cd ~/.rote/rote/workspaces/proof && rote flow run <name> key=value
  ```
  The workspace lives at `~/.rote/rote/workspaces/<name>`. This `cd && rote …` compound is a
  necessary exception to the one-command-per-call rule (the cwd must hold for the command).
  `--force` skips the "search for existing flows first" gate. Clean up later with
  `rote workspace clean` if desired.
- **Detect before offering.** Confirm the rote binary (`command -v rote`) before any rote
  command, and detect installed editors (`command -v code|cursor|antigravity`) before
  offering the extension path. Never present an install target that isn't there.
- **Detect, don't assume — and never from memory.** Every fact (installed? where? logged
  in?) comes from a live probe in this run, not from recalled notes, CLAUDE.md, or a "rote is
  active here" assumption. A stale memory must never short-circuit a probe; if memory and a
  live probe disagree, the probe wins. Once rote is present, start from `rote whoami` and
  branch on its real output.
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
