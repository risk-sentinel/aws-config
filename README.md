# aws-config

Collects **AWS Config** rule-evaluation results as **HDF** (Heimdall Data Format) evidence for the SPARC ATO package.

This repo **defines the capture workflow + variables** — but holds **no AWS role and no credentials**. It's a GitHub **reusable workflow** (`workflow_call`); the consumer (**`sparc-validate`**) calls it and supplies its **scanner-role** credentials. Definition lives here; execution + creds live in the caller — the same "upstream defines, consumer runs with its role" model as the cis-* profile overlays.

## Why no role here

`.github/workflows/fetch-aws-config.yml` is `on: workflow_call`. When `sparc-validate` invokes it, the OIDC token is minted as **`repo:risk-sentinel/sparc-validate:*`** (the caller), so the **existing `sparc-validate-scanner` role trust matches** — no new role, no aws-config trust, no aws-config credentials. The scanner role already has (or just needs the `enable_aws_config_evidence_for_sparc_validate` flag flipped for) AWS Config read.

## How the caller uses it

```yaml
# in sparc-validate/.github/workflows/validate.yml
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

Per-region artifacts `aws-config-<account>-<region>.hdf.json` (+ `.saf.hdf.json`). The **artifact is the output**: the caller's `aggregate-failures` step collects it and feeds the **sparc-iac OSCAL/ATO pipeline** (which already ingests sparc-validate artifacts). No S3 upload here (the scanner image has no `aws` CLI; S3 archival, if wanted, is a separate publish job).

## Variables (`workflow_call` inputs)

| Input | Default | Purpose |
|---|---|---|
| `regions` | `["us-east-1"]` | JSON array of regions to capture |
| `output_prefix` | `aws-config` | artifact filename prefix |
| `workload_name` | `sparc-aws-config` | stamped into the HDF passthrough |

## Prerequisites

| Need | Where | Status |
|---|---|---|
| **AWS Config enabled + recording** (NIST 800-53 r5 conformance pack) | sparc-iac `AWS/config/` | **live in prod** (`enable_aws_config`, `enable_conformance_pack`) |
| **Scanner-role AWS Config read** | sparc-iac `enable_aws_config_evidence_for_sparc_validate` | **flip to `true`** (policy already written; default off) |
| **`DOCKERHUB_*`** (authenticated image pulls) | org secrets | present |
| **`hdf` + `saf` CLIs** | `sparc-auditor` v0.1.5 | present (hdf 3.2.0) |

## Scope

AWS Commercial only (account `752531709667`); GovCloud deferred.

## References

- Tracking: [aws-config#1](https://github.com/risk-sentinel/aws-config/issues/1) · pin: sparc-validate#195
- Consumer: [sparc-validate#194](https://github.com/risk-sentinel/sparc-validate/issues/194)
