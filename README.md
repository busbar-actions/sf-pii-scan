> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-pii-scan

Scan a captured Salesforce dataset for PII and credentials. Emits a JSON findings report, an optional SARIF report for GitHub Code Scanning, and an optional PR-comment summary.

Built on the `sf-datasets` `DataValidator`: catches SSN, email, phone, credit card, DOB, IP, passport, driver's license, bank account, and medical IDs; AWS / Stripe / GitHub / Slack / SendGrid / Twilio keys; private keys, OAuth tokens, database URLs, password fields; Salesforce-credential detectors (SFDX auth URL, session/access tokens, OAuth refresh tokens); plus high-entropy strings as a generic secrets net.

**No org auth required** — this action scans files on disk only. It never contacts Salesforce.

## What it does

Loads a dataset *directory* (the output of `busbar-sf data capture` / `data extract`) and validates every text value. The loader auto-detects the storage format: a directory containing `manifest.json` is read as CSV storage, otherwise it is read as the binary (sled) storage. It is not a recursive CSV walker — point it at a capture directory, not an arbitrary tree of CSVs.

For each finding the binary records its category (e.g. `email`, `aws_access_key`, `high_entropy`), severity (`info` / `low` / `medium` / `high` / `critical`), the SObject, field, and a redacted snippet of the match.

The single `sf-pii-scan` binary owns all logic and UX. Under `GITHUB_ACTIONS=true` it reads its `INPUT_*` contract, writes the JSON report (and SARIF), computes the finding counts, writes `GITHUB_OUTPUT`, a job-summary table, an annotation, and a PR-comment body, and exits non-zero when the `fail-on-severity` threshold is breached. The action itself only installs the binary, passes inputs through, and wires the three GitHub-native publish steps (Code Scanning upload, artifact upload, PR comment).

## Usage

```yaml
name: PII Scan

on:
  pull_request:
    paths:
      - 'data/extracts/**'

permissions:
  contents: read
  pull-requests: write      # only if comment-pr (default true)
  security-events: write    # only if upload-sarif (default true)

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: busbar-actions/sf-pii-scan@v1
        with:
          input-path: data/extracts   # a capture/extract directory
          severity-threshold: medium
          fail-on-severity: critical
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `input-path` | yes | — | Captured dataset directory to scan (CSV storage with `manifest.json`, or binary/sled storage). |
| `severity-threshold` | no | `medium` | Minimum severity to include in the report. One of `info`, `low`, `medium`, `high`, `critical`. |
| `fail-on-severity` | no | `critical` | Exit non-zero if any finding meets or exceeds this severity. Same scale; set to `never` to disable. |
| `config-file` | no | `` | Path to a `ValidationConfig` **JSON** file (custom patterns, skip lists, entropy thresholds). Optional. |
| `report-path` | no | `pii-findings.json` | Where to write the JSON findings report. |
| `sarif` | no | `true` | Also emit a SARIF v2.1.0 report. |
| `sarif-path` | no | `pii-findings.sarif` | Where to write the SARIF report (only used when `sarif=true`). |
| `upload-sarif` | no | `true` | Upload the SARIF to GitHub Code Scanning (requires `security-events: write`). |
| `comment-pr` | no | `true` | Post a findings summary as a PR comment on `pull_request` events (requires `pull-requests: write`). |
| `upload-artifact` | no | `true` | Upload the JSON report (and SARIF if generated) as a workflow artifact. |
| `artifact-name` | no | `pii-findings` | Name for the uploaded artifact. |
| `version` | no | `latest` | `sf-pii-scan` release tag to download (e.g. `v0.1.0`). `latest` resolves the most recent release. |
| `binary-repo` | no | `busbar-actions/actions-dist` | GitHub repo that publishes the `sf-pii-scan` binary releases. |

## Outputs

| Output | Description |
|---|---|
| `findings-count` | Total findings at or above `severity-threshold`. |
| `critical-count` | Number of Critical-severity findings. |
| `high-count` | Number of High-severity findings. |
| `medium-count` | Number of Medium-severity findings. |
| `threshold-breached` | `"true"` if any finding met or exceeded `fail-on-severity`. |
| `report-path` | Path to the JSON findings report (echoes the `report-path` input). |
| `sarif-path` | Path to the SARIF report (empty if `sarif=false`). |

## Auth / permissions model

This action does **not** consume a Salesforce access token and does **not** request `id-token: write` — it reads files that a prior capture/extract step wrote to the workspace. No OIDC trust onboarding is required.

The permissions it does need are GitHub-side, and only for the publish steps:

- `security-events: write` — to upload SARIF to Code Scanning (`upload-sarif`, default true).
- `pull-requests: write` — to post the PR comment (`comment-pr`, default true).
- `contents: read` — to check out the repository.

If you disable `upload-sarif` and `comment-pr`, `contents: read` is sufficient.

## Chaining with a capture step

`input-path` expects a dataset directory produced upstream. Until a `busbar-actions/sf-capture` action exists, capture with the binary directly. Capture is the step that needs Salesforce credentials (via the OIDC token-exchange model); the scan step that follows does not.

```yaml
- uses: actions/checkout@v4

- name: Capture data        # this step authenticates to SF; the scan does not
  run: |
    busbar-sf data capture --sobjects "Account,Contact,Lead" --output ./captured

- uses: busbar-actions/sf-pii-scan@v1
  with:
    input-path: ./captured
    severity-threshold: low     # be noisy during dev
    fail-on-severity: critical
```

## Custom validation config

```yaml
- uses: busbar-actions/sf-pii-scan@v1
  with:
    input-path: ./captured
    config-file: .busbar/pii-scan.json
```

`config-file` is a [`ValidationConfig`](https://github.com/busbar-agency/busbar-extensions/blob/main/crates/sf-datasets/src/pipeline/validator.rs) **JSON** document (the binary parses it with `serde_json`) — toggles, custom patterns, skip lists, and entropy thresholds. The action's `severity-threshold` overrides the config's `min_severity`.

## Observability

The binary routes its action outputs, job-summary table, annotation, and SARIF/JSON reports through the shared `github-actions-ux` reporter, so results land in the Job Summary and as workflow annotations automatically. The fail-on breach is surfaced as an `::error` annotation and a non-zero exit; sub-threshold findings as a `::warning`; a clean scan as a `::notice`.
