<!-- markdownlint-disable -->

# Hardening Report: changesets--action/v1.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **changesets--action/v1.8.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `run:` blocks in release-pr.yml directly interpolate `${{ ... }}` expressions inside shell commands (sub-rule a), and several steps expand env vars without double-quoting (sub-rule b).

Sub-rule (a) — direct expression interpolation in run: blocks:
- Step `report_in_progress` (line 20): `echo "in_progress_reaction_id=$(gh api /repos/${{github.repository}}/issues/comments/${{github.event.comment.id}}/reactions ...)"` — `github.repository` and `github.event.comment.id` are interpolated directly into the shell command.
- Step `resolve_requested_sha` (lines 48–49): `gh api /repos/${{ github.repository }}/pulls/${{ github.event.issue.number }}/commits ...` and `gh pr view ${{ github.event.issue.number }} ...` — attacker-influenced values interpolated directly.
- Step `get_pr_head_repository` (line 69): `gh pr view ${{ github.event.issue.number }} ...` interpolated directly in run.
- Step `check_version_packages` (line 104): `gh pr view ${{ github.event.issue.number }} ...` interpolated directly in run.
- Unnamed step (line 110): `yarn changeset version --snapshot pr${{ github.event.issue.number }}` — `github.event.issue.number` interpolated directly.
- Multiple steps in `release` and `report-failure-if-needed` jobs (lines 118, 121, 124–127, 143, 146, 149–151): `gh api` and `gh pr comment` commands with `${{ github.repository }}`, `${{ github.event.comment.id }}`, `${{ github.event.issue.number }}`, `${{ needs.release_check.outputs.resolved_sha }}`, `${{ github.event.comment.url }}` interpolated directly.

Sub-rule (b) — unquoted shell variable expansion of untrusted data:
- Step `check_authorization` (line 34): `if [[ $AUTHOR_ASSOCIATION == 'MEMBER' || ... ]]` — `$AUTHOR_ASSOCIATION` (sourced from `github.event.comment.author_association`) is unquoted.
- Step `Fetch validated commit` (line 97): `git fetch --no-tags pr-head $RESOLVED_SHA` — `$RESOLVED_SHA` (sourced from `needs.release_check.outputs.resolved_sha`) is unquoted.
- Step `Checkout validated commit` (line 101): `git checkout --detach $RESOLVED_SHA` — `$RESOLVED_SHA` is unquoted.

Locations:

- `.github/workflows/release-pr.yml:20`
- `.github/workflows/release-pr.yml:34`
- `.github/workflows/release-pr.yml:48`
- `.github/workflows/release-pr.yml:69`
- `.github/workflows/release-pr.yml:97`
- `.github/workflows/release-pr.yml:101`
- `.github/workflows/release-pr.yml:104`
- `.github/workflows/release-pr.yml:110`
- `.github/workflows/release-pr.yml:118`
- `.github/workflows/release-pr.yml:121`
- `.github/workflows/release-pr.yml:124`
- `.github/workflows/release-pr.yml:143`
- `.github/workflows/release-pr.yml:146`
- `.github/workflows/release-pr.yml:149`

### github-env-injection (severity: high)

Three `run:` steps in release-pr.yml write values derived from untrusted GitHub context expressions to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`).

1. Step `report_in_progress` (line 20): Writes the output of `gh api /repos/${{github.repository}}/issues/comments/${{github.event.comment.id}}/reactions ... --jq '.id'` to `$GITHUB_OUTPUT`. The API response value (reaction ID) is written without sanitization: `echo "in_progress_reaction_id=$(...)" >> "$GITHUB_OUTPUT"`.

2. Step `get_pr_head_repository` (line 69): Writes the output of `gh pr view ${{ github.event.issue.number }} --json headRepositoryOwner,headRepository --jq '...'` to `$GITHUB_OUTPUT` without sanitization: `echo "head_repository=$(...)" >> "$GITHUB_OUTPUT"`. The issue number is attacker-influenced and the API response is written unsanitized.

3. Step `check_version_packages` (line 104): Writes the output of `gh pr view ${{ github.event.issue.number }} --json headRefName --jq '...'` to `$GITHUB_OUTPUT` without sanitization: `echo "version_packages=$(...)" >> "$GITHUB_OUTPUT"`. The branch name returned from the API could contain newline characters enabling injection into subsequent steps that consume this output.

Locations:

- `.github/workflows/release-pr.yml:20`
- `.github/workflows/release-pr.yml:69`
- `.github/workflows/release-pr.yml:104`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all script injection and github-env-injection findings in .github/workflows/release-pr.yml:

1. script-injection (sub-rule a): Moved all ${{ }} expressions from run: blocks into env: blocks across all steps in release_check, release, and report-failure-if-needed jobs. Variables like github.repository, github.event.comment.id, github.event.issue.number, needs.release_check.outputs.*, github.event.comment.url, and github.run_id are now passed via env: and referenced as $GH_REPOSITORY, $GH_COMMENT_ID, $GH_ISSUE_NUMBER, etc.

2. script-injection (sub-rule b): Added double-quotes around previously unquoted shell variables: $AUTHOR_ASSOCIATION in check_authorization step, and $RESOLVED_SHA in both the git fetch and git checkout steps.

3. github-env-injection: Added sanitization (printf '%s' "$raw" | tr -d '\n\r') before writing to $GITHUB_OUTPUT in three steps: report_in_progress (reaction ID), get_pr_head_repository (repository name), and check_version_packages (branch name check result).

