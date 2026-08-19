<!-- markdownlint-disable -->

# Hardening Report: changesets--action/v1.5.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **changesets--action/v1.5.1** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Both workflow files reference external actions using mutable version tags (@v4) instead of pinned full-length commit SHAs. This exposes the workflow to supply-chain attacks if the tag is moved to a malicious commit. Affected references: actions/checkout@v4 and actions/setup-node@v4 in both ci.yml and version-or-publish.yml.

Locations:

- `.github/workflows/ci.yml:12`
- `.github/workflows/ci.yml:15`
- `.github/workflows/version-or-publish.yml:14`
- `.github/workflows/version-or-publish.yml:18`

### missing-permissions (severity: medium)

Neither .github/workflows/ci.yml nor .github/workflows/version-or-publish.yml defines a top-level or job-level `permissions:` block. Without explicit permissions, the GITHUB_TOKEN is granted its default (broad) permissions, violating the principle of least privilege. Each workflow should declare minimal required permissions (e.g., `contents: read` for CI, `contents: write` and `pull-requests: write` for the release workflow).

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/version-or-publish.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions

**Notes:**

Fixed both workflow files: (1) Pinned actions/checkout@v4 to SHA 11d5960a326750d5838078e36cf38b85af677262 and actions/setup-node@v4 to SHA 49933ea5288caeca8642d1e84afbd3f7d6820020 in both ci.yml and version-or-publish.yml, preserving the original tag as a comment. (2) Added top-level permissions blocks: ci.yml gets `contents: read` (minimal for checkout/test), version-or-publish.yml gets `contents: write` and `pull-requests: write` (needed for creating release PRs and publishing).

