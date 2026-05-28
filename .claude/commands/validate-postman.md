Walk me through validating Postman artifacts in the UI.

From the latest green run's "Print onboarding outputs" step, extract:
- workspace-url / workspace-id
- spec-id
- collections (baseline, smoke, contract)
- mock-url
- monitor-id

Present as a numbered list with the URL/value for each. After each item, wait
for me to type "ok" or paste an issue. If issue: invoke the /log-issue flow.

Checklist to confirm in the UI:
1. Workspace exists with expected naming
2. Spec is in Spec Hub and parses
3. Three collections exist and have populated requests
4. Environments populated (one per env in environments-json)
5. Mock server reachable
6. Monitor created

End with: "Ready to re-run for idempotency? (Y/n)"
