# busbar-actions/sf-pii-scan

Scan extracted Salesforce datasets for PII and secrets. Emits a JSON findings report, a SARIF report for GitHub Code Scanning, and a PR comment summary.

Built around the `sf-datasets` validator: catches SSN, email, phone, credit card, DOB, IP, passport, driver's license, bank account, medical ID; AWS / Stripe / GitHub / Slack / SendGrid / Twilio keys; private keys, OAuth tokens, database URLs, password fields; plus high-entropy strings as a generic secrets net.

## What it does

Wraps `busbar-sf data scan` over a file or directory of CSVs (typically the output of `busbar-sf data capture` or `data extract`). For each finding:

- Categorises it (e.g. `email`, `aws_access_key`, `high_entropy`).
- Assigns a severity (`info` / `low` / `medium` / `high` / `critical`).
- Records the SObject, field, and a *redacted* snippet of the match.

The action then publishes the report three ways: workflow artifact, GitHub Code Scanning (SARIF), and a PR comment summary.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `input-path` | yes | — | File or directory to scan. Directories are walked for CSVs. |
| `severity-threshold` | no | `medium` | Drop findings below this severity. One of `info`, `low`, `medium`, `high`, `critical`. |
| `fail-on-severity` | no | `critical` | Fail the workflow if any finding meets or exceeds this severity. Set to `never` to disable. |
| `config-file` | no | `` | Path to a `ValidationConfig` YAML (custom patterns, skip lists, entropy thresholds). |
| `report-path` | no | `pii-findings.json` | Where to write the JSON findings report. |
| `sarif` | no | `true` | Also emit a SARIF report. |
| `sarif-path` | no | `pii-findings.sarif` | Where the SARIF goes. |
| `upload-sarif` | no | `true` | Upload SARIF to GitHub Code Scanning (requires `security-events: write`). |
| `comment-pr` | no | `true` | Post a summary comment on pull_request events. |
| `upload-artifact` | no | `true` | Upload JSON (and SARIF) as a workflow artifact. |
| `artifact-name` | no | `pii-findings` | Artifact name. |
| `version` | no | `latest` | `busbar-sf` release tag. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Where to fetch the binary from. |

## Outputs

| Output | Description |
|---|---|
| `findings-count` | Total findings at or above `severity-threshold`. |
| `critical-count` | Critical findings. |
| `high-count` | High findings. |
| `medium-count` | Medium findings. |
| `threshold-breached` | `"true"` if `fail-on-severity` was met. |
| `report-path` | Path to the JSON report. |
| `sarif-path` | Path to the SARIF (empty if `sarif=false`). |

## Example: scan extracted dataset on PR

```yaml
name: PII Scan

on:
  pull_request:
    paths:
      - 'data/extracts/**'

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: busbar-actions/sf-pii-scan@v1
        with:
          input-path: data/extracts
          severity-threshold: medium
          fail-on-severity: critical
```

## Example: scan after capture (chained)

```yaml
name: Capture + Scan

on:
  workflow_dispatch:
    inputs:
      sobjects:
        description: 'Comma-separated SObjects to capture'
        required: true
        default: 'Account,Contact,Lead'

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  capture-and-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Capture data
        env:
          SF_ACCESS_TOKEN: ${{ secrets.SF_ACCESS_TOKEN }}
          SF_INSTANCE_URL: ${{ secrets.SF_INSTANCE_URL }}
        run: |
          # Direct binary use until a busbar-actions/sf-capture action exists.
          busbar-sf data capture \
            --sobjects "${{ inputs.sobjects }}" \
            --output ./captured

      - uses: busbar-actions/sf-pii-scan@v1
        with:
          input-path: ./captured
          severity-threshold: low      # be noisy during dev
          fail-on-severity: critical
```

## Example: custom validation config

```yaml
- uses: busbar-actions/sf-pii-scan@v1
  with:
    input-path: data/extracts
    config-file: .busbar/pii-scan.yml
```

Where `.busbar/pii-scan.yml` is a [`ValidationConfig`](https://github.com/busbar-agency/busbar-extensions/blob/main/crates/sf-datasets/src/pipeline/validator.rs) YAML — toggles, custom patterns, skip lists, and entropy thresholds.

## Dependencies (current status)

This action is fully scaffolded but **not yet runnable end-to-end**. Two upstream pieces need to land first:

1. **`busbar-sf data scan` subcommand** — the validator at `crates/sf-datasets/src/pipeline/validator.rs` is library-only today. We need a CLI surface that reads CSVs (single file or recursive dir), scans them with `DataValidator`, optionally loads a YAML `ValidationConfig`, and writes:
   - JSON: `[Finding, ...]` (the existing serde representation).
   - SARIF: `Finding` mapped to a SARIF v2.1.0 result — category → ruleId, severity → level, field+sobject → location, snippet → message text. One file per scanned source becomes a `physicalLocation`.
   - Exit code: non-zero when `--fail-on <severity>` is met or exceeded.

   Suggested flags:
   ```
   busbar-sf data scan \
     --input <file-or-dir> \
     [--config <validation-config.yml>] \
     [--severity <min-to-include>] \
     [--output <findings.json>] \
     [--format json|sarif|both] \
     [--sarif <findings.sarif>] \
     [--fail-on <severity>]
   ```

2. **Binary publication to `busbar-actions/actions-dist`** — same dependency as `busbar-actions/sf-schema`. A release pipeline in `busbar-extensions` that builds `busbar-sf` for five targets and pushes them to `actions-dist` releases under the asset names this action expects (see `action.yml` → "Determine asset name").

Once both land, tag this action `v1` and consumers can pin it.
