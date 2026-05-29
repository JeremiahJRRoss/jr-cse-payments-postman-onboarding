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
