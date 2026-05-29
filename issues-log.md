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
