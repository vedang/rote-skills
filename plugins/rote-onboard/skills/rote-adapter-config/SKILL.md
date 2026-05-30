---
name: rote-adapter-config
description: >
  Tune an existing rote adapter — auth, base URL, write guard, sensitivity, capability index,
  OAuth re-auth, GraphQL field filter, policies, subagent skill, grouping, versioning. A menu
  of small show → confirm → apply → re-show operations on an adapter you already created. Use
  when the user says "configure <adapter>", "change the base url", "re-auth notion", "rebuild
  the capability index", "enable an auth scheme", "set up sensitivity / write guard",
  "/rote-configure-adapter". For creating a NEW adapter, use rote-adapter-create instead.
---

# rote-adapter-config — tune an existing adapter

Iterative configuration of an adapter that already exists. Unlike creation (a linear
dry-run-first pipeline), this is a **menu of discrete operations**, each a small loop:
**show current state → confirm the change → apply → re-show**. Return to the menu after each.

Rules: determine facts from live commands (never memory); one command per Bash call; secrets
never captured in chat (masked handoff). If rote isn't on PATH, resolve its absolute path.

---

## Open — show current state

Identify the adapter and show what's there now:

```bash
rote adapter list
```
```bash
rote adapter info <id>
```
```bash
rote adapter keys <id> --json
```

Then present the operation menu (AskUserQuestion). Run only what the user picks; re-show the
relevant state after each change.

---

## Operations (all verified against the live CLI)

### Settable keys — base URL, name, description, tags, sensitivity tier

```bash
rote adapter set <id> base_url <url>
```
Other settable keys (same command): `name`, `description`, `tags` (csv),
`sensitivity_tier` (`low|medium|high`), `additional_headers.<name>` (`""` deletes),
`enable_parameter_cleaning` (bool). Check current values first with `rote adapter keys <id>`.

### Auth — single scheme

```bash
rote adapter auth update <id> --bearer-token <ENV_VAR>
```
Variants: `--api-key-header <name=ENV>`, `--api-key-query <name=ENV>`, `--none` (remove auth).
The token **value** is a secret — set it via the masked `rote powerpack credentials` (user's
own terminal) or `rote token set <ENV> "<value>"` as an explicit opt-in. Never echo a token.

### Auth — multi-scheme (per-operation adapters)

```bash
rote adapter auth scheme list <id>
```
```bash
rote adapter auth scheme enable <id> <scheme>      # idempotent; descriptor optional
rote adapter auth scheme add <id> <scheme> [descriptor]   # (re-)binds + runs credential flow
rote adapter auth scheme disable <id> <scheme> [--force]  # --force to disable the default
```

### Re-authorize OAuth

```bash
rote adapter reauth <id> [--scheme <name>] [--force-reregister]
```
Opens a browser. `--force-reregister` re-runs DCR when the provider pruned the client (rare).

### Write guard

```bash
rote adapter guard show <id>
rote adapter guard init <id>
```
`init` generates `write_guard.json` from the adapter's tools (allow/confirm/deny gradient).
**There is no first-class "disable" command** — say so; disabling means removing the guard
files manually.

### Sensitivity classification

```bash
rote sensitivity upgrade            # download/update the index (first time)
rote sensitivity apply <id> --json  # classify this adapter's fields
rote sensitivity lookup <id> --json # show current classification
```

### Capability index

```bash
rote adapter capability status
rote adapter capability rebuild
```

### GraphQL field filter (GraphQL adapters only)

```bash
rote adapter tool-spec <id> --show
```
Then `--add` / `--rm <INDEX>` / `--action <skip|keep>` with `--tool`/`--type`/`--field`/
`--desc-prefix`/`--desc-contains` to scope which fields the selection set requests. Skip
entirely for non-GraphQL adapters.

### Policies (rate limit, cost, retry, cache, circuit breaker)

```bash
rote adapter policies <id>
```
Generate from a preset: `rote adapter policies <id> --generate --preset <github|openai|conservative>`.

### Adapter subagent skill

```bash
rote adapter agent list
rote adapter agent generate <id> [--force]
```
Generates/refreshes the per-adapter subagent template. (To wire the rote skill itself into an
agent, that's `rote install skill --provider <name> --agents` — a different, broader command.)

### Group / version

```bash
rote adapter group set <id> <group>     # also: group unset <id>, group list
rote adapter bump <id> [--minor|--major]
```

---

## Honest limits — state these when relevant

- **No post-creation toolset toggle.** Toolset selection is create-time only (via the
  dry-run + `toolset_filters` in rote-adapter-create). To change which toolsets are active,
  recreate the adapter, or — for GraphQL — use `tool-spec` for field-level filtering. Don't
  offer a toolset-toggle command; it doesn't exist.
- **No write-guard disable command** — only `init` / `show`.

---

## Pattern (per operation)

1. **Show** current value (`info` / `keys` / `guard show` / `auth scheme list` / `tool-spec --show`).
2. **Confirm** the change with the user.
3. **Apply** the verified command.
4. **Re-show** the affected state so the user sees the effect.
5. Return to the menu, or exit.
