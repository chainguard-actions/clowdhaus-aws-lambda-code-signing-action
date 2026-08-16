<!-- markdownlint-disable -->

# Hardening Report: clowdhaus--aws-lambda-code-signing-action/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **clowdhaus--aws-lambda-code-signing-action/v2.0.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): GitHub Actions expressions are directly interpolated inside run: shell command strings in two steps of integration.yml.

1. "Get archive version" step (line 82): `${{ secrets.AWS_S3_BUCKET }}` is embedded directly in a shell command: `LATEST_VERSION=$(aws s3api list-object-versions --bucket ${{ secrets.AWS_S3_BUCKET }} ...)`. Any ${{ }} expression in a run: block is a script-injection risk because YAML template substitution happens before the shell ever sees the string.

2. "Test outputs" step (lines 99-101): `${{ steps.signed.outputs.job-id }}`, `${{ steps.signed.outputs.signed-object-key }}`, and `${{ steps.signed.outputs.renamed-signed-object-key }}` are all directly interpolated in echo commands. The steps.*.outputs.* context is explicitly an untrusted/workflow-controllable source. These values should be passed via env: variables and then referenced as quoted shell variables (e.g., `echo "$STEP_OUTPUT"`).

Fix: Move all ${{ }} values into an env: block and reference them as double-quoted shell variables inside the run: script.

Locations:

- `.github/workflows/integration.yml:82`
- `.github/workflows/integration.yml:99`
- `.github/workflows/integration.yml:100`
- `.github/workflows/integration.yml:101`

### github-env-injection (severity: high)

The "Get archive version" step in integration.yml writes an unsanitized value to $GITHUB_ENV. The variable LATEST_VERSION is populated from the output of an AWS CLI call (`aws s3api list-object-versions ...`) and then written directly to the environment file with `echo "LATEST_VERSION=$LATEST_VERSION" >> $GITHUB_ENV` without first applying the required sanitization step (`printf '%s' "$LATEST_VERSION" | tr -d '\n\r'`). An attacker who can influence the S3 object version metadata could inject newlines into this value to poison the GitHub Actions environment, potentially overwriting arbitrary environment variables for subsequent steps.

Locations:

- `.github/workflows/integration.yml:83`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed two high-severity findings in .github/workflows/integration.yml:

1. **script-injection** (lines 82, 99-101): 
   - 'Get archive version' step: moved `${{ secrets.AWS_S3_BUCKET }}` into an `env:` block as `AWS_S3_BUCKET` and referenced it as `"$AWS_S3_BUCKET"` in the shell command.
   - 'Test outputs' step: moved all three `${{ steps.signed.outputs.* }}` expressions into an `env:` block (`JOB_ID`, `SIGNED_OBJECT_KEY`, `RENAMED_SIGNED_OBJECT_KEY`) and referenced them as double-quoted shell variables.

2. **github-env-injection** (line 83): In the 'Get archive version' step, the `LATEST_VERSION` value is now sanitized with `safe=$(printf '%s' "$LATEST_VERSION" | tr -d '\n\r')` before being written to `$GITHUB_ENV`, preventing newline injection attacks.

