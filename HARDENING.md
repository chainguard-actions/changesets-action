<!-- markdownlint-disable -->

# Hardening Report: changesets--action/v1.5.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **changesets--action/v1.5.3** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks in release-pr.yml directly interpolate ${{ }} expressions inside shell commands. The workflow is triggered by issue_comment, making github.event.comment.id, github.event.comment.url, github.event.issue.number, and github.event.comment.author_association attacker-controllable. Specific violations:

(a) Line 14: `echo "in_progress_reaction_id=$(gh api /repos/${{github.repository}}/issues/comments/${{github.event.comment.id}}/reactions ...)" >> "$GITHUB_OUTPUT"` — direct interpolation of github context in run: block.
(a) Line 35: `run: gh pr checkout ${{ github.event.issue.number }}` — direct interpolation.
(a) Line 40: `echo "version_packages=$(gh pr view ${{ github.event.issue.number }} ...)" >> "$GITHUB_OUTPUT"` — direct interpolation.
(a) Line 47: `run: yarn changeset version --snapshot pr${{ github.event.issue.number }}` — direct interpolation.
(a) Line 59: `run: gh api /repos/${{ github.repository }}/issues/comments/${{ github.event.comment.id }}/reactions ...` — direct interpolation.
(a) Line 62: `run: gh api -X DELETE .../reactions/${{ needs.release_check.outputs.in_progress_reaction_id }}` — direct interpolation.
(a) Line 65: `run: gh pr comment ${{ github.event.issue.number }} --body "... ${{ github.event.comment.url }} ..."` — direct interpolation.
(a) Lines 71-79: report-failure-if-needed job repeats the same patterns with ${{ github.event.comment.id }}, ${{ github.event.comment.url }}, ${{ github.event.issue.number }}.
(b) Line 22: `if [[ $AUTHOR_ASSOCIATION == 'MEMBER' ...` — the env var AUTHOR_ASSOCIATION (sourced from github.event.comment.author_association) is used unquoted in the shell conditional.

Locations:

- `.github/workflows/release-pr.yml:14`
- `.github/workflows/release-pr.yml:22`
- `.github/workflows/release-pr.yml:35`
- `.github/workflows/release-pr.yml:40`
- `.github/workflows/release-pr.yml:47`
- `.github/workflows/release-pr.yml:59`
- `.github/workflows/release-pr.yml:62`
- `.github/workflows/release-pr.yml:65`
- `.github/workflows/release-pr.yml:71`
- `.github/workflows/release-pr.yml:74`
- `.github/workflows/release-pr.yml:77`

### github-env-injection (severity: high)

Two run: steps in release-pr.yml write values derived from untrusted github context directly to $GITHUB_OUTPUT without the required sanitization step (printf '%s' ... | tr -d '\n\r').

1. Line 14 (report_in_progress step): `echo "in_progress_reaction_id=$(gh api /repos/${{github.repository}}/issues/comments/${{github.event.comment.id}}/reactions -f content='eyes' --jq '.id')" >> "$GITHUB_OUTPUT"` — the output of the gh api call (which includes attacker-influenced comment ID) is written directly to $GITHUB_OUTPUT without sanitization.

2. Line 40 (check_version_packages step): `echo "version_packages=$(gh pr view ${{ github.event.issue.number }} --json headRefName --jq '...')" >> "$GITHUB_OUTPUT"` — the issue number from github context is interpolated directly and the result written to $GITHUB_OUTPUT without sanitization.

Locations:

- `.github/workflows/release-pr.yml:14`
- `.github/workflows/release-pr.yml:40`

### unpinned-uses (severity: high)

Multiple workflow files and the ci-setup composite action reference external actions using mutable version tags (@v4) instead of immutable full-length commit SHAs. This exposes the workflow to supply-chain attacks if the tag is moved to a malicious commit.

Failing references:
- .github/workflows/ci.yml: `uses: actions/checkout@v4`
- .github/workflows/release-pr.yml: `uses: actions/checkout@v4`
- .github/workflows/version-or-publish.yml: `uses: actions/checkout@v4`
- .github/actions/ci-setup/action.yml: `uses: actions/setup-node@v4`

All should be pinned to a full 40-character commit SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/ci.yml:14`
- `.github/workflows/release-pr.yml:36`
- `.github/workflows/version-or-publish.yml:14`
- `.github/actions/ci-setup/action.yml:6`

### missing-permissions (severity: medium)

None of the three workflow files define a top-level `permissions:` block, and none of the individual jobs within them define job-level `permissions:` blocks. Without explicit permissions, workflows run with the repository's default token permissions, which may be overly broad (write access to contents, pull-requests, etc.). All three files are affected:
- .github/workflows/ci.yml
- .github/workflows/release-pr.yml
- .github/workflows/version-or-publish.yml

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/release-pr.yml:1`
- `.github/workflows/version-or-publish.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings across four files:

1. **script-injection** (release-pr.yml): Moved all ${{ github.event.comment.id }}, ${{ github.event.comment.url }}, ${{ github.event.issue.number }}, ${{ github.repository }}, and ${{ github.event.comment.author_association }} expressions out of run: blocks into env: blocks. Shell scripts now reference plain environment variables. The AUTHOR_ASSOCIATION variable is now properly double-quoted in the [[ ]] conditional.

2. **github-env-injection** (release-pr.yml): Both GITHUB_OUTPUT writes (in_progress_reaction_id and version_packages) now capture the raw value first, then sanitize with `printf '%s' "$raw" | tr -d '\n\r'` before writing to GITHUB_OUTPUT.

3. **unpinned-uses**: Pinned all four action references to full 40-char SHAs:
   - actions/checkout@v4 → @11d5960a326750d5838078e36cf38b85af677262 # v4 (ci.yml, release-pr.yml, version-or-publish.yml)
   - actions/setup-node@v4 → @49933ea5288caeca8642d1e84afbd3f7d6820020 # v4 (ci-setup/action.yml)

4. **missing-permissions**: Added top-level `permissions: {}` to all three workflow files, plus minimal job-level permissions: ci.yml build job gets `contents: read`; version-or-publish.yml changesets job gets `contents: write` + `pull-requests: write`; release-pr.yml jobs get `issues: write` + `pull-requests: write` (+ `contents: write` for the release job).

