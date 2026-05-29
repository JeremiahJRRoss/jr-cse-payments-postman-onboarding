# Issues Log — jr-cse-payments-postman-onboarding

> Honest documentation of issues encountered during build.
> Append-only. Each entry is dated and specific. This feeds the README's
> "Known Issues and Resolutions" and "AI Assistance" sections — both graded.

---

<!-- Add entries below as you build. Template:

### YYYY-MM-DD — <one-line summary>
**Tried:** <what I was attempting>
**Error:**
```
<exact error text — verbatim>
```
**Changed:** <what I modified>
**Source of fix:** AI / Docs / action.yml / experiment / other

-->
### 2026-05-28 — `variables: write` is not a valid GitHub Actions permission key
**Tried:** Added `variables: write` to the job `permissions:` block (AI's initial draft of onboard.yml) so the action could persist workspace/spec/collection IDs as repo variables.
**Error:**
```
Unexpected value 'variables'   # HTTP 422 on `gh workflow run`, at workflow parse
```
**Changed:** Removed the line. Repo variables are written by the action via the GitHub API using `github-token` / `gh-fallback-token`, not a job permission key.
**Source of fix:** experiment (first run failed at parse) + action.yml. AI had added the invalid key; corrected against the GitHub Actions permission allow-list.

### 2026-05-28 — `spec-url` required at runtime despite being optional in action.yml
**Tried:** Providing only `spec-path` (action.yml marks both optional — "provide either `spec-url` or `spec-path`").
**Error:**
```
Input required and not supplied: spec-url
```
**Changed:** Added `spec-url` alongside `spec-path` (safe superset). The downstream `postman-bootstrap-action@v0` requires `spec-url` at runtime.
**Source of fix:** experiment (second run). Discrepancy vs action.yml's optional declaration; noted in README §11.A #2 and VERIFIED-NOTES.

### 2026-05-28 — default GITHUB_TOKEN cannot write to .github/workflows/
**Tried:** Letting repo-sync commit the generated CI workflow with the default `GITHUB_TOKEN` (AI's initial draft omitted `gh-fallback-token`).
**Error:**
```
refusing to allow ... without `workflows` permission   # HTTP 422 on the commit-back step (third run)
```
**Changed:** Created a fine-grained PAT (Contents/Workflows/Actions/Variables: write) and wired it via `gh-fallback-token`.
**Source of fix:** experiment (third run) + action README. The default token cannot write workflow files by design (supply-chain protection); `gh-fallback-token` is the action's documented escape hatch.

### 2026-05-28 — fine-grained PAT fails silently if mistyped/truncated
**Tried:** Setting the PAT secret directly after creating it, without independently checking it.
**Error:**
```
403  (no clear cause; run failed opaquely)
```
**Changed:** Verify every PAT against `GET https://api.github.com/user` and the repo's `permissions.push` field before running `gh secret set`. This caught one silently-truncated paste during setup.
**Source of fix:** experiment. Build lesson (README §11.A #4) — not an AI error, so not cited in §14.

### 2026-05-28 — Repo rename broke PAT scoping
**Tried:** First workflow run after renaming the repo from
payments-postman-onboarding to jr-cse-payments-postman-onboarding.
**Error:** ```
remote: Permission to JeremiahJRRoss/jr-cse-payments-postman-onboarding.git
denied to JeremiahJRRoss.
fatal: unable to access ... error: 403
```
**Changed:** Edited the PAT in GitHub settings — added the new repo name (`jr-cse-payments-postman-onboarding`) to its repository access list. Token value unchanged.
**Source of fix:** experiment + GitHub PAT settings. Fine-grained PATs are name-scoped, so the token's repository access list still pointed at the old repo name after the rename; re-adding the new name restored authorization. See README §11.A #5 / §14.

### 2026-05-29 — repo-sync emitted generated CI workflow as one escaped line
**Tried:** Inspecting the generated `.github/workflows/payments-tests.yml` after onboarding.
**Error:**
```
File committed as a single physical line with 90 literal \n; invalid YAML
(ScannerError ... line 1, column 49); GitHub flags "Invalid workflow file".
```
**Changed:** Decoded `\n` → real newlines in place; verified lossless against the committed original (re-encode == original, 4370→4280 bytes); confirmed valid YAML with `on` + `jobs.test`.
**Source of fix:** experiment + inspection. Root cause is upstream repo-sync (open-alpha) serializing the file as a JSON-escaped string; byte-identical defect appears in the companion repo, confirming it's upstream, not a local slip.

### 2026-05-29 — decision: generated CI workflow won't pass on example.com (layer 2)
**Tried:** Making the now-valid `payments-tests.yml` (PR R1) actually pass in CI.
**Error:**
```
Resolve Postman Resource IDs step: "Missing Postman resource IDs in
.postman/resources.yaml" — .postman/ is gitignored, so the file is absent on a
clean CI checkout; collections also target example.com (placeholder hosts).
```
**Changed:** No code change. Chose Option C (blueprint §3): document the limitation rather than commit `.postman/` metadata. Recorded in README §8 with the one-line customer fix (un-ignore `.postman/` + supply real URLs/`POSTMAN_API_KEY`).
**Source of fix:** decision. Consistent with the monitor's by-design red status on example.com; A/B (commit `.postman/resources.yaml`) is the change a real customer makes for a green badge. Refs: submission review item 1, layer 2.

### 2026-05-29 — onboarding re-run is NOT idempotent: new collection IDs + workspace duplicates
**Tried:** Validating README §10's idempotency claim by observation (not assumption) — a second no-change `workflow_dispatch` run of `onboard.yml` on `main` (run 26652077213, green in 53s).
**Error:**
```
Re-run minted new collection IDs instead of reusing them: sync commit 6fa0606
rewrote all three collection.yaml `id`s (Smoke 7d304463-… → 88c5f9ff-…), and the
Postman workspace gained duplicate collections. `gh variable list` shows only the
two input vars (POSTMAN_USER_ID, REQUESTER_EMAIL); no POSTMAN_* resource-ID
variables are persisted. Run log shows no variable-write attempt and no 403.
```
**Changed:** No code change. The candidate fix (github-token → GH_FALLBACK_TOKEN) was investigated and ruled out — the action logs no variable-write attempt, so this isn't a token-permission failure. Instead documented the observed non-idempotent behavior across every surface that carried the old claim: README §10 + §13, SETUP.md (Step 7 tip, Step 9, troubleshooting, checklist), and VERIFIED-NOTES.md.
**Source of fix:** experiment (observed re-run) + inspection (commit diff, `gh variable list`, run-log grep). Honest disclosure: re-runs duplicate; a durable fix is upstream (action must persist/reuse resource IDs) or a custom output-capture layer — out of scope here. Refs: submission review item 4/5 (rerun behavior).

### 2026-05-29 — re-run regenerated payments-tests.yml as the escaped single line again (item 1 regression)
**Tried:** Same idempotency re-run above (run 26652077213). Checked the regenerated CI workflow afterward.
**Error:**
```
Sync commit 6fa0606 rewrote .github/workflows/payments-tests.yml back to one
escaped physical line (0 newlines, 90 literal \n, invalid YAML) — exactly the
item-1 defect, regenerated on re-run, overwriting the committed decode on main.
```
**Changed:** Re-applied the item-1 decode (lossless, verified vs 6fa0606: 4370→4280 bytes; parses with on + jobs.test) and committed it back to main.
**Source of fix:** experiment + inspection. This is direct confirmation of the item-1 §12 trade-off: repo-sync re-escapes the file every run, so the durable fix is a post-generation decode step inside `onboard.yml` (self-heal) plus reporting the escaping upstream. Refs: submission review item 1 (recurrence), item 4/5.

### 2026-05-29 — rerun behavior locked: workspace reused (linked_match), children accumulate, no repo variables; empty bootstrap/repo-sync outcomes
**Tried:** Pinning down rerun/idempotency behavior across several no-change `onboard.yml` runs (the "persists IDs as repo variables → idempotent" claim had never been verified), then reconciling the result in the Postman UI.
**Error:**
```
- Workspace REUSED across runs — run log: "Using canonical workspace (linked_match)";
  matched via the repo<->workspace git-sync link, not via any stored ID.
- Spec, all three collections, mock, and monitor RE-CREATED with new IDs every run
  ("Created new mock", "Created new monitor"; the sync commit rewrites each collection.yaml id).
- ZERO GitHub repo variables written — `gh variable list` shows only the two input vars
  (POSTMAN_USER_ID, REQUESTER_EMAIL); no POSTMAN_* resource-ID vars, no write attempt, no 403.
- UI-confirmed ACCUMULATION: after 5 runs the reused workspace held 15 collections
  (5x Baseline / Contract / Smoke) + multiple duplicate "payment-refund-service - dev" environments.
- Secondary (§0.3): the Print step's `bootstrap-outcome` / `repo-sync-outcome` outputs came back
  EMPTY both runs — the orchestrator doesn't populate those output names, though the run still
  produced every artifact. /run-and-watch treats empty outcomes as a red flag; here it isn't one.
```
**Changed:** No code change (`onboard.yml` untouched). Documentation correction only: rewrote README §10 to the observed behavior (workspace reused via the git-sync link; spec/collections/mock/monitor re-created and accumulating), corrected §13, fixed §12's cleanup bullet, added the §14 AI-disclosure entry, aligned SETUP §7/§9/troubleshooting and VERIFIED-NOTES, and added the 15-collections accumulation evidence to docs/VALIDATION-EVIDENCE.md.
**Source of fix:** experiment (multiple re-runs) + inspection (run log "linked_match", `gh variable list`, sync-commit `id` diffs) + UI confirmation (collection count). Honest disclosure: re-runs accumulate; the durable fix is a pre-run reuse-or-clean guard or upstream ID persistence — out of scope here. Refs: submission review item 5 (feeds items 2, 3, plan Q5).

### 2026-05-29 — item 6: brief asks for "JSON exports" but committed collections are git-sync YAML
**Tried:** Reconciling the submission requirement ("Generated Postman collections (JSON exports)") against what the repo actually commits under `postman/collections/`.
**Error:**
```
README §9 / repo-structure tree and SETUP Step 6 called postman/collections/
"3 JSON files / JSON exports", but the committed artifacts are git-sync YAML
directory trees (collection.yaml + *.request.yaml), not .json. Format mismatch
an evaluator's checklist would catch.
```
**Changed:** Option C (BLUEPRINT-json-vs-yaml §2). Kept the YAML canonical; added `postman/exports/{baseline,contract,smoke}.json` — single-file v2.1 exports of the same three collections — and corrected README/SETUP wording so nothing labels the YAML "JSON".

JSON export results (all valid per `jq -e '.info.name'`; v2.1.0 schema):

| File | UID exported | bytes | requests | `_postman_id` == committed YAML `id` |
|---|---|---|---|---|
| `baseline.json` | `38960911-8ed6bcc0-3da6-442d-8623-f1e650e22187` | 90,658 | 6 | MATCH |
| `contract.json` | `38960911-f860f9b5-65db-4b8a-a59e-34f3de0cc1fd` | 116,801 | 7 | MATCH |
| `smoke.json` | `38960911-cf28226b-bac2-4da3-b062-cf059f669caa` | 100,454 | 7 | MATCH |

**Deviation from blueprint §2 Step 1 (documented):** did NOT delete the workspace + do a fresh `onboard.yml` run. This environment has no `gh` CLI and the GitHub MCP surface exposes no workflow-dispatch, so a clean run was not runnable here. It was also unnecessary: the committed `collection.yaml` files already carry their `id`s, and those exact collections were still live in the workspace (the `2026-05-29T18:25Z` set). Exporting those UIDs gives provable coherence — verified `_postman_id == committed YAML id` for all three — which is the coherence rule's intent. Trade-off: the workspace still holds its ~15 accumulated duplicates (a demo/item-1 concern, not item 6). Acceptance gate (§3) passed: exports valid, no doc calls the YAML "JSON" outside `postman/exports/`, SETUP paths match disk, `postman/collections/` + protected files untouched.
**Source of fix:** experiment (Postman API `GET /collections/{uid}` per committed `id`; `jq` coherence check) + BLUEPRINT-json-vs-yaml Option C. Refs: submission review item 6.
