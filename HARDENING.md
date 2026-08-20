<!-- markdownlint-disable -->

# Hardening Report: changesets--action/v1.6.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **changesets--action/v1.6.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All three workflow files use `actions/checkout@v4` — a mutable tag reference, not a pinned 40-character commit SHA. This allows supply-chain attacks if the tag is moved to a malicious commit. Affected references: `actions/checkout@v4` in ci.yml (line 16), release-pr.yml (line 35), and version-or-publish.yml (line 12).

Locations:

- `.github/workflows/ci.yml:16`
- `.github/workflows/release-pr.yml:35`
- `.github/workflows/version-or-publish.yml:12`

### missing-permissions (severity: medium)

None of the three workflow files declare a top-level `permissions:` block, and no individual job declares `permissions:` either. Without explicit permissions, workflows inherit the repository's default token permissions (often `write-all`), violating the principle of least privilege.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/release-pr.yml:1`
- `.github/workflows/version-or-publish.yml:1`

### script-injection (severity: high)

Multiple `run:` blocks in release-pr.yml directly interpolate GitHub Actions expressions into shell commands (rule a). The workflow is triggered by `issue_comment`, meaning any user who can comment on an issue can influence `github.event.comment.*` and `github.event.issue.number`. Specific violations:
- Line ~14: `echo "...$(gh api /repos/${{github.repository}}/issues/comments/${{github.event.comment.id}}/reactions ...)" >> "$GITHUB_OUTPUT"` — `github.repository` and `github.event.comment.id` interpolated directly.
- Line ~40: `run: gh pr checkout ${{ github.event.issue.number }}` — issue number interpolated directly into shell.
- Line ~44: `echo "version_packages=$(gh pr view ${{ github.event.issue.number }} ...)" >> "$GITHUB_OUTPUT"` — issue number interpolated directly.
- Line ~53: `run: yarn changeset version --snapshot pr${{ github.event.issue.number }}` — issue number interpolated directly.
- Lines ~57–60: `gh api /repos/${{ github.repository }}/issues/comments/${{ github.event.comment.id }}/reactions ...` — multiple expressions interpolated.
- Line ~62: `gh pr comment ${{ github.event.issue.number }} --body "...[${{ github.event.comment.url }}]..."` — comment URL (attacker-controlled) interpolated directly into shell.
- Lines in report-failure-if-needed job: same patterns with `github.event.comment.id`, `github.event.issue.number`, `github.event.comment.url`.
All these must be moved to `env:` variables and the shell expansions must be double-quoted.

Locations:

- `.github/workflows/release-pr.yml:14`
- `.github/workflows/release-pr.yml:40`
- `.github/workflows/release-pr.yml:44`
- `.github/workflows/release-pr.yml:53`
- `.github/workflows/release-pr.yml:57`
- `.github/workflows/release-pr.yml:59`
- `.github/workflows/release-pr.yml:62`
- `.github/workflows/release-pr.yml:70`
- `.github/workflows/release-pr.yml:72`
- `.github/workflows/release-pr.yml:74`

### github-env-injection (severity: high)

Two `run:` blocks in release-pr.yml write to `$GITHUB_OUTPUT` with values derived from `${{ }}` expressions interpolated directly into the shell command, without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`):
1. The `report_in_progress` step (line ~14) writes `in_progress_reaction_id=$(gh api /repos/${{github.repository}}/issues/comments/${{github.event.comment.id}}/reactions ...)` to `$GITHUB_OUTPUT`. The `github.event.comment.id` and `github.repository` values are interpolated directly.
2. The `check_version_packages` step (line ~44) writes `version_packages=$(gh pr view ${{ github.event.issue.number }} ...)` to `$GITHUB_OUTPUT`. The `github.event.issue.number` value is interpolated directly.
In both cases, a newline embedded in the expression value could inject additional key=value pairs into `$GITHUB_OUTPUT`, affecting downstream steps.

Locations:

- `.github/workflows/release-pr.yml:14`
- `.github/workflows/release-pr.yml:44`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings across three workflow files:

1. **unpinned-uses**: Pinned `actions/checkout@v4` → `actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4` in ci.yml, release-pr.yml, and version-or-publish.yml.

2. **missing-permissions**: Added top-level `permissions: {}` to all three files. Added minimal job-level permissions where write access is needed (issues: write, pull-requests: write, contents: write).

3. **script-injection**: In release-pr.yml, moved all `${{ github.event.comment.id }}`, `${{ github.event.issue.number }}`, `${{ github.event.comment.url }}`, `${{ github.repository }}`, and `${{ github.run_id }}` expressions out of `run:` blocks into `env:` blocks. All shell references are double-quoted.

4. **github-env-injection**: Both GITHUB_OUTPUT writes that used github context values now sanitize with `printf '%s' "$VAR" | tr -d '\n\r'` before writing — specifically the `report_in_progress` step (reaction_id) and the `check_version_packages` step (version_packages result).

### Iteration 2

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

1. Pinned actions/setup-node@v6 to full SHA 249970729cb0ef3589644e2896645e5dc5ba9c38 in .github/actions/ci-setup/action.yml, preserving the tag as a comment. 2. Double-quoted all three $AUTHOR_ASSOCIATION expansions inside the [[ ]] conditional in .github/workflows/release-pr.yml (changed from `$AUTHOR_ASSOCIATION` to `"$AUTHOR_ASSOCIATION"` in each comparison), satisfying the script-injection rule for workflow-controllable data.

