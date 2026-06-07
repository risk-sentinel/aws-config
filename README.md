# aws-config

Collects **AWS Config** rule-evaluation results as **HDF** (Heimdall Data Format) evidence for the SPARC ATO package.

This repo is the **capture point** for AWS Config evidence. The retrieval logic (and the AWS credentials to run it) live here; downstream, **`sparc-validate` only *includes* the HDF this repo publishes** ([sparc-validate#194](https://github.com/risk-sentinel/sparc-validate/issues/194)) — it never re-fetches.

## How it works

`.github/workflows/fetch-aws-config.yml` runs nightly (+ on demand), assumes an AWS OIDC role, pulls AWS Config findings into HDF, and publishes them.

```
AWS Config  ──(SAF aws_config2hdf)──▶  HDF  ──▶  artifact + 'latest' release asset
                                                      │
                                                      ▼
                                        sparc-validate#194 (include / aggregate)
                                                      │
                                                      ▼
                                        sparc-iac OSCAL → ATO evidence
```

## Retrieval methods

Both MITRE tools convert AWS Config → HDF. Both belong here (the choice is ours, not the consumer's).

| Method | Command | Status |
|---|---|---|
| **SAF CLI `aws_config2hdf`** | `saf convert aws_config2hdf -r <region> -o <out>.hdf.json` | **Active** — baked into `risksentinel/sparc-auditor`; emits standard `profiles[].controls[]` HDF the failure-export / OSCAL pipeline already ingests. |
| **hdf CLI `fetch aws-config`** | `hdf fetch aws-config --region <region> -o <out>.hdf.json` | **Long-term primary** — richer **target** attribution + **failed-result enrichment**. Stubbed in the workflow; **blocked** on the `hdf` Go binary being added to the signed image (tracked in `risk-sentinel/container-build-sign`). |

We standardize on the **hdf CLI** long-term for its target + enrichment; the SAF path is the buildable baseline today and stays as a cross-check during transition.

- SAF CLI: <https://saf-cli.mitre.org/#aws-config-to-hdf>
- hdf CLI: <https://github.com/mitre/hdf-libs/blob/main/hdf-cli/README.md#fetch-aws-config>

## Output

- **Per-run artifact** `aws-config-<account>-<region>.hdf.json` (90-day retention).
- **Rolling `latest` GitHub release** whose assets are replaced each run — a **stable URL** for sparc-validate to `gh release download latest`.

Account + region are stamped into the filename (provenance) from the OIDC role's account id; the hdf-CLI path will additionally stamp them into the HDF target (verifiable with `hdf list targets` / `hdf info`).

## Prerequisites

| Need | Where | Status |
|---|---|---|
| **AWS Config enabled + recording** with rules deployed (Commercial account) | sparc-iac | required — empty fetch otherwise |
| **OIDC role** trusting `repo:risk-sentinel/aws-config:*` with AWS Config read (`config:Describe*`/`Get*`/`List*`) + `sts:GetCallerIdentity`, exposed as the `AWS_ROLE_ARN` secret | sparc-iac | required |
| **`DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`** (authenticated image pulls) | org secrets | present |
| **`hdf` binary** in the scanner image (for the long-term primary) | container-build-sign | pending |

## Scope

AWS Commercial only (account `752531709667`); GovCloud deferred, consistent with the rest of the SPARC compliance pipeline.

## References

- Tracking issue: [aws-config#1](https://github.com/risk-sentinel/aws-config/issues/1)
- Consumer: [sparc-validate#194](https://github.com/risk-sentinel/sparc-validate/issues/194)
