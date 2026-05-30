---
name: rote-adapter-create
description: >
  Create a rote adapter for any API — dry-run-first. Discovers the spec (built-in catalog →
  web search → local file), analyzes it without committing (`rote adapter new <id> <spec>
  --dry-run`), then pre-fills auth, base-url, and toolset choices from the analysis — asking
  you only where it's genuinely ambiguous. When auth is ambiguous or low-confidence, researches
  the provider's docs to recommend a scheme and produce setup steps (API key / OAuth2). Use
  when the user says "add an adapter", "connect
  to <API>", "create a stripe/notion/datadog adapter", "build an adapter from this OpenAPI
  spec", "/rote-add-adapter". Never creates from a discovered spec without a clean dry-run
  first. Hands off to rote-adapter-config for tuning afterward.
---

# rote-adapter-create — dry-run-first adapter creation

Build an adapter for any API the way the rote-vscode wizard does: **analyze the spec with no
side effects, then drive the choices from what the analysis found.** You hold the dry-run JSON
in context and ask the user only the questions the analysis can't answer.

Core rules:
- **Never `--yes` create from a discovered spec without a successful `--dry-run` first.** The
  dry-run is the validation gate at every discovery branch.
- **Determine facts from live commands, never from memory.** (If the rote binary isn't on
  PATH, resolve its absolute path — same as the setup skill.)
- One command per Bash call. Secrets (API tokens) are never captured in chat — hand off to the
  masked wizard (see Stage 5).

---

## Stage 0 — Discover the spec (catalog → web → local file)

Ask what API the user wants (prose): "Which API? (e.g. notion, stripe, datadog — or paste a
spec URL / file path)". Then resolve a **spec source** through this chain. The spec source is
either a catalog id, a URL, or a local path.

### 0a. Catalog (built-in, ~872 specs)

```bash
rote adapter catalog search "stripe"
```

If there's a match, show details — the user sees auth type, spec URL, and notes:

```bash
rote adapter catalog info stripe
```

**Important resolution detail (verified):** `rote adapter new <catalog-id>` (no `--dry-run`)
auto-resolves the spec from the catalog, but **`--dry-run` does NOT resolve a catalog id** —
it treats the argument as a file path. So for the dry-run, use the **Spec URL** from
`catalog info`, not the catalog id:

```bash
rote adapter new <id> <SPEC_URL_FROM_CATALOG_INFO> --dry-run
```

For MCP-type entries (catalog `info` Spec Type = "MCP", e.g. Notion), don't dry-run — go to
the MCP path (Stage 5, MCP note).

### 0b. Web search (not in catalog)

If `catalog search` is empty, web-search for the API's **machine-readable spec** — an OpenAPI
JSON/YAML URL or a GraphQL endpoint, **not** a human docs page. Propose the candidate URL to
the user, then **validate it by dry-run** before trusting it (Stage 1). If the dry-run fails
(404, HTML-not-a-spec, 403/401), say so plainly and try the next candidate or ask the user for
the right URL. Wrong-spec is the main failure mode here — never create from an unvalidated
web-found URL.

### 0c. Local file

If the user has a spec file, use the path directly: `rote adapter new <id> <path> --dry-run`.

Pick an adapter **id** with the user (lowercase, hyphens ok, e.g. `stripe`, `acme-api`).

---

## Stage 1 — Analyze (dry-run)

```bash
rote adapter new <id> <spec-source> --dry-run
```

This emits JSON and **creates nothing**. Keep the JSON in context. Its shape (verified):

- `spec`: `{ title, version, openapi_version, base_url, operation_count, spec_size_bytes }`
- `toolsets[]`: `{ name, tool_count, confidence, methods{GET,POST,…}, read_operations, write_operations }`
- `auth`: `{ type, ... }` — `type` is `none` | `bearer` | `api_key_header` | `api_key_query` |
  `basic` | `per_operation`. For `per_operation`: `schemes{}`, `default_scheme`.
- `auth_scoring`: `{ recommended{auth_type,confidence,source,detail}, all_schemes[], has_ambiguity }`
- `summary`: `{ total_toolsets, total_tools, get/post/put/delete_operations, read/write_operations }`

If the dry-run errors, report the error verbatim and return to discovery (the URL/spec is the
problem, not the adapter).

---

## Stage 2 — Review

Show the user the headline facts from `spec` + `summary`: title, version, base_url,
`total_tools` and read/write split. **Only ask for a base-url override if `spec.base_url` is
empty** (some specs omit `servers`). Offer to set a custom display name/description only if the
user wants to.

---

## Stage 3 — Auth (resolve, don't punt)

Drive from `auth` + `auth_scoring`. Pick a branch from two fields — `has_ambiguity` and
`recommended.confidence` — then **resolve ambiguity by reading the provider's own docs**, not
by handing the user a blind multiple-choice:

```
auth.type == "none"                                          → skip, no auth
has_ambiguity == false  AND  recommended.confidence >= 0.8   → CONFIRM (no research)
has_ambiguity == false  AND  recommended.confidence  < 0.8   → RESEARCH-LITE (verify the lone default)
has_ambiguity == true                                        → RESEARCH-FULL (disambiguate + setup)
auth.type == "per_operation"                                 → RESEARCH-FULL scoped to default_scheme
```

Low confidence is itself a trigger — a single-scheme guess at 0.5 (common for GraphQL specs:
`detail` says "no directives found") is rote *guessing*, so verify it before the user pastes a
token, not after the probe 401s.

- **CONFIRM** → state the scheme and proceed, pre-filling the token env var from
  `auth.token_env` / `key_env`. e.g. "Detected bearer auth (0.95 confidence, from the spec).
  I'll use `STRIPE_TOKEN` — sound right?"
- **RESEARCH-LITE / RESEARCH-FULL** → run the **Auth research sub-routine** below, then present
  a *researched recommendation with setup steps inline* (not a blind menu).
- **`per_operation`** → research the `default_scheme`; you can create with it now and enable
  other `schemes` later via **rote-adapter-config** (`auth scheme`).

### Auth research sub-routine (read-only, pre-create)

Runs between Stage 1 (dry-run) and Stage 5 (create). Creates nothing, touches no secrets — it
only sharpens *which* `config-json` you assemble and *what instructions* the user gets.

1. **Derive search terms from context (never memory):** provider = `spec.title` / adapter id;
   candidate schemes = `auth_scoring.all_schemes[].auth_type.type`. Query templates:
   - `"<provider> API authentication" (api key OR oauth OR bearer)`
   - `"<provider> create API key"` / `"<provider> oauth2 client credentials"`
   - Prefer the provider's own docs domain (developer.x.com, docs.x.com) over blogs.

2. **Fetch the top authoritative hit** and extract a structured profile **per candidate scheme**:
   ```
   { scheme: bearer | api_key_header | api_key_query | oauth2_authorization_code |
             oauth2_client_credentials | basic,
     header_format: "Authorization: Bearer {token}" | "Authorization: {token}" |
                    "X-Api-Key: {token}" | ...,
     token_env:    <suggested env var>,
     how_to_obtain: [ ordered steps ],
     doc_url:      <source>,
     caveats:      [ "personal keys omit Bearer prefix", "key scoped per-workspace", ... ] }
   ```

3. **Reconcile research against the dry-run** — provider docs win over the dry-run guess, but
   cite `doc_url` for every claim:
   - **Confirms** the recommended scheme → proceed with it, now carrying the correct
     `header_format` + setup steps.
   - **Contradicts** it (dry-run guessed bearer, docs say `X-Api-Key`) → present both via
     `AskUserQuestion`, lead with the doc-backed one, cite the source.
   - **Inconclusive** → fall back to asking the user (today's behavior), but with whatever
     partial setup info was found.

### Present a researched recommendation, not a blind menu

Use `AskUserQuestion`. Lead with the doc-backed scheme; put the setup steps in the option
description so the user can act without leaving the flow.

**API-key shape:**
```
Recommended: API key in header  (developer.linear.app)
  Header:  Authorization: <key>     ← raw, no "Bearer " prefix
  Get one: linear.app/settings/api → "Create key" → Personal API key
  Env var: LINEAR_API_TOKEN
[ Use this ]  [ Use OAuth2 instead ]  [ Paste a different scheme ]
```

**OAuth2 shape** (the case people get stuck on — surface what they must register up front):
```
Recommended: OAuth2 authorization_code  (provider docs URL)
  Need: client_id, client_secret, redirect_uri, scopes [...]
  Register the app: <console URL from docs>
  rote runs: new-from-mcp / OAuth-DCR opens a browser
[ Walk me through OAuth setup ]  [ Use a static token if supported ]
```

### Guardrails

- **Read-only and pre-create.** Research never creates an adapter, never asks for or echoes a
  token value. It only changes the config-json and the instructions.
- **Docs are authoritative over the dry-run guess**, but every claim cites `doc_url`. The
  dry-run's `detail` ("no directives found") is the signal that rote is guessing — trust docs
  more there.
- **The token value is a secret** — see Stage 5 for the masked handoff. Research produces
  *how to obtain* it and the *header format*, never the value.
- **The probe is still the final gate.** Research raises confidence and yields correct setup,
  but Stage 6's `<id>_probe` is what actually proves auth works.

---

## Stage 4 — Toolsets (the "choosing tools" step)

The dry-run `toolsets[]` is the menu. **This is the only place toolsets are chosen** — there
is no post-creation toolset toggle.

- **Small APIs** (a handful of toolsets) → present them all (name · tool_count · methods) and
  let the user include/exclude.
- **Large APIs** (Stripe has 128 toolsets / 587 tools) → do NOT dump all of them. Summarize:
  "This API has {total_toolsets} toolsets / {total_tools} tools. Want all of them, or shall I
  narrow to specific areas?" Then let the user name areas (search `toolsets[].name`) or take
  all. Default to **all** if they don't care.

Build `toolset_filters` from the selection: `{"<toolset-name>": "enabled", ...}` (omit to
include everything).

---

## Stage 5 — Create + post-ops

Assemble `--config-json` from the auth + toolset decisions and create:

```bash
rote adapter new <id> <spec-source> --yes --config-json '{"auth":{...},"toolset_filters":{...}}'
```

(Add `--base-url <url>` if Stage 2 required it, `--group <name>` if grouping.) For the
config-json shape, mirror the dry-run `auth` block for the chosen scheme.

**MCP servers** (Stage 0 flagged Spec Type "MCP") use a different create command — no spec
dry-run, OAuth/DCR runs during creation:
```bash
rote adapter new-from-mcp <id> <mcp-url> --headless
```
For OAuth-DCR servers (Notion) this opens a browser; tell the user to complete it.

### Post-create (offer each; run what the user wants)

- **Credentials** (static-token auth): hand off to the masked wizard — tell the user to run
  `rote powerpack credentials` in their own terminal (masked), or set one with
  `rote token set <ENV> "<value>"` only as an explicit, transcript-visible opt-in. Never echo
  a token. (Same secrets discipline as the setup skill.)
- **Write guard**: `rote adapter guard init <id>`
- **Sensitivity**: `rote sensitivity upgrade` (if needed) then `rote sensitivity apply <id> --json`
- **Adapter subagent**: `rote adapter agent generate <id>`
- **Capability index**: `rote adapter capability rebuild`

---

## Stage 6 — Confirm + hand off

```bash
rote adapter info <id>
```

Show it's ready (tools/toolsets/auth). Offer to:
- **Prove it works** — probe inside a workspace (creation alone doesn't run it):
  ```bash
  rote init proof --seq --force
  ```
  ```bash
  cd ~/.rote/rote/workspaces/proof && rote <id>_probe "<query>"
  ```
- **Tune it** — invoke **rote-adapter-config** for base-url, auth schemes, sensitivity, etc.

**Closing line** (only on a clean create + green probe): land one dry one-liner keyed to this
run's `total_tools` — the shared convention and rules live in [INDEX.md](../../INDEX.md).
e.g. "512 tools talking straight to the provider's API — no metered middleman quietly billing
you per call." Skip it if the run was rocky.

---

## Notes

- `--dry-run` creates nothing — safe to run repeatedly while discovering the right spec.
- A provider-side `HTTP 401/403` on dry-run means the API gates even reading its spec (rare);
  most catalog specs analyze cleanly.
- After a successful create, suggest the main **rote** skill for day-to-day use
  (`rote flow search "<intent>"` before any direct adapter call).

---

## Related onboard skills

Part of the **rote-onboard** sequence ([INDEX.md](../../INDEX.md) is the full map):
- **Next:** `/rote-onboard:rote-adapter-config` — tune the adapter you just made (auth, base
  URL, write guard, sensitivity).
- **First-run / setup:** `/rote-onboard:rote-setup` · **Keep current:**
  `/rote-onboard:rote-update`
