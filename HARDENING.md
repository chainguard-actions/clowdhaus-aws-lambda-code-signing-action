<!-- markdownlint-disable -->

# Hardening Report: clowdhaus--aws-lambda-code-signing-action/v1.5.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **clowdhaus--aws-lambda-code-signing-action/v1.5.1** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): GitHub Actions expressions are directly interpolated inside `run:` shell command strings, allowing template substitution before the shell processes the value. In the 'Get archive version' step, `${{ secrets.AWS_S3_BUCKET }}` is embedded directly in the shell command. In the 'Test outputs' step, `${{ steps.signed.outputs.job-id }}`, `${{ steps.signed.outputs.signed-object-key }}`, and `${{ steps.signed.outputs.renamed-signed-object-key }}` are all directly interpolated inside `echo` commands. Any `${{ ... }}` expression inside a `run:` block is a script-injection risk because YAML template substitution occurs before the shell ever sees the value. These should be passed via `env:` variables and then referenced as quoted shell variables (e.g., `"$VAR"`).

Locations:

- `.github/workflows/integration.yml:31`
- `.github/workflows/integration.yml:48`

### unpinned-uses (severity: high)

Multiple `uses:` references are pinned to mutable tags or branch names instead of immutable 40-character commit SHAs, making the workflow vulnerable to supply-chain attacks if the referenced tag or branch is moved or compromised. Failing references: in integration.yml — `actions/checkout@v4` (tag), `aws-actions/configure-aws-credentials@master` (branch); in release.yml — `actions/checkout@v4` (tag), `actions/setup-node@v4` (tag). Each should be pinned to a full SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/integration.yml:22`
- `.github/workflows/integration.yml:25`
- `.github/workflows/release.yml:16`
- `.github/workflows/release.yml:21`

### missing-permissions (severity: medium)

The `release.yml` workflow has no top-level `permissions:` key and its only job (`release`) also has no job-level `permissions:` key. Without an explicit permissions block, the workflow inherits the repository's default token permissions, which may be overly broad (e.g., `write` access to contents). A minimal explicit permissions block should be added (e.g., `permissions: contents: write` if semantic-release needs to push tags, and nothing else).

Locations:

- `.github/workflows/release.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all three findings: (1) script-injection in integration.yml — moved ${{ secrets.AWS_S3_BUCKET }} and three ${{ steps.signed.outputs.* }} expressions out of run: blocks into env: blocks, referencing them as quoted shell variables; (2) unpinned-uses — pinned actions/checkout@v4 to SHA 11d5960a326750d5838078e36cf38b85af677262, aws-actions/configure-aws-credentials@master to SHA ffc08eae7350b1061d7de219e2135c75561fb680, and actions/setup-node@v4 to SHA 49933ea5288caeca8642d1e84afbd3f7d6820020 in both workflow files; (3) missing-permissions — added top-level `permissions: contents: write` to release.yml since semantic-release needs to push tags and create releases.

