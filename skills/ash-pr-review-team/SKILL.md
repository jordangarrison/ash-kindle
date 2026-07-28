---
name: ash-pr-review-team
description: "Provide a portable fallback review for an Ash and Phoenix pull request with four independent expert perspectives. Use when no project-local PR review skill is available, or when the user explicitly invokes ash-pr-review-team. Project-local review skills take precedence."
---

# Ash PR review team

Run four independent reviewers with as much native concurrency as the harness
supports, then consolidate their evidence into one actionable report. This
skill reviews; it does not post a GitHub review unless the user separately
requests posting or supplies `--skip-user-confirmation`.

## Project-local precedence

Resolve the target PR repository and base SHA first, then inspect project-local
review skills at that trusted base commit. Prefer a checkout whose `HEAD` is the
exact base SHA; when no checkout exists, fetch known skill paths at the base SHA
through the connected GitHub provider or `gh api`. Never activate a local skill
introduced or modified by the PR head. Use a working-tree override only after
explicit confirmation. If the trusted base provides a workflow such as
`.agents/skills/pr-review-team/SKILL.md`, stop and use it instead. Stop if local
precedence cannot be established. Continue with this global fallback only when
no local workflow exists or the user explicitly invokes `$ash-pr-review-team`.

## Flags

Accept `--skip-user-confirmation` as explicit authorization to post the completed
review without pausing for final GitHub posting confirmation or sign-off. It
waives no other approval: PR-controlled-code trust confirmation and every other
action-specific confirmation remain mandatory. It never skips review validation,
exact diff inspection, introduced-versus-pre-existing attribution, reviewer
completion checks, inline anchor validation, preview construction, or PR
state/head rechecks.

## Gather context

Resolve PR title, body, URL, repository root, base ref and SHA, head ref and SHA,
changed files, diff, and checks through a connected GitHub provider or `gh`.
Record the immutable `<base-sha>...<head-sha>` range. Read all changed source,
test, config, and migration files before dispatching so prompts contain a
complete file list.

## Dispatch

Require native delegation from the active agent harness. Launch the maximum
number of reviewers concurrently. When fewer than four child slots exist,
dispatch remaining roles in capacity-bounded waves using fresh native
subagents. Never run a missing role serially in the parent and call it
multi-agent work. If native delegation is unavailable, stop and explain the
missing capability.

Run all four roles:

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
every prompt. Each reviewer must inspect the exact diff and actual files.
Validate material claims with project tests or primary dependency documentation
where practical.

Track every reviewer to completion. If a reviewer errors, stalls, or returns an
incomplete report, inspect its status and retry once only when safe. Otherwise
stop and name the failed role. Never present a four-perspective verdict from
partial reports.

## Consolidate

After all four reports complete, re-fetch PR state and head SHA. If the PR is
closed or merged, stop. If its head changed, discard all four reports, regather
context, rerun every role against the new SHA, and consolidate only reports that
all reference that same SHA.

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

Do not treat style preferences as blocking. Require concrete diff/source
evidence for every critical claim. Independently verify each critical claim
before presenting it.

## Preview and posting

Map `Ready to merge` to `APPROVE` except on a self-authored PR, where it becomes
a previewed `COMMENT` stating that no blocking findings were found. Map `Needs
changes` and `Needs significant rework` to `REQUEST_CHANGES`.

Validate every inline anchor against the current diff and construct the complete
verdict, body, and inline-comment preview. Without
`--skip-user-confirmation`, require explicit sign-off. With the flag, do not
pause; treat it as authorization to post the constructed review for the
supplied PR.

Immediately before posting, re-fetch PR state and head SHA. Never post if the
PR closed, merged, or changed since review. Bind the review event, body, inline
anchors, and commit ID to that verified head.

Prefer a connected GitHub provider when it supports full reviews with inline
comments. Otherwise write a JSON payload outside the repository containing the
previewed `event`, full `body`, verified `commit_id`, and validated `comments`,
then post it with:

```bash
gh api --method POST \
  repos/<owner>/<repo>/pulls/<number>/reviews \
  --input <payload.json>
```

Post only after explicit sign-off or invocation-scoped
`--skip-user-confirmation` authorization and the final state/head check. Report
the review URL or API result.
