<!-- markdownlint-disable -->

# Hardening Report: changesets--action/v1.5.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **changesets--action/v1.5.2** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Both workflow files reference GitHub Actions using mutable version tags (@v4) instead of full 40-character commit SHA digests. This exposes the workflow to supply-chain attacks if the tag is moved to a malicious commit. Affected references: actions/checkout@v4 and actions/setup-node@v4 in both ci.yml and version-or-publish.yml.

Locations:

- `.github/workflows/ci.yml:13`
- `.github/workflows/ci.yml:16`
- `.github/workflows/version-or-publish.yml:13`
- `.github/workflows/version-or-publish.yml:16`

### missing-permissions (severity: medium)

Neither workflow file defines a top-level `permissions:` key, and no job within either file defines job-level permissions. Without explicit permissions, workflows run with the default (potentially broad) token permissions. Minimal permissions should be declared — for example, `permissions: read-all` at the top level and then only the specific scopes needed per job.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/version-or-publish.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions

**Notes:**

Fixed both workflow files (.github/workflows/ci.yml and .github/workflows/version-or-publish.yml):
1. unpinned-uses: Pinned actions/checkout@v4 to SHA 11d5960a326750d5838078e36cf38b85af677262 and actions/setup-node@v4 to SHA 49933ea5288caeca8642d1e84afbd3f7d6820020 in both files, preserving the tag as a comment.
2. missing-permissions: Added top-level `permissions: {}` to both files to deny all permissions by default. Added job-level permissions: `contents: read` for the CI build job, and `contents: write` + `pull-requests: write` for the changesets job (required for creating release PRs and publishing packages).

