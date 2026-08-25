# aws-config

Collects **AWS Config** rule-evaluation results as **HDF** (Heimdall Data Format) evidence for an ATO package.

This repo **defines the capture workflow + variables** — but holds **no AWS role and no credentials**. It's a GitHub **reusable workflow** (`workflow_call`); the consuming repository calls it and supplies its own **scanner-role** credentials. Definition lives here; execution + creds live in the caller — the same "upstream defines, consumer runs with its role" model as the cis-* profile overlays.

## Why no role here

`.github/workflows/fetch-aws-config.yml` is `on: workflow_call`. When the caller invokes it, the OIDC token is minted as **`repo:<your-org>/<caller-repo>:*`** (the caller), so the caller's **existing scanner-role trust matches** — no new role, no trust entry for this repo, no credentials here. The scanner role needs AWS Config read permission.

## How the caller uses it

```yaml
# in <caller-repo>/.github/workflows/validate.yml
jobs:
  aws-config:
    uses: risk-sentinel/aws-config/.github/workflows/fetch-aws-config.yml@v0.1.0
    with:
      regions: '["us-east-1"]'
      workload_name: sparc-aws-config
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}          # the scanner role
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

## What it does (per region)

1. Assume the caller's role (OIDC) → AWS_* env creds (both `hdf` and `saf` read them via the standard chain — no `-a/-s/-t` needed).
2. **`hdf fetch aws-config`** (primary — target + failed-result enrichment, `baselines[]/requirements[]` schema).
3. **`saf convert aws_config2hdf`** (cross-check — `profiles[].controls[]` schema).
4. **`saf supplement passthrough write`** stamps the **workload + target** (account / region / timestamp / run URL) into each HDF's `passthrough`, so the evidence self-identifies.
5. `hdf validate` each output; **upload as artifacts**.

## Output

Per-region artifact **`aws-config-<account>-<region>.hdf.json`** — the hdf-cli fetch output **normalized to the legacy `profiles[].controls[]` HDF schema** via `hdf convert --to hdf@1`, so it drops straight into the existing `failure_export` / `hdf_to_oscal.py` pipeline with **no custom shape-handling**. Plus the SAF cross-check `.saf.hdf.json` (already that schema). The richer hdf-cli `baselines[]/requirements[]` form is kept locally as `.raw.json` (not uploaded) pending the fully-native-OSCAL migration.

The **artifact is the output**: the caller's `aggregate-failures` step collects it and feeds the **downstream OSCAL/ATO pipeline** (which already ingests artifacts of this shape). No S3 upload here (the scanner image has no `aws` CLI).

## Variables (`workflow_call` inputs)

| Input | Default | Purpose |
|---|---|---|
| `regions` | `["us-east-1"]` | JSON array of regions to capture |
| `output_prefix` | `aws-config` | artifact filename prefix |
| `workload_name` | `sparc-aws-config` | stamped into the HDF passthrough |

## Prerequisites

| Need | Where | Status |
|---|---|---|
| **AWS Config enabled + recording** (NIST 800-53 r5 conformance pack) | your IaC repo, e.g. `AWS/config/` | enable the recorder + conformance pack |
| **Scanner-role AWS Config read** | your IaC repo | grant AWS Config read to the scanner role |
| **`DOCKERHUB_*`** (authenticated image pulls) | org secrets | present |
| **`hdf` + `saf` CLIs** | `sparc-auditor` v0.1.5 | present (hdf 3.2.0) |

## Scope

AWS Commercial only (account `752531709667`); GovCloud deferred.

## References

- Tracking: [aws-config#1](https://github.com/risk-sentinel/aws-config/issues/1) · pin: sparc-validate#195
- Consumer: [sparc-validate#194](https://github.com/risk-sentinel/sparc-validate/issues/194)
