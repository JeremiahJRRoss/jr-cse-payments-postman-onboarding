# Payments Repo — Sequential Claude Code Prompts

> **Companion to:** `BLUEPRINT.md` (the spec). This file is the turn-by-turn
> operational guide. Each turn is one paste into `claude`.
>
> **How to use:**
> 1. Complete pre-flight setup below.
> 2. Open the repo in Claude Code (`claude` from the repo root).
> 3. Work through turns in order. Each turn has: paste-able prompt, what to
>    expect back, and what to decide before the next turn.
> 4. If something goes sideways, see §Recovery prompts at the end.
>
> **Voice rule:** these prompts mirror the voice of `CLAUDE.md` and `BLUEPRINT.md`
> — direct, specific, expects the agent to stop and wait at decision points.
> Don't soften them.

---

## Pre-flight (do this before opening Claude Code)

| # | Step | Verify |
|---|---|---|
| P1 | Postman Enterprise trial active | Can create a workspace in the UI |
| P2 | PMAK API key generated | Value in hand (`PMAK-...`) |
| P3 | Postman CLI `postman login` complete | Browser auth succeeded |
| P4 | Extract access token:<br>`cat ~/.postman/postmanrc \| jq -r '.login._profiles[].accessToken'` | Token in hand |
| P5 | `gh` CLI authenticated | `gh auth status` clean |
| P6 | Repo `jr-cse-payments-postman-onboarding` created on GitHub, cloned locally | `git remote -v` correct |
| P7 | Both secrets set on the repo:<br>`gh secret set POSTMAN_API_KEY`<br>`gh secret set POSTMAN_ACCESS_TOKEN` | `gh secret list` shows both |
| P8 | Postman user ID:<br>`curl -H "X-API-Key: $POSTMAN_API_KEY" https://api.getpostman.com/me \| jq '.user.id'` | Numeric ID in hand |
| P9 | `CLAUDE.md` and `BLUEPRINT.md` committed to repo | Files visible on GitHub |
| P10 | Capture these for substitution into prompts:<br>• YOUR_EMAIL<br>• YOUR_POSTMAN_USER_ID<br>• OWNER (your GitHub username or org) | All three written down |

If P1-P10 aren't all complete, fix that first. Don't paper over.

---

## PR 0 — Scaffolding

Goal: lay down every static file. No workflow runs yet.

### Turn 0.1 — Create scaffolding and show tree

Paste:

```
Turn 0.1 — Scaffold the repo per BLUEPRINT.md §4 PR 0.

Steps:
1. Fetch the spec from the brief:
   mkdir -p specs
   curl -fsSL -o specs/payment-refund-api-openapi.yaml \
     https://raw.githubusercontent.com/postman-cs/cse-exercise/main/specs/payment-refund-api-openapi.yaml
   Verify it's non-empty and starts with `openapi: 3.0.3`.

2. Create everything else in the §4 PR 0 file tree:
   - .gitignore (content from BLUEPRINT.md §4 PR 0 step 3)
   - .claude/settings.json (BLUEPRINT.md §5)
   - .claude/commands/log-issue.md (BLUEPRINT.md §6)
   - .claude/commands/run-and-watch.md (BLUEPRINT.md §6)
   - .claude/commands/validate-postman.md (BLUEPRINT.md §6)
   - .github/workflows/.gitkeep (empty)
   - postman/collections/.gitkeep (empty)
   - service.config.yml (BLUEPRINT.md §7)
   - issues-log.md (BLUEPRINT.md §8)
   - README.md stub (BLUEPRINT.md §4 PR 0 step 8)

3. Run `tree -a -I '.git'` and show me the output.

4. STOP. Do NOT commit or push yet. Wait for my review.
```

**Expect:** All files created. Tree output matches BLUEPRINT.md §4 PR 0 layout.
Agent waits.

**Decide:** Does the tree match? Any missing files? Any extra files? Spec is
verbatim from brief (no edits)? No secret values in any file?

---

### Turn 0.2 — Commit and push

Paste:

```
Turn 0.2 — Tree reviewed; proceed to commit.

Steps:
1. `git add -A`
2. Commit with the message template from BLUEPRINT.md §4 PR 0.
3. Push to origin/main (or to your default branch).
4. Walk me through the PR 0 acceptance gate (4 boxes in BLUEPRINT.md §4 PR 0).
5. STOP after acceptance gate confirmed.
```

**Expect:** Commit hash, push success, acceptance gate walked.

**Decide:** PR 0 done. Move to PR 1.

---

## PR 1 — Workflow file + first green run

Goal: working `.github/workflows/onboard.yml` that runs end-to-end and produces
all expected Postman artifacts.

### Turn 1.1 — Verify environment and inspect upstream action

Paste:

```
Turn 1.1 — Verify secrets and inspect upstream action contract.

Steps:
1. `gh secret list`. Confirm POSTMAN_API_KEY and POSTMAN_ACCESS_TOKEN both present. STOP and report if either is missing.

2. Fetch the upstream action.yml:
   gh api repos/postman-cs/postman-api-onboarding-action/contents/action.yml \
     --jq .content | base64 -d > /tmp/upstream-action.yml
   head -100 /tmp/upstream-action.yml

3. Check latest tag:
   gh api repos/postman-cs/postman-api-onboarding-action/tags --jq '.[0].name'

4. Compare input names in /tmp/upstream-action.yml against the workflow shape
   in BLUEPRINT.md §4 PR 1.

5. Report back:
   - Both secrets present? (yes/no)
   - List of input names from upstream action.yml
   - Latest immutable tag (or "rolling @v0 only")
   - Any input name drift between upstream and BLUEPRINT shape — be specific

STOP and wait for my decision on which tag to pin (@v0 rolling vs v0.x.y immutable).
```

**Expect:** Secret check, input list from upstream, tag info, drift summary.

**Decide:** Pin to `@v0` (rolling, gets fixes) or `v0.x.y` (immutable, reproducible).
For a demo / submission, `v0.x.y` is safer. If no immutable tag exists, use `@v0`.

---

### Turn 1.2 — Draft workflow file

Paste (substitute placeholders first):

```
Turn 1.2 — Draft the onboarding workflow file. DO NOT COMMIT.

Inputs:
- YOUR_EMAIL: <your email>
- YOUR_POSTMAN_USER_ID: <your numeric Postman user ID>
- OWNER: <your github username or org>
- ACTION_REF: <@v0 or v0.x.y — your decision from Turn 1.1>

Steps:
1. Create branch: `git checkout -b pr-1-workflow`
2. Create .github/workflows/onboard.yml from the shape in BLUEPRINT.md §4 PR 1
   "Workflow file shape" section, substituting:
   - <YOUR_EMAIL> → my email above
   - <YOUR_POSTMAN_USER_ID> → my user ID above
   - <OWNER> in spec-url → OWNER above
   - @v0 in the uses: line → ACTION_REF above
3. Show me the complete file content in your response.
4. Run `cat .github/workflows/onboard.yml` to confirm it's saved.
5. STOP. Do NOT commit or push yet.
```

**Expect:** Full file content, all placeholders substituted, file saved to disk.

**Decide:** Review the file carefully. Verify each input is correct. Verify the
`spec-url` matches your repo's raw URL pattern.

---

### Turn 1.3 — Commit, push, watch first run

Paste:

```
Turn 1.3 — Workflow file approved. Commit, push, and watch the first run.

Steps:
1. `git add .github/workflows/onboard.yml`
2. Commit with message: "PR 1 WIP: initial onboard workflow draft"
   (We'll squash to the final message after we get green.)
3. Push the branch.
4. Either merge to main or run `gh workflow run onboard.yml --ref pr-1-workflow`
   to trigger. Decide which based on the workflow's trigger config — if it only
   fires on push to main, merge; if workflow_dispatch is enabled, run on the branch.
   Tell me which path you took.
5. Run /run-and-watch.
6. Report:
   - Run ID
   - Status (success / failure)
   - If failure: paste the most relevant ~30 lines of failed log
   - If success: list every output (workspace name/URL, spec ID, collection IDs, env IDs, monitor ID, mock URL)

STOP and wait.
```

**Expect:** Run completes with status reported. Outputs extracted on success.

**Decide:** Green → skip to Turn 1.6. Failed → continue to Turn 1.4.

---

### Turn 1.4 — Diagnose a failed run (use only if Turn 1.3 failed)

Paste:

```
Turn 1.4 — Diagnose the failed run.

Steps:
1. Pull more log context if needed:
   gh run view <RUN_ID> --log-failed
2. Identify the root cause — be specific:
   - Which step failed?
   - What exact error?
   - What input or condition triggered it?
3. Propose the SMALLEST possible fix. Do NOT apply it yet.
4. Invoke /log-issue and append the entry to issues-log.md with:
   - Tried
   - Error (exact text)
   - Proposed change
   - Source: where did the fix idea come from (action.yml? docs? AI guess?)
5. Report your diagnosis and proposed fix. STOP and wait for my approval.
```

**Expect:** Specific diagnosis, smallest-possible fix, logged issue entry.

**Decide:** Approve the fix, modify it, or reject and ask agent to re-diagnose.

---

### Turn 1.5 — Apply fix and re-run (use after each Turn 1.4)

Paste:

```
Turn 1.5 — Apply the approved fix.

Steps:
1. Apply the fix exactly as discussed. Show me the diff before committing.
2. Commit with a descriptive message (NOT a generic "fix").
3. Push.
4. Run /run-and-watch.
5. Report status (success / failure + relevant log if failed).
6. STOP.

If still failed: I'll send Turn 1.4 again. Do NOT squash any commits. The
iteration history is evidence the brief evaluates on.
```

**Expect:** Fix applied, new run status reported.

**Decide:** Green → Turn 1.6. Failed again → back to Turn 1.4.

---

### Turn 1.6 — UI validation walkthrough

Paste:

```
Turn 1.6 — Run is green. Walk me through Postman UI validation.

Steps:
1. Run /validate-postman.
2. For each artifact (workspace, spec, 3 collections, envs, monitor):
   - Tell me the URL
   - Tell me what to verify on that page
   - Wait for me to type "ok" or paste an issue before moving on
3. If I paste an issue at any point, invoke /log-issue immediately.
4. After the final artifact is confirmed, ask me: "Ready to test idempotency
   by re-running with no changes?"

STOP and wait at each artifact.
```

**Expect:** Numbered walkthrough. Agent waits between each item.

**Decide:** At each step, confirm artifact is correct. At the end, decide whether
to proceed to Turn 1.7 (still in PR 1, finalize) or jump straight to PR 2 (Turn 2.1).

---

### Turn 1.7 — Finalize PR 1

Paste:

```
Turn 1.7 — Finalize PR 1.

Steps:
1. Confirm postman/collections/ has JSON files committed by repo-sync. List them.
2. Check issues-log.md has every iteration from PR 1 logged.
3. Compose the final commit message per BLUEPRINT.md §4 PR 1 commit message
   template, filling in:
   - The green run URL
   - The date you verified upstream action.yml
4. Decide with me whether to:
   (a) Squash all PR 1 commits into one with the final message, OR
   (b) Keep iteration commits and add the final message as a merge commit
   The brief evaluates on honest debugging documentation — option (b) is
   safer. Recommend that unless I tell you otherwise.
5. Walk me through the 9-box acceptance gate in BLUEPRINT.md §4 PR 1.
6. Merge to main.

STOP after merge confirmed.
```

**Expect:** Acceptance gate walked. PR 1 merged.

**Decide:** PR 1 done. Move to PR 2.

---

## PR 2 — Idempotency + Postman validation evidence

Goal: document what actually happens on re-run and on spec change.

### Turn 2.1 — Re-run with no changes and observe deltas

Paste:

```
Turn 2.1 — Idempotency test: re-run with no changes.

Steps:
1. `git checkout -b pr-2-validation`
2. `gh workflow run onboard.yml`
3. /run-and-watch
4. After green, compare each Postman artifact to its PR 1 baseline:
   - Workspace: same one (reused), or new (duplicated)?
   - Spec in Spec Hub: updated in place, or duplicated?
   - Collections: same IDs (overwritten), or new IDs (duplicated)?
   - Environments: same, updated, or duplicated?
   - Monitor: same, recreated, or duplicated?
5. For each artifact, give me:
   - The Postman URL to look at
   - What you expect (based on action behavior, not guesses)
   - What I should verify
6. Wait for my "ok" or issue paste after each.
7. Report the final delta summary.

STOP after report.
```

**Expect:** Re-run completes, agent walks through each artifact comparison.

**Decide:** Any duplication is a critical finding. Log it. Decide whether the
behavior is acceptable for the demo or needs investigation.

---

### Turn 2.2 — Write docs/idempotency.md

Paste:

```
Turn 2.2 — Write docs/idempotency.md.

Steps:
1. mkdir -p docs
2. Draft docs/idempotency.md with three sections per BLUEPRINT.md §4 PR 2 step 4:
   - Section 1: Inputs/mechanisms that produced idempotency (e.g., the action
     reads repo variables like POSTMAN_WORKSPACE_ID set by the first run).
     Be specific based on what you observed.
   - Section 2: Observed deltas table (artifact / first run / second run / verdict).
     Populate from Turn 2.1 observations.
   - Section 3: One-sentence answer for plan §12 Q5 ("What happens on re-run? Duplicates?").
3. Show me the full file content. STOP. Do NOT commit yet.
```

**Expect:** Full file content with concrete observations, not predictions.

**Decide:** Approve, or request fixes if section 3 sounds hedged.

---

### Turn 2.3 — Screenshot capture walkthrough

Paste:

```
Turn 2.3 — Walk me through capturing screenshots.

Steps:
1. mkdir -p docs/screenshots
2. List the 5 required screenshots:
   - workspace-home.png
   - spec-hub.png
   - collection-baseline-expanded.png
   - environments-view.png
   - monitor-results.png
3. For each:
   - Tell me the Postman URL to navigate to
   - Tell me what should be visible in the screenshot
   - Wait for me to confirm "saved" before moving to the next
4. After all 5: run `ls docs/screenshots/` to confirm.

STOP and wait at each.
```

**Expect:** Numbered walkthrough, waits at each screenshot.

**Decide:** Confirm each screenshot is captured before moving on.

---

### Turn 2.4 — Spec-change scenario

Paste:

```
Turn 2.4 — Spec-change scenario test.

Steps:
1. Edit specs/payment-refund-api-openapi.yaml: bump info.version from 2.1.0 to 2.1.1.
   Show me the diff before committing.
2. Commit and push.
3. /run-and-watch (the push trigger should fire on spec path).
4. After complete, observe what changed in Postman UI:
   - Spec in Spec Hub: version updated?
   - Collections: regenerated, or same?
   - Environments: unchanged?
   - Monitor: unchanged?
5. Append the observed behavior to docs/idempotency.md as a new section
   "Behavior on Spec Change".
6. Show me the updated section. STOP.
```

**Expect:** Spec edit, run triggered, observations appended to doc.

**Decide:** Was the behavior expected? Anything to flag in issues-log.md?

---

### Turn 2.5 — Finalize PR 2

Paste:

```
Turn 2.5 — Finalize PR 2.

Steps:
1. `git add docs/`
2. Commit per BLUEPRINT.md §4 PR 2 commit message template, filling in:
   - The observed re-run behavior in one phrase
   - The observed spec-change behavior in one phrase
3. Push.
4. Walk me through the 4-box acceptance gate in BLUEPRINT.md §4 PR 2.
5. Merge to main.

STOP after merge confirmed.
```

**Expect:** Acceptance gate walked. PR 2 merged.

**Decide:** PR 2 done. Move to PR 3.

---

## PR 3 — README sections 1-4

Goal: top half of the README — overview, structure, architecture.

### Turn 3.1 — Draft sections 1-4

Paste (substitute first):

```
Turn 3.1 — Draft README sections 1-4. DO NOT COMMIT.

Inputs:
- LOAN_ORIGINATION_REPO_URL: <https://github.com/<owner>/loan-origination-postman-onboarding>

Steps:
1. `git checkout -b pr-3-readme-top`
2. Replace README.md (currently stub) with sections 1-4 per BLUEPRINT.md §4 PR 3:
   - Section 1: Overview (one paragraph)
   - Section 2: Services Onboarded (include LOAN_ORIGINATION_REPO_URL cross-link)
   - Section 3: Repository Structure (annotated tree)
   - Section 4: Workflow Architecture (action chain explanation)
3. For section 4: pull facts from the actual `.github/workflows/onboard.yml`.
   Do NOT invent. Reference the action chain as documented in the upstream
   `postman-cs/postman-api-onboarding-action` README if needed.
4. Show me the complete README. STOP. Do NOT commit.
```

**Expect:** Full README content for sections 1-4.

**Decide:** Approve. Pay particular attention to section 4 — it should match
the workflow file, not generic Postman docs.

---

### Turn 3.2 — Commit PR 3

Paste:

```
Turn 3.2 — Commit PR 3.

Steps:
1. `git add README.md`
2. Commit per BLUEPRINT.md §4 PR 3 commit message template.
3. Push.
4. Walk me through the 3-box acceptance gate in BLUEPRINT.md §4 PR 3.
5. Merge to main.

STOP after merge.
```

**Expect:** Acceptance gate walked. PR 3 merged.

**Decide:** PR 3 done. Move to PR 4.

---

## PR 4 — README sections 5-9

Goal: the evaluator-relevant sections. Each grounded in observation.

### Turn 4.1 — Section 5 (Universal vs Per-Service matrix)

Paste:

```
Turn 4.1 — Draft README section 5. DO NOT COMMIT.

Steps:
1. `git checkout -b pr-4-readme-mid`
2. Write section 5 (Universal Pattern vs Per-Service Differences) per
   BLUEPRINT.md §4 PR 4 step 5.
3. Use the starter rows in BLUEPRINT.md, but VERIFY each row against:
   - The actual workflow file inputs (`.github/workflows/onboard.yml`)
   - The actual spec
   - What I observed in Postman UI
4. Drop any row you can't verify against the repo state.
5. Add any row that surfaced in PR 1-2 that isn't in the starter list.
6. Show me the section content (do not write to README.md yet). STOP.
```

**Expect:** Matrix with observed (not predicted) values.

**Decide:** Approve. Push back on any row that looks copy-pasted from the
BLUEPRINT skeleton without verification.

---

### Turn 4.2 — Section 6 (What checks prove vs not)

Paste:

```
Turn 4.2 — Draft README section 6. DO NOT COMMIT.

Steps:
1. Read the spec (`specs/payment-refund-api-openapi.yaml`) for business rules
   in description fields (the 180-day refund window, max 5 refunds per txn, etc.).
2. Write section 6 (What Generated Checks Validate vs What Still Needs
   Service-Specific Knowledge) per BLUEPRINT.md §4 PR 4 step 6.
3. Adapt to THIS spec's specifics — the section should clearly demonstrate
   awareness of the payment-refund business rules, not be generic.
4. Show me the section. STOP.
```

**Expect:** Section with concrete examples from this spec.

**Decide:** Approve. Reject if it's generic Postman-doc filler.

---

### Turn 4.3 — Sections 7-9

Paste:

```
Turn 4.3 — Draft README sections 7, 8, 9. DO NOT COMMIT.

Steps:
1. Section 7 (Customer-Side Requirements): six items per BLUEPRINT.md §4 PR 4 step 7.
2. Section 8 (Run Instructions): copy-paste runnable. Test by reading: "could
   someone without my context follow this?" Use the command examples in
   BLUEPRINT.md §4 PR 4 step 8.
3. Section 9 (Validation Evidence): link to the green run from PR 1,
   link to committed postman/collections/, embed 2-3 screenshots from
   docs/screenshots/ via relative markdown image links.
4. Now write all of sections 5, 6, 7, 8, 9 into README.md in one edit (appending
   after section 4).
5. Show me the final README. STOP.
```

**Expect:** Full README sections 1-9 in place.

**Decide:** Read through. Anything aspirational? Anything you can't back up?

---

### Turn 4.4 — Commit PR 4

Paste:

```
Turn 4.4 — Commit PR 4.

Steps:
1. `git add README.md`
2. Commit per BLUEPRINT.md §4 PR 4 commit message template.
3. Push.
4. Walk me through the 4-box acceptance gate in BLUEPRINT.md §4 PR 4.
5. Merge to main.

STOP after merge.
```

**Expect:** Acceptance gate walked. PR 4 merged.

**Decide:** PR 4 done. Move to PR 5.

---

## PR 5 — README sections 10-14 + AI disclosure

Goal: close out the README. AI disclosure must be specific.

### Turn 5.1 — Sections 10-13

Paste:

```
Turn 5.1 — Draft README sections 10-13. DO NOT COMMIT.

Steps:
1. `git checkout -b pr-5-readme-close`
2. Section 10 (Rerun / Idempotency Behavior): one-sentence summary based on
   docs/idempotency.md; link to that file.
3. Section 11 (Known Issues and Resolutions): read issues-log.md. For each
   issue that involved ≥1 hour of friction OR ≥1 AI correction, write a
   paragraph (tried / error / changed / source of fix). Target ≥3 entries.
   If issues-log.md has fewer than 3 substantive entries, flag this to me —
   it means the build went suspiciously smoothly OR the logging was lax.
4. Section 12 (Trade-offs): use the candidates in BLUEPRINT.md §4 PR 5 step 12.
   Add anything from THIS build that surprised us.
5. Section 13 (Scaling Considerations): one paragraph pointing at the deck's
   90-day roadmap (deck lives outside this repo — don't try to include it).
6. Show me sections 10-13. STOP. Do NOT write to README.md yet.
```

**Expect:** Four sections drafted. Section 11 has real entries.

**Decide:** If section 11 is thin, dig into issues-log.md yourself to verify.
This section is graded.

---

### Turn 5.2 — Section 14 (AI disclosure)

Paste:

```
Turn 5.2 — Draft README section 14 (AI disclosure). DO NOT COMMIT.

Steps:
1. Read the entire issues-log.md.
2. Use the template in BLUEPRINT.md §4 PR 5 "AI disclosure template".
3. For each of the four subsections:
   - "What AI generated" — list specific items (not "helped with stuff")
   - "What I validated manually" — list specific verification steps
   - "What AI got wrong (and I corrected)" — at minimum 3 specific items.
     Every entry MUST reference a real iteration from issues-log.md.
     If you can't trace an entry to issues-log.md, REMOVE it.
   - "Why this matters" — one paragraph
4. Show me the section. STOP. Wait for my review.

Anti-pattern check: If any "What AI got wrong" entry could apply to any project
(e.g., "AI was overconfident"), it's too generic. Replace with the specific
correction from issues-log.md or remove.
```

**Expect:** AI disclosure with specific, traceable entries.

**Decide:** Read every "What AI got wrong" entry. Reject anything generic.
Cross-check against issues-log.md.

---

### Turn 5.3 — Final review, commit, submit

Paste:

```
Turn 5.3 — Finalize PR 5 and prepare for submission.

Steps:
1. Write sections 10, 11, 12, 13, 14 into README.md in one edit (appending
   after section 9).
2. Do a final read-through of the COMPLETE README. For each section, ask:
   - Is anything aspirational rather than done?
   - Is anything a placeholder that should have been filled in?
   - Does this match what the repo actually contains?
   Flag everything you find. Don't fix without my approval.
3. After any flagged items are resolved: commit per BLUEPRINT.md §4 PR 5 commit
   message template. Push.
4. Walk me through the 4-box acceptance gate in BLUEPRINT.md §4 PR 5.
5. Merge to main.
6. Report:
   - Final repo URL
   - Total commit count across all 6 PRs
   - PR 0-5 merge status
   - Any open items I should know about before submitting

STOP after report.
```

**Expect:** Final README complete. PR 5 merged. Repo URL ready.

**Decide:** Run the §15.2 pre-submission checklist from the plan file. Submit.

---

## Recovery prompts (when things go sideways)

### R1 — Agent is making up information

```
Stop. You appear to be generating from memory rather than the repo state.

Re-ground:
1. Run `pwd && git status && git log --oneline -10`
2. Read CLAUDE.md and BLUEPRINT.md fully.
3. Read issues-log.md fully.
4. Tell me what PR and turn we're on based on the repo state.
5. Wait for my next instruction.
```

### R2 — Agent skipped a step or pushed without approval

```
Stop. You skipped a checkpoint.

Recovery:
1. `git log --oneline -10` — show me what was committed.
2. `git status` — show me the working tree.
3. If a commit was pushed that I didn't approve, do NOT revert without asking.
   Tell me what you did and what state we're in. I'll decide whether to revert
   or move on.
4. Re-read BLUEPRINT.md §10 (Anti-patterns).
5. Wait for my next instruction.
```

### R3 — A run keeps failing with the same error

```
We're looping on the same failure. Stop iterating fixes.

Steps:
1. Pull the full failure log: `gh run view <RUN_ID> --log`
2. Search the log for the FIRST error (not the last) — root cause is usually
   earlier than the visible failure.
3. Hypothesize root cause. Be specific.
4. Identify what we have NOT yet tried:
   - Inspected upstream action.yml for input schema mismatch?
   - Verified secret values are set, not just secret names?
   - Verified the spec-url is actually fetchable (try `curl -I <url>`)?
   - Checked GitHub Actions permissions (actions: write, contents: write)?
   - Tried with a smaller test input set?
5. Propose ONE new diagnostic to try (not a fix — a diagnostic).
6. Wait for my approval before running it.
```

### R4 — Agent is generating verbose responses that eat context

```
Stop. Your responses are too long.

New rule for the rest of this session: 
- Lead with the answer in 1-3 sentences.
- Show output, then stop. No restating what you did.
- No preamble like "I'll now..." or "Let me...". Just do.
- Tables and code blocks only when actually useful.

Acknowledge with "ok" and continue from where we were.
```

### R5 — User wants to skip ahead to a later PR

```
I want to skip ahead to PR <N> Turn <M>. Confirm the prerequisites are met:

1. List the prerequisites for that PR from BLUEPRINT.md §4 PR <N>.
2. For each prerequisite, check the repo state and tell me yes/no.
3. If any prerequisite is missing, tell me what's missing and STOP.
4. If all are met, proceed with that turn's prompt.
```

---

## When you're done

After PR 5 is merged:

- [ ] Run the §15.2 pre-submission checklist from the plan file
- [ ] Re-read the README end-to-end one more time as if you've never seen the repo
- [ ] Verify both repos (this one + loan-origination) are publicly accessible OR
      evaluator email has been added as collaborator
- [ ] Capture the repo URL for the submission
- [ ] Switch context back to the deck (which lives outside this repo)

---

## Turn count summary

| PR | Turns | Cumulative |
|---|---|---|
| 0 | 2 | 2 |
| 1 | 7 (1.1–1.7; 1.4 and 1.5 loop on failures) | 9 |
| 2 | 5 | 14 |
| 3 | 2 | 16 |
| 4 | 4 | 20 |
| 5 | 3 | 23 |

**Target: 23 turns + recovery prompts as needed.** If you're 10 turns past the
target and not done, something's wrong — invoke R1 or R3.
