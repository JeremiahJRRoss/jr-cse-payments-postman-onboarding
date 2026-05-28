# Verified Action Contract Notes

Captured by inspecting `postman-cs/postman-api-onboarding-action/action.yml` and
README directly. Where these notes conflict with BLUEPRINT.md / PROMPTS.md (written
earlier), **these notes win** — they're from the live action.

## Version pin
- Only `@v0` exists (rolling open-alpha). No immutable `v0.x.y` tags published yet.
- Earlier docs referencing `@v1` were wrong. Use `@v0`.

## Spec input: spec-path vs spec-url
- The action accepts EITHER `spec-url` (fetchable HTTPS) OR `spec-path` (repo-root
  relative, read from the checked-out workspace).
- This scaffold uses **`spec-path`** — more robust, no dependency on the raw URL
  being reachable/public before the run.

## Permissions
- Needs `actions: write`, `contents: write`, AND `variables: write`.
  The `variables: write` is what lets the action persist workspace/spec/collection
  IDs as repo variables for idempotent re-runs.

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
- `starter-assets/actions/onboard-service/action.yml` line ~35 passes
  `spec-url: ${{ inputs.spec-path }}` — feeds a local path into the URL input.
  If you pass only a path, it lands in spec-url (expects HTTPS) and fails.
  We call the upstream action directly and use spec-path correctly.

## checkout version
- Official examples use `actions/checkout@v5`. This scaffold uses v5.
