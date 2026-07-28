---
name: ash-pr-review-team
description: Review an Ash and Phoenix pull request with four parallel expert perspectives covering Ash resource design, Phoenix LiveView UI, security, and performance/SRE. Use before merging a feature, after major implementation, or when the user asks for a thorough Ash/Phoenix PR review.
---

# Ash PR review team

Run four independent reviewers in parallel, then consolidate their evidence into
one actionable report. This skill reviews; it does not post a GitHub review
unless the user separately requests posting.

## Gather context

Resolve PR title, body, URL, base, head, changed files, diff, and checks through
a connected GitHub provider or `gh`. Read all changed source, test, config, and
migration files before dispatching so prompts contain a complete file list.

## Dispatch

Require native parallel delegation from the active agent harness. If unavailable,
stop and explain that this workflow requires four independent agents; do not
simulate them with serial passes.

Launch these reviewers together:

1. Ash Framework Expert:
   [references/ash-framework.md](references/ash-framework.md)
2. UI/LiveView Expert:
   [references/liveview-ui.md](references/liveview-ui.md)
3. Security Reviewer:
   [references/security.md](references/security.md)
4. Performance/SRE Reviewer:
   [references/performance-sre.md](references/performance-sre.md)

Replace `{PR_TITLE}`, `{PR_SUMMARY}`, `{FILES_LIST}`, and `{BRANCH}` in every
prompt. Each reviewer must inspect actual files and verify claims with project
tests or official dependency docs where practical.

## Consolidate

After all four reports complete:

1. Deduplicate overlapping findings and note independent agreement.
2. Reject unsupported assertions and findings unrelated to the PR.
3. Classify remaining items as Critical, Important, Minor, or Positive.
4. Produce:
   - verdict: Ready to merge, Needs changes, or Needs significant rework;
   - critical and important findings with evidence;
   - concise minor follow-ups;
   - positives;
   - ordered action plan with rough effort.

Do not treat style preferences as blocking. Independently verify every critical
claim before presenting it.
