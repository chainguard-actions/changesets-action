<!-- markdownlint-disable -->

# Hardening Report: changesets--action/v1.7.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **changesets--action/v1.7.0** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All three workflow files reference `actions/checkout@v4` using a mutable tag instead of a pinned 40-character commit SHA. This allows the upstream action to be silently replaced with malicious code. All occurrences: ci.yml (actions/checkout@v4), release-pr.yml (actions/checkout@v4), version-or-publish.yml (actions/checkout@v4).

Locations:

- `.github/workflows/ci.yml:16`
- `.github/workflows/release-pr.yml:33`
- `.github/workflows/version-or-publish.yml:12`

### permissions (severity: medium)

missing-permissions: None of the workflow files define a top-level `permissions:` key, and no job within them defines job-level permissions. This means workflows run with the default (potentially broad) token permissions.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/release-pr.yml:1`
- `.github/workflows/version-or-publish.yml:1`

### script-injection (severity: high)

rule-(a): Multiple `run:` blocks in release-pr.yml directly interpolate `${{ }}` expressions into shell commands. The workflow is triggered by `issue_comment`, meaning any GitHub user can influence these values. Affected steps and patterns:
- Line 15: `echo "in_progress_reaction_id=$(gh api /repos/${{github.repository}}/issues/comments/${{github.event.comment.id}}/reactions ...)" >> "$GITHUB_OUTPUT"` — github.repository and github.event.comment.id interpolated directly.
- Line 37: `run: gh pr checkout ${{ github.event.issue.number }}` — issue number interpolated directly into shell.
- Line 42: `echo "version_packages=$(gh pr view ${{ github.event.issue.number }} ...)" >> "$GITHUB_OUTPUT"` — issue number interpolated directly.
- Line 49: `run: yarn changeset version --snapshot pr${{ github.event.issue.number }}` — issue number interpolated directly.
- Line 57: `run: gh api /repos/${{ github.repository }}/issues/comments/${{ github.event.comment.id }}/reactions ...` — multiple expressions interpolated.
- Line 60: `run: gh api -X DELETE /repos/${{ github.repository }}/issues/comments/${{ github.event.comment.id }}/reactions/${{ needs.release_check.outputs.in_progress_reaction_id }}` — multiple expressions interpolated.
- Line 63: `run: gh pr comment ${{ github.event.issue.number }} --body "..."` — issue number and comment URL interpolated.
- Lines 72, 75, 78 (report-failure-if-needed job): same patterns repeated.

Locations:

- `.github/workflows/release-pr.yml:15`
- `.github/workflows/release-pr.yml:37`
- `.github/workflows/release-pr.yml:42`
- `.github/workflows/release-pr.yml:49`
- `.github/workflows/release-pr.yml:57`
- `.github/workflows/release-pr.yml:60`
- `.github/workflows/release-pr.yml:63`
- `.github/workflows/release-pr.yml:72`
- `.github/workflows/release-pr.yml:75`
- `.github/workflows/release-pr.yml:78`

### github-env-injection (severity: high)

Two `run:` steps in release-pr.yml write values derived from untrusted GitHub context expressions directly to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`):
1. The `report_in_progress` step (line 15) writes `${{github.repository}}` and `${{github.event.comment.id}}` directly into `$GITHUB_OUTPUT` via command substitution: `echo "in_progress_reaction_id=$(gh api /repos/${{github.repository}}/issues/comments/${{github.event.comment.id}}/reactions ...)" >> "$GITHUB_OUTPUT"`.
2. The `check_version_packages` step (line 42) writes `${{ github.event.issue.number }}` directly into `$GITHUB_OUTPUT`: `echo "version_packages=$(gh pr view ${{ github.event.issue.number }} ...)" >> "$GITHUB_OUTPUT"`. These unsanitized writes allow newline injection into the output file, potentially enabling environment variable poisoning in downstream steps.

Locations:

- `.github/workflows/release-pr.yml:15`
- `.github/workflows/release-pr.yml:42`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings across three workflow files:

1. **unpinned-uses**: Pinned `actions/checkout@v4` to full SHA `11d5960a326750d5838078e36cf38b85af677262` in ci.yml, release-pr.yml, and version-or-publish.yml.

2. **missing-permissions**: Added `permissions: {}` at the top level of all three workflow files. Added minimal job-level permissions where needed: `contents: write` + `pull-requests: write` for the changesets job in version-or-publish.yml; `issues: write` + `pull-requests: write` for release_check and report-failure-if-needed jobs in release-pr.yml; `contents: write` + `pull-requests: write` + `issues: write` for the release job in release-pr.yml.

3. **script-injection**: All `${{ }}` expressions in `run:` blocks in release-pr.yml moved to `env:` blocks and referenced as plain environment variables. This covers github.repository, github.event.comment.id, github.event.issue.number, github.event.comment.url, github.run_id, and needs.release_check.outputs.in_progress_reaction_id.

4. **github-env-injection**: The report_in_progress step (line 15) and check_version_packages step (line 42) now capture raw command output into a variable, sanitize with `printf '%s' "$raw" | tr -d '\n\r'`, then write the sanitized value to `$GITHUB_OUTPUT`.

### Iteration 2

**Fixes applied:** unpinned-uses

**Notes:**

Pinned `actions/setup-node@v6` to its full commit SHA `249970729cb0ef3589644e2896645e5dc5ba9c38` in `.github/actions/ci-setup/action.yml`, preserving the tag as a comment for readability.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted $AUTHOR_ASSOCIATION variable in the check_authorization step's bash [[ ]] comparison in .github/workflows/release-pr.yml. Added double quotes around all three occurrences of $AUTHOR_ASSOCIATION to prevent glob/pattern expansion on the left-hand side of the comparison.

