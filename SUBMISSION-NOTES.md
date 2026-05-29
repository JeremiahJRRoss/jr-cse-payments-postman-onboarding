# Submission notes — payments onboarding

My own reading guide to this submission. Everything below points to a file in
this repo so you can verify it directly.

## What to look at first

This repo is the **canonical onboarding** — one GitHub Actions workflow
([`.github/workflows/onboard.yml`](.github/workflows/onboard.yml)) that drives
the open-alpha `postman-cs/postman-api-onboarding-action@v0` end to end:
workspace, spec, three collections, environments, mock, monitor. The companion
repo (`jr-cse-loan-origination-postman-onboarding`) is the **adaptation** —
the same shape on different compute/auth/env-count. The consulting story (ROI,
90-day roadmap, GitLab CI port, handoff) lives in the **deck**, out of this repo.

Fastest path through the evidence: [`docs/VALIDATION-EVIDENCE.md`](docs/VALIDATION-EVIDENCE.md)
(five screenshots + the rerun-sprawl shot), then `README.md §5` and `§10`.

## Core thesis: universal vs per-service

Onboarding a new service changes **8 input values** in `onboard.yml`. Everything
else — the action chain, the job structure, the permissions block, the generated
baseline/smoke/contract collections, and the environment scaffolding — is
identical. The 8 are: `project-name`, `domain`, `domain-code`, `spec-url`,
`spec-path`, `environments-json`, `env-runtime-urls-json`, `ci-workflow-path`.
This is the thing the whole submission is built to prove (see `README.md §5`,
and the side-by-side with Loan Origination there).

## Honest finding: idempotency & workspace sprawl

I verified re-run behavior by observation, not assumption. The action **reuses
the workspace** (git-sync `linked_match`) but **re-creates the spec, all three
collections, the mock, and the monitor with new IDs on every run**, and writes
no repo variables — so N no-change runs accumulate N×3 collections in one
workspace (measured: 5 runs → 15 collections). Evidenced in
[`README.md §10`](README.md), [`docs/VALIDATION-EVIDENCE.md §6`](docs/VALIDATION-EVIDENCE.md),
and the `2026-05-29` rerun entries in [`issues-log.md`](issues-log.md). This is
why I treat onboarding as a **create-once-per-service** operation, and why a
pre-run reuse-or-clean guard is the headline governance hardening as the catalog
scales (scoped in `README.md §12 #1`) — the forward-looking hook the deck builds on.

## AI usage

AI drafted the first cut of the workflow, the README prose, and the scaffolding
config; I validated the load-bearing parts by hand — reading the upstream
`action.yml`, running the workflow against a real workspace, and checking each
artifact in the UI. AI also got several things wrong (invented `variables: write`,
dropped `spec-url`, claimed idempotent re-runs) that I caught and corrected. Full
disclosure with specifics in [`README.md §14`](README.md); the debugging path is
in [`issues-log.md`](issues-log.md).

## Known trade-offs

- **Broken generated CI workflow.** repo-sync emits `payments-tests.yml` as
  invalid YAML and re-emits it every run; I quarantined it to
  [`evidence/generated-artifacts/`](evidence/generated-artifacts/) so it doesn't
  show a false red ✗. Durable fix (self-heal) scoped in `README.md §12 #2`.
- **CI can't pass on `example.com`.** Generated collections and the monitor
  target placeholder hosts by design; the customer-side fix is real URLs +
  `.postman/` metadata (`README.md §8`, `§12 #3`).
- **Spec version is `2.1.1`**, reflecting a deliberate spec-change / re-onboard
  test; spec, `service.config.yml`, and the SETUP check all agree.
- **Workspace not reset here.** Resetting the live Postman workspace to one clean
  run is a manual pre-demo step (out of scope for this branch).
