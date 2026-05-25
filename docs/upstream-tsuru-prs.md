# Upstream tsuru-client PRs

Living catalog of contributions Coffeece wants to land in
[`tsuru/tsuru-client`](https://github.com/tsuru/tsuru-client) (and a few in
`tsuru/tsuru` server). Each is something we prototype here first (per the
"Coffeece-first, propose upstream second" policy in
[`~/.claude/plans/redhat-mentioned-the-openshell-iridescent-pretzel.md`](../../../../home/guilhermebr/.claude/plans/redhat-mentioned-the-openshell-iridescent-pretzel.md)),
then push as an independent PR from `~/code/guilhermebr/tsuru-client`.

PRs are opened one at a time. After merge, the corresponding wrapper code in
this repo simplifies or deletes; net wrapper code decreases over time.

Already shipped upstream (no PR needed — discovered 2026-05-24):
- ✅ Subcommand-style routing (`tsuru app create` works as alias of `app-create`) — `ManagerV2` already does this for any hyphenated name registered with a topic.
- ✅ `tsuru completion bash|zsh|fish` — Cobra default, ManagerV2 inherits it.

---

## P1 — directly unblocks Coffeece deploy composite

### 1. `tsuru app deploy --create`

Auto-create the app if it doesn't exist before deploying. Today the wrapper
does a `GET /apps/{name}` check and a separate `app create` call before
`app deploy`. Upstream would make this a single command.

**Flag surface (proposal):**
```
--create                    Create the app if it doesn't exist
--platform string           Platform (required with --create if no tsuru.yaml)
--plan string               Plan (optional)
--pool string               Pool (optional)
--team string               Team owner (optional, defaults to user's first team)
```

**Behavior:**
- Without `--create`: current behavior. 404 → error.
- With `--create` + missing app: call create, then deploy.
- With `--create` + existing app: no-op on create, proceed to deploy.

**Touch:** `tsuru/client/deploy.go` (`AppDeploy.Flags()` + `Run()` pre-flight),
`tsuru/client/deploy_test.go` (new test cases).

**Design notes:**
- Keep the flags additive — don't change `AppCreate` itself.
- `--create` is the gate; the other flags are inert without it (or error if
  `--platform=foo` is passed without `--create`).
- Look at how `tsuru.yaml` is currently parsed server-side — if `platform:` is
  there, `--platform` could be optional. Worth checking before designing the
  flag.

---

### 2. `tsuru app deploy --env-file` and `--env KEY=VAL`

Apply env vars as part of the deploy, without a separate `env-set` step. Today
the wrapper calls `EnvSet` with `--no-restart`, then runs `AppDeploy` (which
restarts). Upstream should collapse that.

**Flag surface (proposal):**
```
--env-file string           Read KEY=VALUE pairs from a dotenv file (set --private by default)
--env stringArray           Set additional KEY=VALUE (repeatable; not private by default)
--env-private               Treat values from --env as private (no effect if only --env-file used)
```

**Behavior:**
- Variables applied with `NoRestart: true` (the deploy itself restarts).
- `--env-file` defaults to private (env files usually hold secrets).
- On key conflict, `--env` (CLI) wins over `--env-file`.
- Format errors in the env file → fail before any server call.

**Touch:** `tsuru/client/deploy.go` (parser + EnvSet call before deploy
proper), `tsuru/client/deploy_test.go`.

**Design notes:**
- The env-file parser in [`cmd/coffeece/project.go`](../cmd/coffeece/project.go)
  (`parseEnvFile`) is small (~30 LoC) and contribution-ready — lift it directly
  into the upstream PR.
- Pair this PR with #1 so the wrapper's whole `coffeece deploy` composite
  reduces to: `tsuru app deploy --create --env-file …` + the headline name.

---

## P2 — significant UX gains, larger touch surface

### 3. `-o json|yaml|wide` on list commands

Output formats on `app list`, `service list`, `pool list`, `plan list`, and
similar. Today these print TabWriter-formatted text only.

**Flag surface:**
```
-o string    Output format: text|json|yaml|wide  (default "text")
```

**Behavior:**
- `text` — current behavior.
- `json` — array of structs matching the existing API response shape.
- `yaml` — same as JSON but YAML.
- `wide` — text with extra columns (e.g., for `app list`: include router,
  description, tags).

**Touch:** each command's `Run()` (substantial: ~5–10 commands), shared
formatter in `tsuru/client/formatter/` (likely needs a new package),
`tsuru/client/*_test.go` (one new test case per command).

**Design notes:**
- Big PR. Worth splitting per command if maintainers prefer smaller chunks.
- Start with `app list` as the proof-of-concept; bring others on once the
  shared formatter is reviewed.
- Mirror `kubectl`'s `-o` flag semantics; treat that as the design reference.

---

### 4. `tsuru login --browser` (browser OAuth flow)

OPEN URL → loopback callback → token written to `~/.tsuru/`. Today `tsuru
login` is interactive username/password. Browser flow is more modern and works
with OIDC providers (which Coffeece already uses).

**Flag surface:**
```
tsuru login --browser    Open the configured auth URL in the user's browser
```

**Touch:** `tsuru/auth/login.go` (new flow branching off existing command),
new file `tsuru/auth/browser.go` for the localhost listener.

**Design notes:**
- Coffeece needs a `coffeece login` that targets `https://api.coffeece.com`
  specifically; the underlying mechanism (browser flow) goes upstream as a
  generic `tsuru login --browser`. Coffeece's `coffeece login` becomes a thin
  preset.
- Requires the Tsuru server to be configured with an OAuth/OIDC endpoint
  reachable from the user's browser. Coffeece's setup already meets this.
- Reference: GitHub CLI `gh auth login --web`, Fly.io `flyctl auth login`.

---

## P3 — quality of life, no urgency

### 5. Library functions for `Deploy`, `CreateApp`, `SetEnvs`

Expose the core operations as importable functions in `tsuru-client/tsuru/client`
so wrappers (like this CLI) don't have to parse synthetic flags into command
structs.

**Today's pattern (in [`cmd/coffeece/deploy.go`](../cmd/coffeece/deploy.go)):**
```go
c := &client.AppCreate{}
c.Flags().Parse([]string{"--plan", plan, ...})  // synthetic flag parsing
c.Run(childContext(parent, []string{app, platform}))
```

**Proposed (after PR):**
```go
client.CreateApp(ctx, client.CreateAppOpts{Name: app, Platform: platform, Plan: plan, ...})
```

**Touch:** new file `tsuru/client/api.go` exposing functions; existing
command structs become thin wrappers over the functions.

**Design notes:**
- Refactor — not a feature. Maintainers may prefer to keep this internal.
- If PRs #1 and #2 are accepted, the wrapper's need for synthetic-flag parsing
  drops significantly; this PR matters less. Reassess after #1+#2 ship.

---

### 6. Tag releases as `vX.Y.Z` (semver) instead of `X.Y.Z`

Current upstream tags are like `1.34.0`. Go modules treat untagged-with-`v`
releases as pseudo-versions only. Adding the `v` prefix would let consumers
pin to clean semantic versions (`@v1.34.0` instead of
`@v0.0.0-20260509012344-7a35a1d378f8`).

**Touch:** `Makefile` / release script; existing tags retagged (or just new
ones going forward).

**Design notes:**
- Pure repo maintenance — issue, not PR. Open with the release script change
  as the proposed fix.
- Coffeece currently consumes via pseudo-version (`@main`); this would let us
  pin to a clean tagged release for v0.1.0 release notes.

---

## P4 — server-side (`tsuru/tsuru`, not `tsuru-client`)

### 7. Native `env:` in `tsuru.yaml` honored server-side

Move env-on-deploy from a client flag (PR #2) to a manifest field. `tsuru.yaml`
already lives at the project root and is parsed server-side; adding an `env:`
map there is the cleanest long-term home.

**Touch:** `tsuru/app/yaml.go` (or wherever `tsuru.yaml` parsing lives),
server-side deploy handler.

**Design notes:**
- Bigger change. Requires server-side coordination, migration plan, docs.
- Don't pursue until PR #2 has shipped and there's real user demand for the
  manifest form.

---

## Conventions

- Each PR is opened from `~/code/guilhermebr/tsuru-client` against
  `tsuru/tsuru-client` main.
- PR titles framed as generic Tsuru improvements ("Add `--create` flag to
  `app deploy`"), not "for Coffeece."
- PR description references the Coffeece release where the change shipped —
  proof it was used, not theoretical.
- If a PR stalls >60 days with no maintainer signal, Coffeece keeps shipping
  in the wrapper and revisits.
- After merge, the corresponding wrapper code is simplified or deleted in the
  next Coffeece release; this catalog updated to mark the PR ✅.
