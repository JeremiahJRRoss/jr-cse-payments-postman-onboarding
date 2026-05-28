Trigger the onboarding workflow and watch it to completion.

Steps:
1. `gh workflow run onboard.yml`
2. Wait 5 seconds, then get the run ID:
   `gh run list --workflow=onboard.yml --limit 1 --json databaseId --jq '.[0].databaseId'`
3. `gh run watch <run-id>` until it finishes.
4. If failed: `gh run view <run-id> --log-failed`. Summarize the failure and
   propose the smallest possible fix. Do NOT apply without my confirmation.
5. If succeeded: read the "Print onboarding outputs" step log and extract
   workspace-url, workspace-id, spec-id, mock-url, monitor-id, collections.
   Remind me to validate in the Postman UI.

CRITICAL: a green run is necessary but NOT sufficient. If bootstrap-outcome or
repo-sync-outcome is anything other than "success", or if outputs are empty,
the POSTMAN_ACCESS_TOKEN may be missing/expired and steps silently skipped.
Flag this explicitly.
