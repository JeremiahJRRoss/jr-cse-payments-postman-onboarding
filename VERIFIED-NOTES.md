# Verified Action Contract Notes

Facts checked against the live `action.yml` on 2026-05-29. **The committed
`onboard.yml` is the source of truth for the contract; this file annotates why
each input is set.** Where this file and `onboard.yml` disagree, `onboard.yml`
wins and this file is wrong — fix it.

## Version pin
- Only `@v0` exists (rolling open-alpha). No immutable `v0.x.y` tags published yet.
- Earlier docs referencing `@v1` were wrong. Use `@v0`.

## Spec input: spec-url vs spec-path
- action.yml accepts **either**: `spec-url` (HTTPS) **or** `spec-path` (repo-root
  relative; used as the bootstrap source "when spec-url is not provided").
- This scaffold's `onboard.yml` provides **both**. Rationale in README §11.A #2 /
  §14: under the open-alpha build the run was observed to require `spec-url` at
  runtime even though action.yml marks it optional, so both are supplied as the
  safe superset. (action.yml documents spec-path-alone as sufficient; if you
  re-confirm that, you may drop spec-url — but the committed contract is both.)

## Permissions
- The job needs `actions: write` and `contents: write` **only**.
- GitHub Actions has **no `variables` permission key** (a `variables: write`
  line is rejected: "Unexpected value 'variables'"). Repo variables are written
  by the action via the GitHub API using `github-token` / `gh-fallback-token` —
  action.yml: "GitHub token used for repo variables and generated commits." See
  onboard.yml's inline note and README §11.A #1.
- **Observed caveat (resource-ID persistence):** the action.yml wording above is
  the *declared* contract. In practice the action wrote **no** `POSTMAN_*`
  workspace/spec/collection-ID variables on this repo (`gh variable list` shows
  only the two input variables), and the run log logs no variable-write attempt —
  so re-runs reuse the workspace (via the git-sync `linked_match`) but re-create
  and **accumulate** its contents (spec, collections, mock, monitor) rather than
  reusing them. Idempotency is **not** observed here; see README §10.

## ci-workflow-path
- Default is `.github/workflows/ci.yml` (NOT `onboard.yml`). The earlier
  "it overwrites itself" claim was wrong. Still set it explicitly per service.

## postman-access-token (critical)
- Session-scoped, manually extracted, expires silently. Without it, workspace
  git-sync (Bifrost), governance assignment, and system-env association are
  SILENTLY SKIPPED. Run looks green; workspace is half-built. #1 failure mode.

## org-mode / workspace-team-id
- If your PMAK's team is org-scoped, the Postman API requires a team ID. Symptom:
  team/org error on first run. Fix: `org-mode: true` + `workspace-team-id`
  (both commented in onboard.yml; set POSTMAN_TEAM_ID variable).

## mTLS (loan-origination relevance)
- The action HAS `ssl-client-cert`, `ssl-client-key`, `ssl-client-passphrase`,
  `ssl-extra-ca-certs` inputs (base64 PEM), passed through to repo-sync for CI
  workflow generation.
- The loan spec declares only JWT in `securitySchemes` (mTLS is in description
  prose), so the generated COLLECTION wires JWT. mTLS is wired via these inputs
  from customer-provided secrets — correct behavior, not a gap.

## starter-assets bug (why we bypass it — decision D2)
- Per the starter README / D2 rationale (not re-verified here — the
  starter-assets path returned HTTP 404 on 2026-05-29): the starter's
  `onboard-service/action.yml` wraps the upstream action and was reported to
  pass `spec-url: ${{ inputs.spec-path }}`, feeding a local path into the URL
  input, which expects HTTPS.
- We bypass the starter and call `postman-cs/postman-api-onboarding-action@v0`
  directly, supplying **both** `spec-url` and `spec-path` (see Spec input above).

## checkout version
- Official examples use `actions/checkout@v5`. This scaffold uses v5.
