<!-- markdownlint-disable -->

# Hardening Report: clowdhaus--aws-lambda-code-signing-action/v1.5.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **clowdhaus--aws-lambda-code-signing-action/v1.5.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references are pinned to mutable tags or branches instead of immutable 40-character commit SHAs, making the workflow vulnerable to supply-chain attacks if the referenced tag or branch is moved or compromised.

In `.github/workflows/integration.yml`:
- `uses: actions/checkout@v4` (tag `v4`)
- `uses: aws-actions/configure-aws-credentials@master` (branch `master` — especially dangerous)

In `.github/workflows/release.yml`:
- `uses: actions/checkout@v4` (tag `v4`)
- `uses: actions/setup-node@v4` (tag `v4`)

All should be pinned to full SHA digests, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/integration.yml:22`
- `.github/workflows/integration.yml:25`
- `.github/workflows/release.yml:17`
- `.github/workflows/release.yml:23`

### script-injection (severity: high)

Two `run:` blocks in `.github/workflows/integration.yml` directly interpolate `${{ ... }}` expressions into shell commands (rule a), allowing template substitution to inject shell metacharacters before the shell ever parses the string.

(1) The "Get archive version" step (line 31) interpolates `${{ secrets.AWS_S3_BUCKET }}` directly into an `aws` CLI command:
```
LATEST_VERSION=$(aws s3api list-object-versions --bucket ${{ secrets.AWS_S3_BUCKET }} ...)
```
Although `secrets.*` is typically trusted, any `${{ ... }}` expression inside a `run:` block is a template-injection risk — the value is substituted verbatim into the shell script before execution.

(2) The "Test outputs" step (lines 45–47) interpolates `${{ steps.signed.outputs.job-id }}`, `${{ steps.signed.outputs.signed-object-key }}`, and `${{ steps.signed.outputs.renamed-signed-object-key }}` directly into `echo` commands. Step outputs are workflow-controllable data and must not be interpolated directly into shell.

Fix: route all values through `env:` variables and reference them as quoted shell variables (`"$VAR"`) inside the `run:` block.

Locations:

- `.github/workflows/integration.yml:31`
- `.github/workflows/integration.yml:45`

### missing-permissions (severity: medium)

`.github/workflows/release.yml` has no top-level `permissions:` key and its only job (`release`) also has no `permissions:` key. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (e.g. `write` on all scopes for public repositories). Explicit minimal permissions should be declared, e.g.:
```yaml
permissions:
  contents: write  # only what semantic-release needs
```

Locations:

- `.github/workflows/release.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, missing-permissions

**Notes:**

Fixed all three findings:

1. **unpinned-uses**: Pinned all four action references to full SHA digests:
   - `actions/checkout@v4` → `@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4` (both files)
   - `aws-actions/configure-aws-credentials@master` → `@ffc08eae7350b1061d7de219e2135c75561fb680 # master`
   - `actions/setup-node@v4` → `@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4`

2. **script-injection** (integration.yml): Moved all `${{ }}` expressions out of `run:` blocks into `env:` blocks:
   - "Get archive version" step: `${{ secrets.AWS_S3_BUCKET }}` → env var `AWS_S3_BUCKET`, referenced as `"$AWS_S3_BUCKET"` in shell
   - "Test outputs" step: three step outputs moved to env vars `JOB_ID`, `SIGNED_OBJECT_KEY`, `RENAMED_SIGNED_OBJECT_KEY`, referenced as quoted shell variables

3. **missing-permissions** (release.yml): Added top-level `permissions: contents: write` block, which is the minimum permission needed for semantic-release to publish releases and tags.

