---
name: ash-pr-review-team
description: "Provide a portable fallback review for an Ash and Phoenix pull request with four parallel expert perspectives. Use when no project-local PR review skill is available, or when the user explicitly invokes ash-pr-review-team. Project-local review skills take precedence."
---

# Ash PR review team

Run four independent reviewers in parallel, then consolidate their evidence into
one actionable report. This skill reviews; it does not post a GitHub review
unless the user separately requests posting.

## Project-local precedence

Before gathering PR context, inspect the skills available from the current
repository. If it provides a project-local PR review workflow such as
`.agents/skills/pr-review-team/SKILL.md`, stop and use that workflow instead.
Continue with this global fallback only when no local workflow exists or the
user explicitly invokes `$ash-pr-review-team`.

## Gather context

Resolve PR title, body, URL, repository root, base ref and SHA, head ref and SHA,
changed files, diff, and checks through a connected GitHub provider or `gh`.
Record the immutable `<base-sha>...<head-sha>` range. Read all changed source,
test, config, and migration files before dispatching so prompts contain a
complete file list.

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

Replace `{PR_TITLE}`, `{PR_SUMMARY}`, `{PR_URL}`, `{REPO_ROOT}`, `{BASE_REF}`,
`{BASE_SHA}`, `{BRANCH}`, `{HEAD_SHA}`, `{DIFF_RANGE}`, and `{FILES_LIST}` in
every prompt. Each reviewer must inspect the exact diff and actual files, then
verify claims with project tests or official dependency docs where practical.

Track every reviewer to completion. If a reviewer errors, stalls, or returns an
incomplete report, inspect its status and retry once only when safe. Otherwise
stop and name the failed role. Never present a four-perspective verdict from
partial reports.

## Consolidate

After all four reports complete, re-fetch PR state and head SHA. If the PR is
closed, merged, or its head changed, stop and refresh the affected reports
before consolidation.

Then:

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
