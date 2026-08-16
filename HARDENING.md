<!-- markdownlint-disable -->

# Hardening Report: clowdhaus--aws-lambda-code-signing-action/v1.5.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **clowdhaus--aws-lambda-code-signing-action/v1.5.2** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references are pinned to mutable tags or branches rather than immutable 40-character commit SHAs, making the workflow vulnerable to supply-chain attacks if those tags/branches are moved or compromised.

In `.github/workflows/integration.yml`:
- `uses: actions/checkout@v5` (tag `v5`)
- `uses: aws-actions/configure-aws-credentials@master` (branch `master`)

In `.github/workflows/release.yml`:
- `uses: actions/checkout@v5` (tag `v5`)
- `uses: actions/setup-node@v6` (tag `v6`)

All should be pinned to full SHA digests, e.g. `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/integration.yml:24`
- `.github/workflows/integration.yml:27`
- `.github/workflows/release.yml:20`
- `.github/workflows/release.yml:24`

### permissions (severity: medium)

Missing `permissions:` block in `.github/workflows/release.yml`. The file has no top-level `permissions:` key and the single `release` job also has no `permissions:` key. Without explicit permissions, the job inherits the default repository token permissions, which may be broader than necessary. A minimal permissions block (e.g. `contents: write` for releases, `id-token: none` for everything else) should be added.

Locations:

- `.github/workflows/release.yml:1`

### script-injection (severity: high)

Two `run:` steps in `.github/workflows/integration.yml` directly interpolate `${{ ... }}` expressions into shell command strings, violating sub-rule (a). GitHub Actions performs template substitution before the shell ever sees the string, so any special characters in the expanded value are interpreted by the shell.

**Step "Get archive version" (line 35–37):** `${{ secrets.AWS_S3_BUCKET }}` is interpolated directly into an `aws s3api` shell command. Even though `secrets.*` is not attacker-controlled in the traditional sense, any `${{ ... }}` expression inside a `run:` block is a script-injection finding per the check rules. The value should be passed via an `env:` variable and referenced as `"$AWS_S3_BUCKET"` in the shell.

**Step "Test outputs" (line 49–52):** `${{ steps.signed.outputs.job-id }}`, `${{ steps.signed.outputs.signed-object-key }}`, and `${{ steps.signed.outputs.renamed-signed-object-key }}` are all interpolated directly into `echo` commands. `steps.*.outputs.*` is an explicitly listed untrusted context. These values should be passed via `env:` variables and double-quoted in the shell.

Locations:

- `.github/workflows/integration.yml:36`
- `.github/workflows/integration.yml:50`
- `.github/workflows/integration.yml:51`
- `.github/workflows/integration.yml:52`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, script-injection

**Notes:**

Fixed all three findings across integration.yml and release.yml:

1. **unpinned-uses**: Pinned all four mutable action references to full 40-character commit SHAs:
   - `actions/checkout@v5` → `@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09 # v5` (both files)
   - `aws-actions/configure-aws-credentials@master` → `@ffc08eae7350b1061d7de219e2135c75561fb680 # master`
   - `actions/setup-node@v6` → `@249970729cb0ef3589644e2896645e5dc5ba9c38 # v6`

2. **permissions**: Added a top-level `permissions:` block to `release.yml` with `contents: write` (required for semantic-release to create releases/tags) and `id-token: none`.

3. **script-injection**: Moved all `${{ }}` expressions out of `run:` shell strings in `integration.yml` into `env:` blocks:
   - `${{ secrets.AWS_S3_BUCKET }}` → env var `AWS_S3_BUCKET`, referenced as `"$AWS_S3_BUCKET"` in the shell
   - `${{ steps.signed.outputs.job-id }}`, `${{ steps.signed.outputs.signed-object-key }}`, and `${{ steps.signed.outputs.renamed-signed-object-key }}` → env vars `JOB_ID`, `SIGNED_OBJECT_KEY`, `RENAMED_SIGNED_OBJECT_KEY`, referenced as `"$JOB_ID"`, `"$SIGNED_OBJECT_KEY"`, `"$RENAMED_SIGNED_OBJECT_KEY"` in the shell.

