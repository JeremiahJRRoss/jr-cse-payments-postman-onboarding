# BLUEPRINT — jr-cse-payments-postman-onboarding

> **Repo role:** Primary working implementation per the Postman CSE take-home brief.
> Houses the canonical onboarding workflow + generated artifacts + the README
> the evaluator reads first.
>
> **Companion:** `loan-origination-postman-onboarding` covers adaptation analysis.
>
> **How to use this file:**
> - Each PR has detailed instructions AND a copy-paste **execute prompt** at the bottom.
> - To launch a PR: open this repo in Claude Code (`claude`), copy the execute
>   prompt for that PR, fill in the placeholders, paste.
> - Claude Code auto-loads `CLAUDE.md` and reads `BLUEPRINT.md` when prompted.

---

## 0. Source of truth

The strategic plan (`postman-cse-project-plan-CLD.md`) lives in the candidate's
claude.ai project workspace, **not in this repo**. Section references like §4.2
below point to that plan. This BLUEPRINT.md is self-contained for build work.

---

## 1. Critical constraints

1. **Mixed infrastructure** — Lambda+API Gateway (this service), ECS+ALB, k8s. Auth: OAuth, JWT, mTLS, API keys.
2. **Mixed CI/CD** — GitHub Actions (this repo) + one team on GitLab CI that won't switch.
3. **Inconsistent OpenAPI coverage** — across the org. This service's spec is current.
4. **Platform team stretched thin** — no new tools to maintain after CSE leaves.
5. **Governance, not utilization** — value story is catalog + discoverability + automated coverage.

---

## 2. Decisions affecting this repo

| # | Decision | Why it matters here |
|---|---|---|
| D1 | Payments + Loan Origination as the two pilot services | This repo = Payments |
| D2 | Use `postman-cs/postman-api-onboarding-action@v0` directly | Bypass starter; cleaner workflow + full input surface |
| D3 | One repo per service | This BLUEPRINT covers one repo only |
| D8 | AI disclosure is specific, not generic | Format in PR 5 |

The `@v0` pin matters: plan §14's example referenced `@v1`, which does not exist.
Use `@v0` (rolling open-alpha channel) or `v0.x.y` (immutable). Verify the current
immutable tag at `https://github.com/postman-cs/postman-api-onboarding-action/releases`
during PR 1.

---

## 3. Pre-PR setup (manual, before any Claude Code work)

| # | Step | Verify |
|---|---|---|
| S1 | Postman Enterprise trial active | Workspace creation works in the UI |
| S2 | `POSTMAN_API_KEY` (PMAK) generated, copied | `PMAK-...` value in hand |
| S3 | Postman CLI installed; `postman login` complete | Browser auth flow finished |
| S4 | Extract `POSTMAN_ACCESS_TOKEN`:<br>`cat ~/.postman/postmanrc \| jq -r '.login._profiles[].accessToken'` | Token returned, copied |
| S5 | `gh` CLI authenticated | `gh auth status` shows logged in |
| S6 | This repo created on GitHub, cloned locally | `git remote -v` shows the right origin |
| S7 | Both secrets set: `gh secret set POSTMAN_API_KEY` + `gh secret set POSTMAN_ACCESS_TOKEN` | `gh secret list` shows both |
| S8 | Your numeric Postman user ID grabbed:<br>`curl -H "X-API-Key: $POSTMAN_API_KEY" https://api.getpostman.com/me \| jq '.user.id'` | Numeric ID in hand |

If any step fails, fix before PR 0. Don't paper over auth issues.

---

## 4. PR sequence

Six PRs. PR 0 is pure scaffolding (no workflow execution). PR 1 is the first risky
thing. PRs 2-5 build evaluator-facing artifacts.

Each PR section below has:
1. **Goal**
2. **Prerequisites**
3. **Detailed steps**
4. **Acceptance gate** (PR is not done until all boxes checked)
5. **Commit message template**
6. **Execute prompt** (copy-paste-able into `claude`; fill placeholders before pasting)

---

### PR 0 — Scaffolding

**Goal:** Lay down every static file Claude Code needs to operate. No workflow
execution. After this PR, `tree` matches the layout below.

**Prerequisites:** S1-S8 complete. Empty repo cloned locally. `CLAUDE.md` and
`BLUEPRINT.md` already committed (you put them there to bootstrap Claude Code).

**Target file tree:**

```
.
├── CLAUDE.md                      ← already exists
├── BLUEPRINT.md                   ← already exists
├── README.md                      ← stub (filled in PR 3-5)
├── .gitignore
├── .claude/
│   ├── settings.json
│   └── commands/
│       ├── log-issue.md
│       ├── run-and-watch.md
│       └── validate-postman.md
├── .github/
│   └── workflows/
│       └── .gitkeep               ← placeholder; workflow added in PR 1
├── specs/
│   └── payment-refund-api-openapi.yaml   ← copied from brief, unmodified
├── postman/
│   └── collections/
│       └── .gitkeep               ← repo-sync populates this in PR 1
├── service.config.yml
└── issues-log.md
```

**Detailed steps:**

1. Create the file tree above.
2. **Spec file** — fetch from the brief repo:
   ```bash
   mkdir -p specs
   curl -fsSL -o specs/payment-refund-api-openapi.yaml \
     https://raw.githubusercontent.com/postman-cs/cse-exercise/main/specs/payment-refund-api-openapi.yaml
   ```
   Verify the file is non-empty and starts with `openapi: 3.0.3`.
3. **`.gitignore`** — standard Node + OS + env patterns:
   ```
   node_modules/
   dist/
   build/
   .DS_Store
   Thumbs.db
   .env
   .env.*
   .vscode/
   .idea/
   *.log
   ```
4. **`.claude/settings.json`** — see §5 below.
5. **Slash commands** (`.claude/commands/*.md`) — see §6 below.
6. **`service.config.yml`** — see §7 below.
7. **`issues-log.md`** — see §8 below.
8. **`README.md`** stub:
   ```markdown
   # jr-cse-payments-postman-onboarding

   Postman API onboarding workflow for the Payments service. Primary working
   implementation for the Postman CSE take-home exercise.

   See companion repo `loan-origination-postman-onboarding` for adaptation analysis.

   Detailed README populated in PR 3-5.
   ```
9. Run `tree -a -I '.git'` and confirm the layout matches.
10. Commit and push.

**Acceptance gate:**

- [ ] `tree -a -I '.git'` matches the target layout
- [ ] Spec file is verbatim from brief (no modifications)
- [ ] `gh repo view --web` URL shows everything on remote
- [ ] No secrets in any committed file

**Commit message:**

```
PR 0: Scaffold payments onboarding repo

- Add .claude/ commands and settings for Claude Code
- Drop in payment-refund OpenAPI spec from brief
- Add service.config.yml documenting per-service inputs
- Stub README + issues log for PR 1+

Refs: BLUEPRINT.md §4 PR 0
```

#### Execute prompt for PR 0

```
Execute PR 0 (Scaffolding) from BLUEPRINT.md.

Context:
- Empty repo. CLAUDE.md and BLUEPRINT.md are already committed.
- Manual setup S1-S8 complete; secrets confirmed via `gh secret list`.

Steps (per BLUEPRINT.md §4 PR 0):
1. Fetch the spec from the brief:
   curl -fsSL -o specs/payment-refund-api-openapi.yaml \
     https://raw.githubusercontent.com/postman-cs/cse-exercise/main/specs/payment-refund-api-openapi.yaml
   Verify it's non-empty and starts with `openapi: 3.0.3`.
2. Create everything else in the §4 PR 0 file tree.
3. Source contents from:
   - .claude/settings.json → BLUEPRINT.md §5
   - .claude/commands/*.md → BLUEPRINT.md §6
   - service.config.yml → BLUEPRINT.md §7
   - issues-log.md → BLUEPRINT.md §8
   - README.md stub → BLUEPRINT.md §4 PR 0 step 8
   - .gitignore → BLUEPRINT.md §4 PR 0 step 3
4. Before committing: run `tree -a -I '.git'` and show me the output.
5. After my approval: commit using the message template in BLUEPRINT.md §4 PR 0,
   then push.
6. Confirm acceptance gate (4 boxes) in BLUEPRINT.md §4 PR 0.

Do NOT push without showing me the file tree first.
```

---

### PR 1 — Workflow file + first green run

**Goal:** A working `.github/workflows/onboard.yml` that runs end-to-end against
a real Postman workspace and produces all expected artifacts.

**Prerequisites:** PR 0 merged. Both secrets confirmed via `gh secret list`.

**Action contract** (verified against upstream README; verify against the live
`action.yml` before committing):

Required inputs:
- `project-name` (string)
- `spec-url` (string — raw HTTPS URL to OpenAPI doc)
- `postman-api-key` (secret)

Strongly recommended (silently degrade without them):
- `postman-access-token` (secret — without it, bifrost git sync, governance
  assignment, and system env association silently skip)
- `github-token: ${{ secrets.GITHUB_TOKEN }}`

Optional but populate for credible demo:
- `domain` — e.g., `payments`
- `domain-code` — short prefix, e.g., `PMT`
- `requester-email` — your email
- `workspace-admin-user-ids` — your numeric Postman user ID
- `environments-json` — defaults to `["prod"]`; populate from spec `servers`
- `env-runtime-urls-json` — map env slug → server URL
- `ci-workflow-path` — set explicitly to avoid the repo-sync action overwriting your onboarding workflow

Permissions on the job:
- `actions: write`
- `contents: write`

**Workflow file shape** (draft this in PR 1; verify against the upstream
`action.yml` before pushing):

```yaml
name: Onboard Payments Service to Postman

on:
  workflow_dispatch: {}
  push:
    paths:
      - 'specs/payment-refund-api-openapi.yaml'
      - '.github/workflows/onboard.yml'

jobs:
  onboard:
    runs-on: ubuntu-latest
    permissions:
      actions: write
      contents: write
    steps:
      - uses: actions/checkout@v4

      - name: Onboard service into Postman
        uses: postman-cs/postman-api-onboarding-action@v0
        with:
          project-name: payment-refund-service
          domain: payments
          domain-code: PMT
          requester-email: <YOUR_EMAIL>
          workspace-admin-user-ids: "<YOUR_POSTMAN_USER_ID>"
          spec-url: https://raw.githubusercontent.com/<OWNER>/jr-cse-payments-postman-onboarding/main/specs/payment-refund-api-openapi.yaml
          environments-json: '["prod","uat","qa","dev"]'
          env-runtime-urls-json: |
            {
              "prod": "https://api.payments.example.com/v2",
              "uat":  "https://api-uat.payments.example.com/v2",
              "qa":   "https://api-qa.payments.example.com/v2",
              "dev":  "https://api-dev.payments.example.com/v2"
            }
          generate-ci-workflow: true
          ci-workflow-path: .github/workflows/payments-tests.yml
          postman-api-key: ${{ secrets.POSTMAN_API_KEY }}
          postman-access-token: ${{ secrets.POSTMAN_ACCESS_TOKEN }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

**Detailed steps:**

1. `git checkout -b pr-1-workflow`.
2. Fetch upstream `action.yml` to verify input names:
   ```bash
   gh api repos/postman-cs/postman-api-onboarding-action/contents/action.yml \
     --jq .content | base64 -d > /tmp/upstream-action.yml
   head -100 /tmp/upstream-action.yml
   ```
   If any input name in the shape above diverges from the live `action.yml`,
   log via `/log-issue` and correct.
3. Check the latest immutable tag:
   ```bash
   gh api repos/postman-cs/postman-api-onboarding-action/tags --jq '.[0].name'
   ```
   If a `v0.x.y` tag exists, prefer it. Log the choice in `issues-log.md`.
4. Draft `.github/workflows/onboard.yml` from the shape above, substituting:
   - `<YOUR_EMAIL>` → your email
   - `<YOUR_POSTMAN_USER_ID>` → your numeric Postman user ID
   - `<OWNER>` → your GitHub username or org
5. Commit and push (push trigger fires the workflow automatically).
6. `/run-and-watch`.
7. If failed: diagnose. Log every iteration via `/log-issue`. **Do NOT squash**
   — iteration history is evidence of debugging.
8. When green, capture all surfaced outputs (workspace name/URL, spec ID,
   collection IDs, environment IDs, monitor ID, mock URL).

**Acceptance gate:**

- [ ] Latest run on `main` is green
- [ ] Postman UI: workspace exists with expected naming
- [ ] Postman UI: spec is in Spec Hub and parses
- [ ] Postman UI: three collections exist (baseline, smoke, contract)
- [ ] Postman UI: environments populated (one per env in `environments-json`)
- [ ] Postman UI: monitor created
- [ ] Mock server URL captured
- [ ] `postman/collections/` contains JSON exports (committed by repo-sync)
- [ ] `issues-log.md` updated with every iteration

**Commit message (final commit only):**

```
PR 1: Working onboarding workflow for payments service

- .github/workflows/onboard.yml calling postman-api-onboarding-action@v0
- environments-json + env-runtime-urls-json wired from spec servers block
- ci-workflow-path set to avoid collision with onboarding workflow
- First green run: <link>

Action inputs verified against upstream action.yml on <date>.

Refs: BLUEPRINT.md §4 PR 1
```

#### Execute prompt for PR 1

```
Execute PR 1 (Workflow file + first green run) from BLUEPRINT.md.

FILL IN BEFORE PASTING:
- YOUR_EMAIL: <your email>
- YOUR_POSTMAN_USER_ID: <numeric ID from `curl -H "X-API-Key: $POSTMAN_API_KEY" https://api.getpostman.com/me | jq '.user.id'`>
- OWNER: <your github username or org>

Context:
- PR 0 merged. Scaffolding in place.
- Both secrets set: verify with `gh secret list` before starting.

Steps (per BLUEPRINT.md §4 PR 1):
1. Create branch `pr-1-workflow`.
2. Verify secrets: `gh secret list`. STOP if either is missing.
3. Fetch upstream action.yml; compare inputs to the shape in BLUEPRINT.md §4 PR 1.
   Log any drift via /log-issue.
4. Check latest immutable tag (`gh api repos/postman-cs/postman-api-onboarding-action/tags --jq '.[0].name'`).
   If a `v0.x.y` exists, prefer it. Log the choice.
5. Draft .github/workflows/onboard.yml from the BLUEPRINT shape, substituting:
   - <YOUR_EMAIL>
   - <YOUR_POSTMAN_USER_ID>
   - <OWNER> in spec-url with OWNER/jr-cse-payments-postman-onboarding
6. Show me the workflow file BEFORE committing.
7. After my approval: commit, push.
8. Run /run-and-watch.
9. If failed: diagnose, propose smallest fix, get my approval, iterate. Log each
   iteration via /log-issue. Do NOT squash.
10. If green: capture all outputs (workspace name/URL, spec ID, collection IDs,
    env IDs, monitor ID, mock URL) and remind me to validate in Postman UI.

Stop and confirm with me before declaring done. Then walk the acceptance gate
(9 boxes) in BLUEPRINT.md §4 PR 1.
```

---

### PR 2 — Idempotency test + Postman validation evidence

**Goal:** Document what actually happens on re-run and on spec change.

**Prerequisites:** PR 1 merged. Latest run green. User has manually eyeballed
the Postman UI.

**Detailed steps:**

1. `git checkout -b pr-2-validation`.
2. Re-run workflow with no changes: `gh workflow run onboard.yml`. `/run-and-watch`.
3. Observe Postman UI vs the PR 1 baseline:
   - Workspace: same (reused), or new (duplicated)?
   - Spec in Spec Hub: updated in place, or duplicated?
   - Collections: same IDs (overwritten), or new IDs (duplicated)?
   - Environments: same, updated, or duplicated?
   - Monitor: same, recreated, or duplicated?
4. Write `docs/idempotency.md`:
   - Section 1: What made it idempotent (e.g., the action reads repo variables
     like `POSTMAN_WORKSPACE_ID` set by the first run)
   - Section 2: Observed deltas table
   - Section 3: One-sentence answer for Q&A Q5
5. Capture Postman UI screenshots (user pastes or saves to `docs/screenshots/`):
   - Workspace home
   - Spec Hub view
   - One collection expanded
   - Environment variables view
   - Monitor results view
6. Spec-change scenario: bump `info.version` in the spec from `2.1.0` → `2.1.1`,
   commit, push, `/run-and-watch`. Document observed behavior in
   `docs/idempotency.md`.
7. Commit.

**Acceptance gate:**

- [ ] `docs/idempotency.md` exists with concrete observations
- [ ] `docs/screenshots/` has at least 5 PNGs
- [ ] Q&A Q5 answer is one sentence, grounded in observation
- [ ] Spec-change re-run behavior documented

**Commit message:**

```
PR 2: Idempotency + spec-change behavior documented

- Re-run with no changes: <observed behavior>
- Spec version bump: <observed behavior>
- Screenshots in docs/screenshots/

Refs: BLUEPRINT.md §4 PR 2, plan §12 Q5
```

#### Execute prompt for PR 2

```
Execute PR 2 (Idempotency + validation evidence) from BLUEPRINT.md.

Context:
- PR 1 merged. Latest run green.
- I've manually validated Postman UI (workspace, spec, collections, envs, monitor).

Steps (per BLUEPRINT.md §4 PR 2):
1. Create branch `pr-2-validation`.
2. Trigger re-run: `gh workflow run onboard.yml`. Then /run-and-watch.
3. After green: walk me through comparing each Postman artifact (workspace, spec,
   3 collections, envs, monitor) to its PR 1 state. Document what changed.
4. Write docs/idempotency.md with three sections per BLUEPRINT.md §4 PR 2 step 4.
5. Walk me through capturing screenshots. I'll save them to docs/screenshots/.
6. Spec-change scenario:
   - Edit specs/payment-refund-api-openapi.yaml: bump info.version 2.1.0 → 2.1.1.
   - Commit, push, /run-and-watch.
   - Document observed behavior in docs/idempotency.md.
7. Commit per BLUEPRINT.md §4 PR 2 message template.
8. Confirm acceptance gate (4 boxes).

Do NOT generate screenshots from memory. I'll provide them.
```

---

### PR 3 — README sections 1-4

**Goal:** Top half of the README — overview, structure, architecture.

**Prerequisites:** PR 2 merged.

**Sections to write (in `README.md`):**

1. **Overview** — one paragraph: what this repo does, which service, which
   Postman workspace it manages.
2. **Services Onboarded** — Payments (this repo) + cross-link to
   `loan-origination-postman-onboarding`.
3. **Repository Structure** — annotated tree, explain each top-level dir.
4. **Workflow Architecture** — the action chain:
   - `postman-api-onboarding-action` orchestrates
   - `postman-bootstrap-action` creates workspace, spec, collections, envs
   - `postman-repo-sync-action` exports collections, generates CI workflow,
     creates mocks + monitors

**Acceptance gate:**

- [ ] Sections 1-4 present and readable by a non-Postman engineer
- [ ] Cross-link to loan-origination repo present in section 2
- [ ] Workflow architecture matches `.github/workflows/onboard.yml`

**Commit message:**

```
PR 3: README sections 1-4 (overview, structure, architecture)

Refs: BLUEPRINT.md §4 PR 3, plan §13
```

#### Execute prompt for PR 3

```
Execute PR 3 (README sections 1-4) from BLUEPRINT.md.

FILL IN BEFORE PASTING:
- LOAN_ORIGINATION_REPO_URL: <https://github.com/<owner>/loan-origination-postman-onboarding>

Context:
- PR 2 merged. docs/idempotency.md and docs/screenshots/ exist.

Steps (per BLUEPRINT.md §4 PR 3):
1. Replace README.md stub with sections 1-4.
2. For section 4 (Workflow Architecture): pull facts from .github/workflows/onboard.yml directly. Do NOT invent.
3. Section 2: use LOAN_ORIGINATION_REPO_URL for the cross-link.
4. Show me the README before committing. Summarize each section.
5. After approval: commit per BLUEPRINT.md §4 PR 3 message template, push.
6. Confirm acceptance gate (3 boxes).
```

---

### PR 4 — README sections 5-9 (evaluator-relevant)

**Goal:** Maps directly to the brief's evaluation criteria.

**Prerequisites:** PR 3 merged.

**Sections to write:**

5. **Universal Pattern vs Per-Service Differences** — a table. Don't copy plan
   §4.4's skeleton; populate from observation. Starter rows:
   - Action chain (universal)
   - Spec upload to Spec Hub (universal)
   - Collection generation (universal)
   - Workspace name (per service)
   - Domain code (per service)
   - Environment URLs (per service)
   - Auth pattern in spec (per service)
   - `ci-workflow-path` (per service, collision avoidance)

6. **What Generated Checks Validate vs What Still Needs Service-Specific Knowledge** —
   one paragraph + two columns. The brief explicitly asks for this.

   What spec-derived collections prove:
   - Request/response shape matches the OpenAPI contract
   - Required parameters present and typed correctly
   - Status codes documented and reachable on the mock
   - Auth scheme declared in spec is wired into the request

   What still needs service-specific knowledge:
   - Real auth credentials (PMAK-only can't issue OAuth tokens for the backend)
   - Business-rule validation (180-day refund window, max 5 refunds per txn —
     these live in spec prose, not as enforceable constraints)
   - Cross-service workflows (refund-then-monitor is a flow, not a contract)
   - Performance characteristics and load thresholds

7. **Customer-Side Requirements** — six items (plan §7.7):
   1. Environment URLs per service per env
   2. Auth secret names + provisioning in customer secret manager
   3. CI/CD runner availability (GH Actions + GitLab runners for GitLab team)
   4. Source IP allowlisting for monitors hitting network-restricted services
   5. Service ownership tags for governance
   6. Spec authoring effort for services without specs

8. **Run Instructions** — copy-paste runnable:
   ```bash
   gh secret set POSTMAN_API_KEY --repo <owner>/<repo>
   gh secret set POSTMAN_ACCESS_TOKEN --repo <owner>/<repo>
   gh workflow run onboard.yml
   gh run watch
   ```
   Then UI validation steps.

9. **Validation Evidence** — link to the green run, link to committed
   `postman/collections/`, embed 2-3 screenshots from `docs/screenshots/`.

**Acceptance gate:**

- [ ] Section 5 table populated from observation, not prediction
- [ ] Section 6 explicitly answers the brief's "what checks prove vs not" question
- [ ] Section 7 has six items
- [ ] Section 8 is copy-paste runnable

**Commit message:**

```
PR 4: README sections 5-9 (universal pattern, validation gaps, customer asks, evidence)

Refs: BLUEPRINT.md §4 PR 4, plan §4.4 §7.7 §13
```

#### Execute prompt for PR 4

```
Execute PR 4 (README sections 5-9) from BLUEPRINT.md.

Context:
- PR 3 merged. Sections 1-4 of README exist.
- Observations from PR 1-2 available in issues-log.md and docs/idempotency.md.

Steps (per BLUEPRINT.md §4 PR 4):
1. Write section 5 (Universal vs Per-Service). Start from the example rows in
   BLUEPRINT.md §4 PR 4 step 5 but VERIFY each against:
   - The actual workflow file inputs
   - The actual spec
   - What I observed in Postman UI
   Drop rows I can't verify; add rows that surfaced in PR 1-2 that I missed.
2. Write section 6 (Generated checks vs service-specific knowledge). Adapt to
   THIS spec's specifics (the 180-day refund window, max-5-refunds rule).
3. Write section 7 (Customer-Side Requirements) — six items per BLUEPRINT.md §4 PR 4 step 7.
4. Write section 8 (Run Instructions). Test by reading aloud: "could someone
   without my context follow this?"
5. Write section 9 (Validation Evidence). Embed run links + 2-3 screenshots
   from docs/screenshots/.
6. Show me each section before moving on. Commit per section OR all together,
   based on whether sections need iteration.
7. Final commit per BLUEPRINT.md §4 PR 4 message template.
8. Confirm acceptance gate (4 boxes).
```

---

### PR 5 — README sections 10-14 + AI disclosure

**Goal:** Close out with honest-documentation sections. AI disclosure must be
specific to pass the brief's grading.

**Prerequisites:** PR 4 merged.

**Sections to write:**

10. **Rerun / Idempotency Behavior** — paste the one-line answer from
    `docs/idempotency.md`; link to the full doc.

11. **Known Issues and Resolutions** — synthesize from `issues-log.md`. For each
    issue ≥1 hour of friction OR ≥1 AI correction: one paragraph (tried, error,
    changed, source of fix).

12. **Trade-offs** — what you'd do with more time. Candidates:
    - Author bespoke contract tests for documented business rules (180-day
      window, max 5 refunds) — currently only spec-shape is validated
    - Set up source-IP allowlisting workflow for monitors hitting mTLS endpoints
    - Translate to GitLab CI before claiming full portability
    - Patch generated multipart-upload request post-generation if it's wrong

13. **Scaling Considerations** — one paragraph pointing at the deck's 90-day
    roadmap.

14. **AI Assistance and Manual Validation** — see template below.

**AI disclosure template** (use this voice):

```markdown
## AI Assistance and Manual Validation

### What AI generated
- Initial draft of `.github/workflows/onboard.yml`, including job structure and
  the `uses:` line. Tool: Claude.
- The `env-runtime-urls-json` content, generated from the spec's `servers` block.
- Initial draft of this README's structure and most prose.

### What I validated manually
- Read `action.yml` for `postman-api-onboarding-action@v0` directly in the
  upstream repo to confirm input schema.
- Ran the workflow against my real Postman workspace and confirmed every
  expected artifact appeared in the UI.
- Re-ran the workflow with no changes to test idempotency (see docs/idempotency.md).
- Diffed Claude's first README draft against the actual repo state to remove
  invented details.

### What AI got wrong (and I corrected)
- AI referenced action version `@v1`. Real version is `@v0` (open-alpha).
  Verified at https://github.com/postman-cs/postman-api-onboarding-action/releases.
- AI initially suggested storing the API key as an `env:` variable. Moved to
  GitHub repo secrets via `${{ secrets.POSTMAN_API_KEY }}`.
- AI omitted `postman-access-token`. Without it, bifrost git sync, governance
  assignment, and system env association silently skip during onboarding.
  Added as a required secret with a README note about session-scope expiration.
- AI defaulted `environments-json` to `["prod"]`. Populated all four envs from
  the spec's `servers` block.
- (Add other specific corrections from your iterations.)

### Why this matters
AI accelerated the YAML and prose drafting by ~5x. It also confidently invented
a version tag and silently dropped a required input. Manual validation against
the upstream `action.yml` caught both. The pattern: AI does the bulk drafting,
human verifies against authoritative sources, both work get logged honestly.
```

**Acceptance gate:**

- [ ] Section 10 grounded in `docs/idempotency.md`
- [ ] Section 11 has ≥3 entries from `issues-log.md`
- [ ] Section 14 has ≥3 specific "what AI got wrong" entries
- [ ] Cross-link to loan-origination repo present in trade-offs

**Commit message:**

```
PR 5: README sections 10-14 (issues, trade-offs, AI disclosure)

Refs: BLUEPRINT.md §4 PR 5, plan §13 §14
```

#### Execute prompt for PR 5

```
Execute PR 5 (README sections 10-14 + AI disclosure) from BLUEPRINT.md.

Context:
- PR 4 merged. README sections 1-9 exist.
- issues-log.md and docs/idempotency.md populated.

Steps (per BLUEPRINT.md §4 PR 5):
1. Section 10: one-sentence rerun summary; link to docs/idempotency.md.
2. Section 11: read issues-log.md. For each issue worth ≥1 hour of friction or
   that involved an AI correction, write a paragraph (tried / error / changed /
   source). Aim for ≥3 entries.
3. Section 12: trade-offs. Start from candidates in BLUEPRINT.md §4 PR 5; add
   anything from this build that surprised us.
4. Section 13: one paragraph on scaling — point at the deck's 90-day roadmap
   (deck lives outside this repo).
5. Section 14: AI disclosure. Use template in BLUEPRINT.md §4 PR 5. CRITICAL:
   every "what AI got wrong" entry must reference a specific iteration from
   issues-log.md. NO generic platitudes. NO "AI helped with YAML." NO inventing
   failures that didn't happen.
6. Final read-through: does the README feel evaluator-ready? Flag anything
   still aspirational.
7. Commit per BLUEPRINT.md §4 PR 5 message template.
8. Confirm acceptance gate (4 boxes). After PR 5: repo URL is ready to submit.
```

---

## 5. `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Bash(gh:*)",
      "Bash(git:*)",
      "Bash(cat:*)",
      "Bash(ls:*)",
      "Bash(rg:*)",
      "Bash(grep:*)",
      "Bash(jq:*)",
      "Bash(tree:*)",
      "Bash(mkdir:*)",
      "Bash(touch:*)",
      "Bash(curl:*)",
      "Bash(diff:*)",
      "Bash(head:*)",
      "Bash(tail:*)",
      "Bash(base64:*)",
      "Read(*)",
      "Write(*)",
      "Edit(*)"
    ]
  }
}
```

Adjust if Claude Code hits approval prompts on benign commands.

---

## 6. Slash command files

### `.claude/commands/log-issue.md`

```markdown
Append a new entry to issues-log.md in the format below.

Ask me:
1. What was I trying to do?
2. What was the exact error or unexpected behavior?
3. What did I change?
4. Did AI suggest the fix, or did I find it via docs / action.yml / experiment?

Format the entry:

### YYYY-MM-DD — <one-line summary>
**Tried:** ...
**Error:** ```<exact text>```
**Changed:** ...
**Source of fix:** AI / Docs / action.yml / experiment / other

Append; do not overwrite. Commit nothing unless I ask.
```

### `.claude/commands/run-and-watch.md`

```markdown
Trigger the onboarding workflow and watch it to completion.

Steps:
1. `gh workflow run onboard.yml`
2. Wait 5 seconds, then:
   `gh run list --workflow=onboard.yml --limit 1 --json databaseId --jq '.[0].databaseId'`
   to get the run ID.
3. `gh run watch <run-id>` until it finishes.
4. If failed: `gh run view <run-id> --log-failed`. Summarize the failure and
   propose the smallest possible fix. Do NOT apply without my confirmation.
5. If succeeded: extract outputs (workspace name, spec ID, collection IDs, env
   IDs, monitor ID, mock URL) and remind me to validate in the Postman UI.
```

### `.claude/commands/validate-postman.md`

```markdown
Walk me through validating Postman artifacts in the UI.

Read the latest green run on onboard.yml. Extract:
- Workspace name / URL
- Spec ID
- Collection IDs (baseline, smoke, contract)
- Environment IDs
- Monitor ID
- Mock server URL

Present as a numbered list with the URL for each. After each item, wait for me
to type "ok" or paste an issue. If issue: invoke the /log-issue flow.

End with: "Ready to re-run for idempotency? (Y/n)"
```

---

## 7. `service.config.yml` template

```yaml
# service.config.yml — per-service onboarding inputs (no secrets)
#
# Documents inputs passed to postman-api-onboarding-action.
# Secrets (POSTMAN_API_KEY, POSTMAN_ACCESS_TOKEN) live in GitHub repo secrets.

service:
  name: payment-refund-service
  domain: payments
  domain_code: PMT
  requester_email: <YOUR_EMAIL>

spec:
  path: specs/payment-refund-api-openapi.yaml

environments:
  - prod
  - uat
  - qa
  - dev

environment_urls:
  prod: https://api.payments.example.com/v2
  uat:  https://api-uat.payments.example.com/v2
  qa:   https://api-qa.payments.example.com/v2
  dev:  https://api-dev.payments.example.com/v2

ci_workflow:
  generate: true
  path: .github/workflows/payments-tests.yml

notes:
  - All environment URLs are example.com (per provided spec); not real.
  - In a real engagement, these come from the platform team and are likely
    behind customer auth + network controls.
```

This file is documentation. The workflow doesn't read this YAML at runtime; the
agent uses it as a single-source reference when authoring the workflow.

---

## 8. `issues-log.md` template

```markdown
# Issues Log — jr-cse-payments-postman-onboarding

> Honest documentation of issues encountered during build. Per plan §4.6.
> Append-only. Each entry is dated and specific.

---

### YYYY-MM-DD — <one-line summary>

**Tried:** <what I was attempting>

**Error:**
```
<exact error text — verbatim, including stack traces if present>
```

**Changed:** <what I modified>

**Source of fix:** AI / Docs / action.yml / experiment / other — be specific
about which AI tool, which doc page, etc.

---
```

Feeds the §11 "Known Issues and Resolutions" section of the final README.
Don't sanitize during build — the rough edges are the evidence the brief evaluates on.

---

## 9. Acceptance gates summary

| PR | Gate |
|---|---|
| 0 | `tree` matches; spec verbatim from brief; no secrets |
| 1 | Green run; 5+ Postman artifacts validated in UI; issues logged |
| 2 | Idempotency documented from observation, not guesswork |
| 3 | README §1-4 readable and accurate |
| 4 | README §5-9 grounded in observation; six customer asks listed |
| 5 | README §10-14 closed; AI disclosure has specific corrections |

A PR is not "done" until its gate is fully checked. No aspirational checks.

---

## 10. Anti-patterns (avoid these)

- **Squashing PR 1 iteration commits** before the user explicitly asks.
  Iteration history is debugging evidence.
- **Filling in observed-behavior sections from the plan's predictions.** Plan
  §4.4 has a predicted matrix. Populate from what you actually saw.
- **Generic AI disclosure.** "AI helped with YAML" doesn't pass. Specific:
  "AI referenced `@v1` (doesn't exist); corrected to `@v0` after reading the
  upstream release page."
- **Skipping the access token.** Without it, ~30% of the onboarding pipeline
  silently degrades. Log as an explicit risk regardless of whether you hit it.
- **Recommending tools the platform team would adopt.** Constraint 4.
- **Pushing to remote without showing the user the change first.**

---

## 11. When this BLUEPRINT is "done"

- [ ] PR 5 merged
- [ ] README is complete and evaluator-ready
- [ ] `issues-log.md` has every iteration's lesson captured
- [ ] AI disclosure passes the "specific not platitude" test
- [ ] Repo URL is shareable

After that, deck and Q&A work happens in the candidate's claude.ai workspace.
