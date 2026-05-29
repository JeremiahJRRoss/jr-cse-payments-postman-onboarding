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
### 2026-05-28 — Repo rename broke PAT scoping
**Tried:** First workflow run after renaming the repo from
payments-postman-onboarding to jr-cse-payments-postman-onboarding.
**Error:** ```
remote: Permission to JeremiahJRRoss/jr-cse-payments-postman-onboarding.git
denied to JeremiahJRRoss.
fatal: unable to access ... error: 403
```

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
