# Generated artifacts (preserved as evidence)

This directory holds tool-generated output kept verbatim as evidence, **not**
hand-authored code. Nothing here is meant to run.

## `payments-tests.yml.txt`

The CI test workflow emitted by `postman-cs/postman-repo-sync-action@v0` (the
repo-sync phase of the onboarding action), preserved exactly as the tool
committed it — byte-for-byte, renamed to `.txt` only so GitHub does not try to
parse it.

**Why it's parked here instead of under `.github/workflows/`:** repo-sync
serializes this file as a single physical line with ~90 literal `\n` sequences
instead of real newlines, which is invalid YAML (`ScannerError ... line 1,
column 49`). GitHub flags that as an **"Invalid workflow file"** — a red ✗ in
the Actions tab next to the green onboarding run. Moving it out of
`.github/workflows/` removes the false failure while keeping the artifact as
evidence of exactly what the open-alpha tool produced.

**It is an upstream open-alpha bug, not a local slip.** The same byte-identical
defect appears in the companion repo, confirming it originates in repo-sync's
JSON-escaped string serialization.

**It regenerates on every run.** Re-running `onboard.yml` re-emits this file,
escaped, at `.github/workflows/payments-tests.yml` — so this quarantine is a
point-in-time cleanup of the committed artifact, not a permanent fix. The
durable fix is a post-generation decode/self-heal step in `onboard.yml` (scoped
in `README.md §12 #2`) or an upstream repo-sync fix. After any future
onboarding run, the regenerated file must be re-decoded or re-quarantined
before review/demo.

**A second, separate reason it would not pass even once decoded:** its "Resolve
Postman Resource IDs" step reads `.postman/resources.yaml`, which is gitignored
and therefore absent on a clean CI checkout, and its collections target
`example.com` placeholder hosts. See `README.md §8` and `§12 #3`.

**Cross-references**

- `issues-log.md` — `2026-05-29 — repo-sync emitted generated CI workflow as one
  escaped line` (original discovery) and `2026-05-29 — re-run regenerated
  payments-tests.yml as the escaped single line again` (confirmed recurrence).
- `README.md §8` — note on the generated CI workflow.
- `README.md §11.A #6` — the iteration record.
- `README.md §12 #2` — the self-healing-generation productionizing move.
