# SETUP — jr-cse-payments-postman-onboarding

Wake-up checklist incorporating four real fixes discovered during the initial
build. Should take ~20 minutes total. Do steps in order.

> **This SETUP supersedes any earlier versions.** It was rewritten after four
> iterations of debugging produced the verified workflow contract.

---

## What you need before starting

| Requirement | Where it comes from |
|---|---|
| macOS with Homebrew | `brew --version` |
| `gh`, `git`, `jq` installed | `brew install gh git jq` |
| Postman CLI installed | `curl -o- "https://dl-cli.pstmn.io/install/osx_64.sh" \| sh` |
| `gh auth status` shows logged in | `gh auth login` |
| Postman account with a team workspace | (you already have this) |
| Your GitHub username — call it `OWNER` | `gh api user --jq .login` |

---

## Step 1 — Capture credentials and identity

These four values get used repeatedly. Set them as shell variables for the
session:

```bash
# 1. Generate a fresh PMAK in the Postman web UI:
#    Postman → Settings → API Keys → Generate API Key
#    Then export it here:
export POSTMAN_API_KEY="PMAK-<paste-your-key-here>"

# 2. Extract the session access token (REQUIRED — see warning below):
postman login                                                                    # browser flow
export POSTMAN_ACCESS_TOKEN="$(cat ~/.postman/postmanrc | jq -r '.login._profiles[].accessToken')"
echo "Access token starts with: ${POSTMAN_ACCESS_TOKEN:0:20}..."                 # confirm not empty

# 3. Your numeric Postman user ID:
export POSTMAN_USER_ID="$(curl -fsSL -H "X-API-Key: $POSTMAN_API_KEY" https://api.getpostman.com/me | jq '.user.id')"
echo "Postman user ID: $POSTMAN_USER_ID"

# 4. Your GitHub owner (username or org):
export OWNER="$(gh api user --jq .login)"
echo "GitHub owner: $OWNER"
```

> ⚠️ **The access token is the #1 failure mode.** It's session-scoped, expires
> silently, and is extracted manually because no public API mints it. Without
> it, the action's workspace git-sync, governance, and system-env steps
> silently skip — the run goes green but the workspace ends up half-built. If
> any of step 6's validation checks fail later, the access token is the most
> likely culprit: re-run step 1's #2 and re-set the secret in step 4.

---

## Step 2 — Create a fine-grained Personal Access Token (PAT)

This step exists because GitHub's default `GITHUB_TOKEN` cannot write to
`.github/workflows/` by design (supply-chain attack prevention). The
onboarding action's repo-sync phase needs to materialize a generated CI test
workflow, so it requires a separate token with the `workflows` scope. This is
exactly what the action's documented `gh-fallback-token` input is for.

### 2.1 Open the PAT creation page

```bash
open "https://github.com/settings/personal-access-tokens/new"
```

### 2.2 Fill in the form

| Field | Value |
|---|---|
| Token name | `postman-onboarding-workflow-write` |
| Description | `gh-fallback-token for postman-cs onboarding action — workflow file writes` |
| Resource owner | Your username (`$OWNER`) |
| Expiration | 30 days |
| Repository access | **Only select repositories** → add `jr-cse-payments-postman-onboarding` and `loan-origination-postman-onboarding` (the loan repo needs to exist first if it doesn't — create empty ones on github.com or come back to this token later) |

**Repository permissions** — set these four to **Read and write** (default is
"No access"):

| Permission | Access |
|---|---|
| Actions | Read and write |
| Contents | Read and write |
| Variables | Read and write |
| Workflows | Read and write |

Leave everything else at "No access." Do not touch Account permissions.

Click **Generate token**. Copy it immediately — it appears once. Starts with
`github_pat_`.

### 2.3 Verify the PAT BEFORE saving it as a secret

```bash
read -s PAT_NEW
# Paste the PAT, press Enter. Input is hidden.

# Both checks must pass:
curl -sS -H "Authorization: Bearer $PAT_NEW" https://api.github.com/user | jq '.login'
# Expected: "<YOUR_USERNAME>"

curl -sS -H "Authorization: Bearer $PAT_NEW" \
  https://api.github.com/repos/$OWNER/jr-cse-payments-postman-onboarding | jq '{name, permissions}'
# Expected: name + permissions object with push: true
# If it returns "Not Found", the repo isn't in the PAT's access list — edit the
# token, add it, retry.
# If the first command returned "Bad credentials", the paste was incomplete —
# regenerate and retry.
```

> ⚠️ **Don't skip this verification.** A PAT that fails silently is the
> second-most-common reason this build hangs. Two minutes here saves a 60-second
> workflow run that fails for opaque reasons.

---

## Step 3 — Upload and create the GitHub repo

### 3.1 Unzip the deliverable

```bash
cd ~/Desktop
# If a previous attempt exists, delete it first:
rm -rf jr-cse-payments-postman-onboarding
unzip jr-cse-payments-postman-onboarding.zip
cd jr-cse-payments-postman-onboarding
ls -la                                          # confirm dotfiles present (.github/, .claude/, .gitignore)
```

### 3.2 Initialize git and create the repo

```bash
git init -b main
git add -A
git commit -m "Initial scaffold: verified onboarding workflow + spec + build docs"
gh repo create jr-cse-payments-postman-onboarding --public --source=. --push
```

**If `gh repo create` fails** with "Unable to add remote 'origin'" or "Name
already exists":

```bash
# The repo or local remote already exists from a previous attempt.
# Set the remote URL explicitly:
git remote set-url origin https://github.com/$OWNER/jr-cse-payments-postman-onboarding.git 2>/dev/null \
  || git remote add origin https://github.com/$OWNER/jr-cse-payments-postman-onboarding.git

# If the GitHub repo doesn't exist yet:
gh repo create jr-cse-payments-postman-onboarding --public 2>/dev/null

git push -u origin main
```

### 3.3 Verify everything pushed

```bash
gh api repos/$OWNER/jr-cse-payments-postman-onboarding/contents/.github/workflows | jq '.[].name'
# Expected: ["onboard.yml"]

gh api repos/$OWNER/jr-cse-payments-postman-onboarding/contents/.claude/commands | jq '.[].name'
# Expected: 3 files (log-issue.md, run-and-watch.md, validate-postman.md)

gh api repos/$OWNER/jr-cse-payments-postman-onboarding/contents/specs | jq '.[].name'
# Expected: ["payment-refund-api-openapi.yaml"]
```

---

## Step 4 — Configure secrets and variables

```bash
# Three secrets — sensitive
echo "$POSTMAN_API_KEY"      | gh secret set POSTMAN_API_KEY      --repo $OWNER/jr-cse-payments-postman-onboarding
echo "$POSTMAN_ACCESS_TOKEN" | gh secret set POSTMAN_ACCESS_TOKEN --repo $OWNER/jr-cse-payments-postman-onboarding
echo "$PAT_NEW"              | gh secret set GH_FALLBACK_TOKEN    --repo $OWNER/jr-cse-payments-postman-onboarding

# Clean up the PAT from your shell environment:
unset PAT_NEW

# Two variables — identity values, not sensitive
gh variable set REQUESTER_EMAIL --repo $OWNER/jr-cse-payments-postman-onboarding --body "your.email@example.com"
gh variable set POSTMAN_USER_ID --repo $OWNER/jr-cse-payments-postman-onboarding --body "$POSTMAN_USER_ID"

# Verify all five are present
gh secret   list --repo $OWNER/jr-cse-payments-postman-onboarding
gh variable list --repo $OWNER/jr-cse-payments-postman-onboarding
```

Expected:
- Secrets (3): `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN`, `GH_FALLBACK_TOKEN`
- Variables (2): `REQUESTER_EMAIL`, `POSTMAN_USER_ID`

> 💡 **Replace `your.email@example.com`** with your actual email before running.
> The action uses this for workspace audit metadata.

---

## Step 5 — Run the onboarding workflow

```bash
gh workflow run onboard.yml --repo $OWNER/jr-cse-payments-postman-onboarding
sleep 5
RUN_ID=$(gh run list --repo $OWNER/jr-cse-payments-postman-onboarding --workflow=onboard.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN_ID" --repo $OWNER/jr-cse-payments-postman-onboarding
```

Expected runtime: 40-90 seconds. Final status should be `success`.

**Three benign annotations** you'll see and should ignore:

1. *"Node.js 20 actions are deprecated..."* — Upstream action's issue; harmless until June 2026.
2. *"A schema property should have a `$ref` property..."* (×9) — Best-practice
   linting on the spec from the brief; doesn't block onboarding.
3. *"Failed to invite requester: Only one role is supported for user ..."* —
   You're already on the workspace; the action tried to add you a second time.
   Benign.

**If the run fails** — see the `Troubleshooting` section at the end of this file.

---

## Step 6 — Sync local working tree

The action commits the generated CI workflow and the collection JSON exports
back to `main`. Pull them down:

```bash
cd ~/Desktop/jr-cse-payments-postman-onboarding
git pull --rebase
git log --oneline -5
ls postman/collections/
ls .github/workflows/
```

Expected:
- `git log` shows a commit by `Postman CSE <help@postman.com>` titled "chore: sync Postman artifacts and metadata"
- `postman/collections/` contains 3 JSON files (Baseline, Contract, Smoke)
- `.github/workflows/` contains both `onboard.yml` AND `payments-tests.yml`

---

## Step 7 — Validate in the Postman UI

**This is the most important step.** A green workflow is necessary but NOT
sufficient — the access token failure mode produces green runs with broken
workspaces.

Open Postman:

```bash
open "https://go.postman.co/workspaces"
```

Find the workspace whose name begins with `[PMT]` — likely `[PMT] payment-refund-service`.

Confirm **all six** items:

| # | Check | What you should see |
|---|---|---|
| 1 | Workspace exists | Sidebar entry, name starts with `[PMT]` |
| 2 | Spec in Spec Hub | APIs section → "Payment Processing API - Refund Service" v2.1.0 renders |
| 3 | Three collections | `[Baseline]`, `[Contract]`, `[Smoke]` payment-refund-service — each opens to show `refunds` and `health` folders with populated requests |
| 4 | Four environments | Environments section: `prod`, `uat`, `qa`, `dev` |
| 5 | Mock server | Mock Servers section shows one mock |
| 6 | Monitor | Monitors section shows one monitor (scheduled or "Ready to run") |

> 💡 **One run = three collections; re-runs add more.** Observed: this
> open-alpha action reuses the **workspace** (git-sync `linked_match`) but
> re-creates the spec, collections, mock, and monitor with new IDs each run, so
> they **accumulate** — it is not idempotent inside the workspace (see README
> §10). 6/9/12 collections means the workflow ran multiple times, not that a
> check "failed." `gh variable list` holds only the two input variables you set;
> no `POSTMAN_*` resource-ID variables are written back:
> ```bash
> gh variable list --repo $OWNER/jr-cse-payments-postman-onboarding
> ```
> Prune duplicates from the UI, keeping the most recent set (one Baseline, one
> Contract, one Smoke).

If any of the six checks fail, the workspace is half-built — see
`Troubleshooting → Workspace empty or partial`.

---

## Step 8 — Capture validation evidence

These artifacts feed into the README's "Validation Evidence" section and are
your fallback during the live demo if Postman or the network misbehaves.

```bash
cd ~/Desktop/jr-cse-payments-postman-onboarding
mkdir -p docs/screenshots
```

Take five screenshots with `Cmd+Shift+4`, space, then click the Postman window:

1. `workspace-home.png` — sidebar showing all artifacts
2. `spec-hub.png` — the rendered spec
3. `baseline-collection.png` — the Baseline collection expanded with requests visible
4. `environments.png` — environments list with variables populated
5. `monitor.png` — the monitor's status page

Move them into the screenshots directory:

```bash
mv ~/Desktop/Screenshot*.png docs/screenshots/ 2>/dev/null
ls docs/screenshots/
```

Capture the run URL too — write it down or paste into a temporary file:

```
https://github.com/$OWNER/jr-cse-payments-postman-onboarding/actions/runs/$RUN_ID
```

You'll reference this URL in README §9.

Commit the screenshots:

```bash
git add docs/screenshots/
git commit -m "docs: validation evidence screenshots"
git push
```

---

## Step 9 — Observe rerun behavior (recommended)

Trigger one more run with no changes and record what actually happens — don't
assume idempotency.

```bash
gh workflow run onboard.yml --repo $OWNER/jr-cse-payments-postman-onboarding
sleep 5
RUN_ID=$(gh run list --repo $OWNER/jr-cse-payments-postman-onboarding --workflow=onboard.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN_ID" --repo $OWNER/jr-cse-payments-postman-onboarding
```

After it goes green, compare the two runs' outputs and the workspace
before/after. **Observed on this repo:** the **workspace ID is stable** (reused
via the git-sync `linked_match`) while the **child IDs change** — the committed
`collection.yaml` `id`s differ each run — and the workspace **accumulates**
duplicate collections and environments. The action persists/reuses no resource
IDs (`gh variable list` shows only the two input variables, never updated by a
run). Confirm in the Postman UI whether the collection count grew. See README
§10 for the full write-up.

This observation is your answer to Q&A question 5 ("What happens on re-run?
Do you get duplicate workspaces?"). Record exactly what you see in README §10.

---

## Troubleshooting

### Workflow fails — "Unexpected value 'variables'"

GitHub Actions rejects `variables: write` as a permission key. The workflow
file should only have `actions: write` and `contents: write`. This SETUP's
workflow is correct; if you see this, you may be running a stale local copy:

```bash
grep -A4 "permissions:" .github/workflows/onboard.yml
# Should only show actions:write and contents:write
```

### Workflow fails — "Input required and not supplied: spec-url"

Despite the action's `action.yml` declaring `spec-url` as optional, the
underlying `postman-bootstrap-action@v0` requires it at runtime. The shipped
workflow includes both `spec-url` and `spec-path`. Verify both are present:

```bash
grep -E "spec-(url|path):" .github/workflows/onboard.yml
# Should show 2 lines
```

### Workflow fails — "remote rejected... without `workflows` permission"

Default `GITHUB_TOKEN` can't write to `.github/workflows/`. The
`gh-fallback-token` input wires a PAT for this. Confirm:

```bash
grep "gh-fallback-token" .github/workflows/onboard.yml
# Should print: gh-fallback-token: ${{ secrets.GH_FALLBACK_TOKEN }}

gh secret list --repo $OWNER/jr-cse-payments-postman-onboarding | grep GH_FALLBACK_TOKEN
# Should show GH_FALLBACK_TOKEN exists
```

If the secret exists but the run still fails, your PAT may have expired or
lost repo access — return to Step 2 and regenerate.

### Workflow fails — "Permission... denied" / 403

The PAT exists but isn't authorized for this repo. Two common causes:

1. **PAT's repo-access list doesn't include this repo.** Open the PAT in
   GitHub settings, edit, add the repo to the access list.
2. **PAT scopes missing.** Confirm Contents+Workflows+Actions+Variables are
   all set to "Read and write."
3. **PAT mistyped or expired.** Regenerate per Step 2.

Verify against the API:

```bash
curl -sS -H "Authorization: Bearer <PAT>" https://api.github.com/repos/$OWNER/jr-cse-payments-postman-onboarding | jq '{name, permissions}'
# permissions.push must be true
```

### Workflow goes green, but workspace is empty or partial in Postman UI

The `POSTMAN_ACCESS_TOKEN` is missing or expired. Without it, workspace
git-sync, governance, and system-env steps silently skip.

```bash
postman login                                                                         # refresh session
export POSTMAN_ACCESS_TOKEN="$(cat ~/.postman/postmanrc | jq -r '.login._profiles[].accessToken')"
echo "$POSTMAN_ACCESS_TOKEN" | gh secret set POSTMAN_ACCESS_TOKEN --repo $OWNER/jr-cse-payments-postman-onboarding
gh workflow run onboard.yml --repo $OWNER/jr-cse-payments-postman-onboarding
```

### Workflow fails — Postman team or org error

If you see "Team feature is not available for your organization" or a
team-related 400, your Postman account isn't on a team plan with sub-teams.
The action handles this gracefully — the message is a warning, not an error.
If the run actually fails because of an org-mode mismatch, see the commented
`org-mode` and `workspace-team-id` lines in `onboard.yml`.

### Multiple sets of collections in the workspace

This open-alpha action reuses the **workspace** but re-creates its contents:
each run provisions a fresh set of collections with new IDs and does **not**
persist/reuse resource IDs as repo variables (see README §10). Duplicates are
therefore expected after any re-run — they **accumulate** — not a sign of a
failed step. Clean up manually in the Postman UI: right-click each older
duplicate → Delete, keeping the most recent Baseline, Contract, and Smoke.

Until the action persists IDs upstream, avoid unnecessary re-runs (onboard once
per service); a re-run will duplicate again.

### "Bad credentials" when verifying the PAT

Most likely a paste truncation. Regenerate (Step 2), and on the new token
triple-click the value field to select all of it before copying.

---

## What "done" looks like

- [ ] All steps 1-7 completed
- [ ] Six Postman UI checks passed (Step 7)
- [ ] Five validation screenshots committed to `docs/screenshots/`
- [ ] Workflow run URL captured for README §9
- [ ] (Optional) Rerun behavior observed and recorded in README §10 (note: re-runs duplicate — not idempotent)

Once all boxes are checked, the Payments repo is submission-ready. Move on to
the loan-origination repo, where you'll apply the same verified pattern
with smaller per-service deltas.
