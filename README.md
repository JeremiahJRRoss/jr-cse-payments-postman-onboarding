# jr-cse-payments-postman-onboarding

Postman API onboarding workflow for the **Payment Refund** service. Primary
working implementation for the Postman CSE take-home exercise.

Companion repo (adaptation analysis): **loan-origination-postman-onboarding**.

> **Status:** Workflow verified working end-to-end against the open-alpha
> `postman-cs/postman-api-onboarding-action@v0`. First green run after a clean
> rebuild took 38 seconds (vs. four iterations during initial discovery — see
> §11). Sections marked `TODO` need screenshot evidence captured during your
> validation pass.

---

## 1. Overview

This repo onboards the Payment Refund API into Postman through a single GitHub
Actions workflow (`.github/workflows/onboard.yml`) that calls the open-alpha
orchestrator action. One push triggers the entire chain — workspace creation,
spec upload, collection generation, environment setup, mock and monitor
creation, and a commit-back of the generated artifacts.

Three repo-level secrets and two repo variables drive the workflow:

| Type | Name | Purpose |
|---|---|---|
| Secret | `POSTMAN_API_KEY` | Long-lived PMAK (Postman web UI → Settings) |
| Secret | `POSTMAN_ACCESS_TOKEN` | Session-scoped token extracted from `~/.postman/postmanrc` after `postman login` — required for Bifrost git-sync, governance, and system-env steps |
| Secret | `GH_FALLBACK_TOKEN` | Fine-grained PAT with Contents/Workflows/Actions/Variables write — required because default `GITHUB_TOKEN` can't write to `.github/workflows/` |
| Variable | `REQUESTER_EMAIL` | Workspace audit metadata |
| Variable | `POSTMAN_USER_ID` | Numeric Postman user ID for workspace admin assignment |

## 2. Services Onboarded

- **Payments (Refund API)** — this repo. Lambda + API Gateway compute, OAuth 2.0 + JWT auth declared in spec.
- **Loan Origination** — companion repo [`jr-cse-loan-origination-postman-onboarding`](https://github.com/JeremiahJRRoss/jr-cse-loan-origination-postman-onboarding). Demonstrates pattern transfer to ECS+ALB compute and JWT auth (mTLS handled via the action's `ssl-client-*` inputs as customer-side cert provisioning).

## 3. Repository Structure

```
jr-cse-payments-postman-onboarding/
├── .github/workflows/
│   └── onboard.yml             # the onboarding workflow (this is the implementation)
├── evidence/generated-artifacts/
│   ├── payments-tests.yml.txt  # the action-generated CI workflow, verbatim — quarantined here
│   └── README.md               # why it's quarantined (invalid-YAML upstream quirk; regenerates each run)
├── specs/
│   └── payment-refund-api-openapi.yaml
├── postman/
│   ├── collections/            # canonical git-sync YAML (action-managed; the workflow reads/writes these)
│   │   ├── [Baseline] payment-refund-service/   # collection.yaml + *.request.yaml
│   │   ├── [Contract] payment-refund-service/
│   │   └── [Smoke] payment-refund-service/
│   └── exports/                # single-file v2.1 JSON of the same collections (per the brief)
│       ├── baseline.json
│       ├── contract.json
│       └── smoke.json
├── docs/screenshots/           # validation evidence (see §9)
├── service.config.yml          # per-service inputs documented (no secrets)
├── issues-log.md               # honest build log — iterations, errors, fixes
├── SETUP.md                    # verified ~20-minute setup path
├── CLAUDE.md / BLUEPRINT.md / PROMPTS.md   # build operating docs
└── VERIFIED-NOTES.md           # facts confirmed against the live action.yml
```

## 4. Workflow Architecture

The action chain works in three phases:

1. **`postman-api-onboarding-action@v0`** orchestrates two lower-level actions and
   resolves identity inputs (workspace admin user, requester email, domain code).
2. **Bootstrap phase** — calls Postman APIs to create the workspace, upload the
   spec to Spec Hub, generate baseline/smoke/contract collections from the spec,
   configure environments per the `environments-json` input, create a mock
   server, and create a monitor.
3. **Repo-sync phase** — exports the generated collections to
   `postman/collections/`, commits them back to `main` (using the
   `gh-fallback-token` PAT — not the default `GITHUB_TOKEN`, which can't write
   to `.github/workflows/` by design), and writes the generated CI test workflow
   (`payments-tests.yml`).

## 5. Universal Pattern vs Per-Service Differences

The pattern that's identical across services:

| Aspect | Universal? |
|---|---|
| Action chain (`postman-api-onboarding-action@v0`) | ✓ Same orchestrator |
| Permissions block (`actions: write`, `contents: write`) | ✓ Same |
| Secret references (POSTMAN_API_KEY, POSTMAN_ACCESS_TOKEN, GH_FALLBACK_TOKEN) | ✓ Same |
| GitHub token wiring (`github-token`, `gh-fallback-token`) | ✓ Same |
| `generate-ci-workflow: true` for continuous coverage | ✓ Same |
| Workflow file structure | ✓ Same |
| Validation pattern (six UI checks) | ✓ Same |

What varies per service:

| Aspect | Payments | Loan Origination |
|---|---|---|
| `project-name` | `payment-refund-service` | `loan-origination-service` |
| `domain` / `domain-code` | `payments` / `PMT` | `lending` / `LND` |
| `spec-path` / `spec-url` | Points at payments spec | Points at loan-origination spec |
| `environments-json` | `["prod","uat","qa","dev"]` (4) | `["prod","staging","dev"]` (3) |
| `env-runtime-urls-json` | payments.example.com URLs | lending-api.example.com URLs |
| `ci-workflow-path` | `payments-tests.yml` | `loan-origination-tests.yml` |

Onboarding a new service changes **8 input values** in `onboard.yml`.
Everything else — the action chain, the job structure, the permissions block,
the generated baseline/smoke/contract collections, and the environment
scaffolding — is identical. The 8 are: `project-name`, `domain`, `domain-code`,
`spec-url`, `spec-path`, `environments-json`, `env-runtime-urls-json`, and
`ci-workflow-path`. (The table above groups some of these into a single row —
`domain` / `domain-code` and `spec-path` / `spec-url` each move together.) See
`ADAPTATION.md` in the loan-origination repo for the actual measured diff.

## 6. What Generated Checks Validate vs What Still Needs Service-Specific Knowledge

The three collections (baseline, smoke, contract) are generated from the
OpenAPI spec, so they validate:

- Request/response **shapes** match the OpenAPI contract
- Required parameters are present and correctly typed
- Documented status codes are reachable on the mock server
- The spec-declared auth scheme (`securitySchemes`) is wired into requests
- Path parameters and query filter shapes match what the spec promises

They do **not** validate:

- Real auth against the actual backend — a PMAK cannot mint OAuth tokens for
  the Payment Refund service itself
- **Business rules that live in spec prose, not as enforceable constraints** —
  e.g., the Refund API's 180-day refund window and per-transaction refund
  limits appear in the spec's `description` fields but aren't expressible as
  JSON Schema constraints. Bespoke contract tests would be needed.
- Cross-service workflows (refund-then-reconcile is a flow involving multiple
  services, not a single-API contract)
- Performance characteristics and load behavior
- Production-only auth integrations (real OAuth providers, mTLS client certs)

This is the right behavior — the action's job is to bootstrap the catalog,
not to replace integration testing.

## 7. Customer-Side Requirements (what the platform team must provide)

1. **Environment URLs per service per env** (dev/staging/prod). The
   `example.com` URLs in this repo come from the brief's specs and need to be
   replaced with real internal URLs.
2. **Auth secret names + provisioning in their secret manager.** For OAuth 2.0,
   client ID and secret. For JWT, signing keys or token-issuance endpoint.
3. **A fine-grained Personal Access Token (or GitHub App)** with `Contents:
   write` and `Workflows: write` on each onboarding repo, surfaced as a repo
   or org secret named `GH_FALLBACK_TOKEN`. Required because the default
   `GITHUB_TOKEN` cannot write to `.github/workflows/` (supply-chain
   protection); the action's `gh-fallback-token` input is the documented
   mechanism for materializing the generated CI test workflow.
   **Caveat for customer:** fine-grained PATs are name-scoped, not ID-scoped —
   if onboarded repos get renamed, the PAT's repository access list must be
   updated or the workflow silently loses authorization. See §11 iteration #5.
4. **CI/CD runner availability.** GitHub Actions runners for most teams. For
   the GitLab CI team mentioned in the brief, the underlying CLIs
   (`postman-bootstrap-action`, `postman-repo-sync-action`) are installable
   via npm and run identically — only the workflow wrapper differs.
5. **Source-IP allowlisting for Postman monitors** hitting network-restricted
   services (especially mTLS endpoints).
6. **Service ownership tags** so onboarded workspaces have known owners for
   governance audits.
7. **Spec-authoring effort** for services without specs — see the Adaptation
   slide in the deck for the traffic-capture / introspection / facilitated-writing
   options.

## 8. Run Instructions

See `SETUP.md` for full first-run setup (~20 minutes from a clean Mac).

Once secrets and variables are configured:

```bash
gh workflow run onboard.yml --repo <OWNER>/jr-cse-payments-postman-onboarding
gh run watch "$(gh run list --repo <OWNER>/jr-cse-payments-postman-onboarding --workflow=onboard.yml --limit 1 --json databaseId --jq '.[0].databaseId')"
```

Then validate the six Postman UI checks per `SETUP.md` Step 7. A green
workflow is necessary but not sufficient — the `POSTMAN_ACCESS_TOKEN` can
silently expire and leave the workspace half-built, so visual verification
is required.

**Note on the generated CI workflow (`payments-tests.yml`).** This is the
*test* pipeline emitted by repo-sync, distinct from the *onboarding* workflow
(`onboard.yml`) run above. Repo-sync serializes it as a single escaped line
(invalid YAML), which GitHub rejects as an "Invalid workflow file" — and it
re-emits the broken file on every run (§11.A #6). Rather than ship that red ✗
next to the green onboarding run, the committed copy is **quarantined verbatim**
at [`evidence/generated-artifacts/payments-tests.yml.txt`](evidence/generated-artifacts/payments-tests.yml.txt)
so GitHub stops parsing it; the [evidence README](evidence/generated-artifacts/README.md)
explains the upstream quirk and that it regenerates each run. Even decoded into
valid YAML it would **not pass yet**, by design, for two reasons: (1) its
"Resolve Postman Resource IDs" step reads `.postman/resources.yaml`, which is
`.gitignore`d and therefore absent on a clean CI checkout, so the step `abort`s
with "Missing Postman resource IDs"; and (2) the collections target
`example.com`, the same placeholder that makes the monitor red by design. This
is the consistent, honest position for the take-home: CI can't pass against
placeholder hosts. The one-line customer fix is to un-ignore `.postman/`
(remove it from `.gitignore` and commit `.postman/resources.yaml`) and supply
real environment URLs + a `POSTMAN_API_KEY` secret. See §11.A #6 and
`issues-log.md` for the full decision record. Refs: submission review item 1,
layer 2.

## 9. Validation Evidence

**Workspace:** [PMT] payment-refund-service
([open in Postman](https://go.postman.co/workspace/1acfc458-0f68-456c-9eed-a1b74292c092))

**First green run after clean rebuild:**
https://github.com/JeremiahJRRoss/jr-cse-payments-postman-onboarding/actions/runs/26606913232
— 38 seconds, single attempt, all six job steps passed.

**Committed artifacts** (committed back by the action's repo-sync phase):
- [`postman/collections/`](postman/collections/) — three git-sync YAML collections (Baseline, Contract, Smoke), the action's canonical format
- [`evidence/generated-artifacts/payments-tests.yml.txt`](evidence/generated-artifacts/payments-tests.yml.txt) — the generated CI workflow, verbatim. Quarantined out of `.github/workflows/` because repo-sync emits it as invalid YAML (see §8 / §11.A #6)

The same three collections are also provided as single-file v2.1 JSON in
`postman/exports/` ([`baseline.json`](postman/exports/baseline.json),
[`contract.json`](postman/exports/contract.json),
[`smoke.json`](postman/exports/smoke.json)) — the JSON the brief asks for. They
are taken from the same run whose YAML is committed here, so the two formats
correspond.

**Walkthrough with all five screenshots:** [`docs/VALIDATION-EVIDENCE.md`](docs/VALIDATION-EVIDENCE.md)

## 10. Rerun / Idempotency Behavior

Observed across multiple no-change runs of `onboard.yml` on `main`
(2026-05-29). The action reuses the **workspace** but **re-creates every object
inside it** on each run:

| Object | Reused on re-run? |
|---|---|
| Workspace | ✅ same ID — matched via the repo↔workspace git-sync link (run log: `Using canonical workspace (linked_match)`) |
| Spec / `[Baseline]` / `[Contract]` / `[Smoke]` collections / mock / monitor | ❌ new IDs every run (log: "Created new mock", "Created new monitor"; the sync commit rewrites each `collection.yaml` `id`) |

The action writes **no GitHub repo variables**. `gh variable list` returns only
the two *input* variables set manually at setup, never updated by a run:

```text
NAME             VALUE         UPDATED
POSTMAN_USER_ID  38960911      ...
REQUESTER_EMAIL  jr@ross.moda  ...
```

There is no stored-ID persistence — an earlier version of this section claimed
one (reuse via written-back repo variables); that mechanism does not exist, was
never verified, and was wrong (see §14). The run log shows no variable-write
attempt and no permission error, so a subsequent run has nothing to read.

**Consequence — confirmed in the workspace:** because the workspace is reused
but its contents are re-created, re-running **accumulates duplicates**. After
five runs the reused workspace held **15 collections** (5× `[Baseline]`, 5×
`[Contract]`, 5× `[Smoke]`) plus multiple duplicate `payment-refund-service -
dev` environments — exactly 3 collections × 5 runs, piled into one workspace
with no cleanup. Screenshot and caption in
[`docs/VALIDATION-EVIDENCE.md`](docs/VALIDATION-EVIDENCE.md). This is the
workspace-sprawl risk (R5/R7) reproduced live.

**How to operate:** treat `onboard.yml` as **create-once per service**. To
re-onboard, clear the prior workspace contents first — delete the older
collections and environments in the Postman UI (right-click → Delete), or nuke
and recreate the workspace — so you return to exactly three collections and one
monitor. A production hardening — a pre-run reuse-or-clean guard keyed on the
workspace git-sync link — is scoped in §12.

## 11. Known Issues and Resolutions

### 11.A — Build iterations (issues that required code/config changes)

Five iterations were needed during initial discovery to land the verified
working workflow; a sixth surfaced during submission review. Each exposed a
real discrepancy between the action's declared contract and its runtime
behavior — documented in `issues-log.md`.

1. **`variables: write` is not a valid GitHub Actions permission key.**
   The job-level `permissions:` block has a fixed allow-list; `variables`
   isn't in it. Repo variables are written via the GitHub API using
   `GITHUB_TOKEN`'s standard scopes, not a job permission. **Fix:** remove
   the line.

2. **`spec-url` is required despite `action.yml` declaring it optional.**
   The orchestrator's `action.yml` says "provide either `spec-url` or
   `spec-path`," but the underlying `postman-bootstrap-action@v0` requires
   `spec-url` at runtime. **Fix:** provide both inputs.

3. **Default `GITHUB_TOKEN` cannot write to `.github/workflows/`.** This is
   by design (supply-chain protection). The action's `gh-fallback-token`
   input exists for exactly this case. **Fix:** create a fine-grained PAT
   with Workflows write, wire it via `gh-fallback-token`.

4. **PAT must be verified before saving as a secret.** A truncated or
   mistyped PAT fails silently — the workflow run reports a 403 with no
   clear cause. **Fix:** always test the PAT against
   `GET https://api.github.com/user` and the target repo's `permissions.push`
   before setting the secret.

5. **Fine-grained PATs are name-scoped, not ID-scoped.** Renaming a repo
   (from `payments-postman-onboarding` to `jr-cse-payments-postman-onboarding`
   during the rebuild) silently broke the PAT's authorization because the
   token's repository access list still pointed at the old name. **Fix:**
   edit the PAT in GitHub settings, add the new repo name to its repository
   access list. Token value unchanged. **Consulting note:** this is a real
   customer-side ask for any engagement where onboarded repos might get
   renamed — surfaced in §7 #3.

6. **Repo-sync emitted the generated CI workflow as one escaped line.**
   `.github/workflows/payments-tests.yml` was committed by the repo-sync phase
   as a single physical line with 90 literal `\n` sequences instead of real
   newlines — invalid YAML (`ScannerError ... line 1, column 49`), which
   GitHub flags as "Invalid workflow file" so it never registers or runs.
   The defect is byte-identical in the companion repo, confirming it's an
   upstream open-alpha serialization bug, not a local commit slip. **Fix:**
   decoded `\n` → real newlines in place, verified lossless against the
   committed original (re-encode == prior commit; 4370→4280 bytes) and that
   it parses with valid `on` + `jobs.test`. Because this is an
   action-generated artifact, the correction is logged here and in
   `issues-log.md` rather than applied silently. **Confirmed recurring:**
   re-running `onboard.yml` regenerates the file escaped again every time
   (observed during the rerun-behavior runs — see §10 and `issues-log.md`).
   **Quarantine decision:** rather than maintain an in-place decode that
   repo-sync overwrites on the next run, the committed copy is parked verbatim
   at `evidence/generated-artifacts/payments-tests.yml.txt` (renamed `.txt` so
   GitHub stops flagging "Invalid workflow file"), with an evidence README
   explaining the upstream quirk. This is a point-in-time cleanup — the file
   regenerates at `.github/workflows/payments-tests.yml` on the next run, so the
   durable fix is still the code-level self-heal (a post-generation decode step
   in `onboard.yml`), the out-of-scope productionizing move noted in §12 #2.
   Refs: submission review item 1, layer 1.

**Pattern across all six:** open-alpha tooling has loose `action.yml`
validation but stricter runtime requirements in the chained downstream
actions, and the sync phase can serialize generated artifacts incorrectly.
The reliable verification is running it (and parsing the output); the
unreliable one is trusting the declared contract.

### 11.B — Annotations on green runs (informational, not blocking)

Three categories of warnings appear on every green run. Documenting them so
an evaluator reading the logs knows I noticed and made deliberate decisions.

**Node.js 20 runtime deprecation (upstream)**

GitHub flags `postman-cs/postman-bootstrap-action@v0` and
`postman-cs/postman-repo-sync-action@v0` (chained internally by the
orchestrator) as running on the deprecated Node.js 20 runtime. Forced
migration to Node 24 is scheduled for June 2026; full removal in September
2026. **This is upstream tooling, not anything in this repo's workflow.**
The action is in open-alpha and the maintainers will publish a Node 24
version before the deadline. I deliberately did not force the opt-in via
`FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` — running an action on a runtime
its maintainers haven't validated could mask issues they should catch in
their own release cycle. The right CSE move is to defer to the upstream
maintainer's timeline.

**OpenAPI spec linting warnings (brief's spec, by design)**

The onboarding action surfaces nine OpenAPI linting warnings on the Payment
Refund spec — all of the form `A schema property should have a $ref
property referencing a reusable schema`. These flag inline schemas in
parameters and response bodies that should be extracted into
`components.schemas` for reusability. The spec ships from the brief
unmodified, so I didn't refactor it; in a real engagement these would be
fed back to the service owner as a spec-quality improvement during the
discovery phase, and they do not block onboarding.

**Workspace role assignment conflict (benign upstream behavior)**

Every run produces a warning of the form:

```
Failed to invite requester: PATCH /workspaces/.../roles failed: 400
Bad Request - Only one role is supported for user 38960911
```

The action attempts to assign the workspace-admin role to the requester
(me), but I already hold the workspace-creator role on the new workspace.
Postman's API rejects the second role assignment. **Functional impact:
zero** — I have admin access via the creator role. In a real engagement
where the requester is a different user from the workspace creator (e.g.,
a CSE provisioning for a service owner), this warning would not appear.

## 12. Trade-offs

What I'd do differently with more time:

- **Author bespoke contract tests for the documented business rules.** The
  180-day refund window and per-transaction refund limits live in spec prose
  only; today's generated collections don't enforce them. A custom Postman
  test script per critical rule would close that gap.
- **Validate the GitLab CI port end-to-end.** The CLIs are documented to work
  identically; ~half a day to confirm in a personal GitLab instance.
- **Build the org-mode path.** The workflow has commented `org-mode: true` /
  `workspace-team-id` lines for org-scoped Postman teams. My team isn't
  org-mode, so I haven't exercised that code path.
- **Surface the PAT-rename issue earlier.** §11.A #5 should be in a
  pre-flight checklist for any customer engagement where repos are likely to
  be renamed during pilot.

### Recommended future changes (productionizing)

The engineering changes I'd make in a real engagement to harden this from a
working take-home into a fleet-ready onboarding pipeline. Each is scoped and
tied to something observed in this repo; **none are implemented here** —
deliberately, the take-home's job is to name and scope them, not build them.
Effort figures are rough estimates, not measured.

1. **Idempotent re-runs — a reuse-or-clean guard.** *(headline)* The action
   reuses the workspace (git-sync `linked_match`) but re-creates the spec, all
   three collections, the mock, and the monitor with new IDs on every run,
   writing no repo variables — so repeated runs accumulate duplicates in one
   workspace (measured: 5 runs → 15 collections; see §10). At 50→300 services
   this scales with re-run *frequency*, not service count, which is why it's the
   headline fix. Wrap the action with a pre-run guard keyed on the same
   workspace git-sync link it already uses: query the workspace and either reuse
   existing child objects by name or delete the prior set before re-creating.
   Alternatives — a post-run dedup step (keep newest, remove prior), or an
   upstream change so the action reuses children the way it already reuses the
   workspace. **Effort:** ~half a day — a guard step in `onboard.yml` wrapping
   the action, or an upstream PR to `postman-api-onboarding-action`. **Interim
   posture:** create-once per service; on re-onboard, clear the workspace first.
2. **Self-healing CI-workflow generation.** Repo-sync emits the generated CI
   workflow as a JSON-escaped single line (literal `\n`), which GitHub rejects
   as an invalid workflow file, and it re-emits the broken file on every re-run
   (§11.A #6). Add a post-generation normalization step in `onboard.yml` that
   decodes `\n`→newlines on the generated workflow after repo-sync commits, so
   re-runs repair it automatically instead of re-emitting the broken file;
   report the escaping upstream. **Effort:** ~1 hour — a post-step in
   `onboard.yml`, plus an upstream issue.
3. **Make the generated CI workflow resolvable on a clean checkout.** The
   generated CI workflow reads `.postman/resources.yaml`, but `.postman/` is
   gitignored, so the resolve step aborts on a fresh CI checkout (§11.A #6
   layer 2; §8). Commit the resolver metadata (after verifying it holds IDs, not
   secrets), or have the workflow derive resource UIDs from the committed
   collection files; the customer's one-line equivalent is to un-ignore
   `.postman/`. **Effort:** small — a `.gitignore` change plus a committed
   metadata file, or upstream.
4. **Drop the public-raw-URL spec dependency.** `onboard.yml` passes a public
   `raw.githubusercontent.com` URL as `spec-url`, so the spec must be pushed to
   a public `main` before a run — even though `action.yml` says `spec-path`
   alone is sufficient (§11.A #2; VERIFIED-NOTES). Once the alpha runtime quirk
   that forced `spec-url` is confirmed resolved, rely on `spec-path` (the
   checked-out file) and remove `spec-url`, eliminating the public-URL and
   push-timing dependency. Re-validate end-to-end before removing. **Effort:**
   trivial once verified — an `onboard.yml` input change.

## 13. Scaling Considerations

The pilot scope is 50 services across one platform team. The pattern scales
because:

- The workflow file is structurally identical across services — exactly **8
  input values** change between services with very different compute and auth
  (enumerated in §5); see `ADAPTATION.md` in the companion repo
- Onboarding is create-once per service: on re-run the action reuses the
  workspace via the git-sync link but re-creates — and therefore accumulates —
  the spec, collections, mock, and monitor (it is **not** idempotent inside the
  workspace; see §10). Cohort rollout should onboard each repo once and avoid
  unnecessary re-runs; cohorting still holds because per-service config is the
  only delta. Re-run safety is a precondition for the cohort math, not a detail:
  making re-runs idempotent (Recommended future changes #1, §12) is what would
  let the platform team re-run during rollout instead of onboarding once and
  freezing
- Customer-side requirements (§7) are knowable upfront, so cohort planning
  reduces to bucketing services by spec freshness + auth pattern + CI platform

The 90-day cohort roadmap and ROI model live in the presentation deck
(outside this repo). Headline: pilot proves the pattern; org-wide rollout
to ~300 services makes the catalog real; the catalog unlocks Agent Mode
queries and dependency graphs.

## 14. AI Assistance and Manual Validation

### What AI generated
- Initial draft of `.github/workflows/onboard.yml`, including the job
  structure, permissions block, and the `uses:` line for the orchestrator.
  Tool: Claude.
- The `env-runtime-urls-json` JSON value, populated from the spec's `servers`
  block.
- Initial draft of this README's structure and most prose.
- The `service.config.yml` documentation file.

### What I validated manually
- Read `action.yml` for `postman-api-onboarding-action@v0` directly in the
  upstream repo. Verified inputs against the live schema.
- Ran the workflow against a real Postman workspace; confirmed each artifact
  appeared in the UI through the six checks in `SETUP.md` Step 7.
- Tested the PAT against `https://api.github.com/user` and the specific
  repo's `permissions.push` field before saving it as a secret — this caught
  one silently-truncated paste during initial setup.
- Tracked the source of every warning annotation in green runs (§11.B) to
  distinguish upstream behavior from issues I introduced.

### What AI got wrong (and I corrected)
- **AI added `variables: write` to the job permissions block.** GitHub Actions
  has a fixed allow-list of permission keys; `variables` isn't one. Caught on
  the first `gh workflow run` with HTTP 422 "Unexpected value 'variables'."
  **Fix:** removed the line. Logged in `issues-log.md` 2026-05-28 entry #1.
- **AI used `spec-path` alone and dropped `spec-url`.** Despite the
  orchestrator's `action.yml` declaring both as optional, the downstream
  `postman-bootstrap-action@v0` requires `spec-url` at runtime. Caught on the
  second run with "Input required and not supplied: spec-url." **Fix:** added
  `spec-url` while keeping `spec-path`. Logged in entry #2.
- **AI omitted `gh-fallback-token` entirely.** The action's documented escape
  hatch for `.github/workflows/` writes was missing from the initial draft.
  Caught on the third run with a 422 "without `workflows` permission" error
  during the commit-back step. **Fix:** created a fine-grained PAT with
  Contents/Workflows/Actions/Variables write, wired it via
  `gh-fallback-token`. Logged in entry #3.
- **AI claimed `ci-workflow-path` default was `onboard.yml`** (collision risk).
  Real default per `action.yml` is `.github/workflows/ci.yml` — no collision
  risk with `onboard.yml`. Documented in `VERIFIED-NOTES.md`.
- **AI claimed `v0.x.y` immutable tags exist** for pinning. Real state: only
  `@v0` rolling exists; no immutable tags are published yet. Documented in
  `VERIFIED-NOTES.md`. Used `@v0`.
- **AI did not anticipate that renaming the repo would break PAT scoping.**
  Fine-grained PATs are name-scoped at the GitHub API level — renaming the
  repo silently invalidated the PAT's repository access list. Discovered on
  the first run after the rebuild's rename from `payments-postman-onboarding`
  to `jr-cse-payments-postman-onboarding`. **Fix:** added new repo name to
  the PAT's access list (no token regeneration needed). Logged as iteration #5.
- **AI claimed re-runs are idempotent via persisted repo variables.** Observed
  across multiple no-change runs (2026-05-29): the action writes **no** repo
  variables (`gh variable list` shows only the two input variables, no
  Postman-resource IDs); on re-run it reuses the workspace (`linked_match`) but
  re-creates the spec, three collections, mock, and monitor with new IDs —
  **accumulating duplicates** (five runs produced 15 collections in one
  workspace; see §10 + `docs/VALIDATION-EVIDENCE.md`). Corrected README §10/§13
  and SETUP §7/§9/troubleshooting to the observed behavior. The variables story
  was never verified before — fixed here. Logged in `issues-log.md` 2026-05-29.

### Why this matters
AI accelerated drafting by maybe 5x and was useful for the prose and the
non-load-bearing scaffolding. On the load-bearing pieces — the workflow file
itself, the action contract — it confidently invented inputs that don't exist
and dropped inputs that do. Every load-bearing claim was verified against the
upstream `action.yml` or against a real run. **The pattern: AI does first
drafts; humans verify against authoritative sources; both contributions are
documented honestly.** Open-alpha tooling makes this verification especially
important because the declared contract (`action.yml` README) lags behind
the runtime contract (what the chained downstream actions require).

## References

- **Brief:** https://github.com/postman-cs/cse-exercise
- **Orchestrator action:** https://github.com/postman-cs/postman-api-onboarding-action
- **Postman Learning Center:** https://learning.postman.com/
- **Postman API authentication:** https://learning.postman.com/docs/developer/postman-api/authentication/
