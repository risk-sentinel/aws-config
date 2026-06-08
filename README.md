# aws-config

Collects **AWS Config** rule-evaluation results as **HDF** (Heimdall Data Format) evidence for the SPARC ATO package.

This repo is the **capture point** for AWS Config evidence. The retrieval logic (and the AWS credentials to run it) live here; downstream, **`sparc-validate` only *includes* the HDF this repo publishes** ([sparc-validate#194](https://github.com/risk-sentinel/sparc-validate/issues/194)) — it never re-fetches.

## How it works

`.github/workflows/fetch-aws-config.yml` runs nightly (+ on demand), assumes an AWS OIDC role, pulls AWS Config findings into HDF, and publishes them.

```
AWS Config ──(hdf fetch aws-config [primary] + SAF cross-check)──▶ HDF ──▶ artifact + 'latest' release asset
                                                                              │
                                                                              ▼
                                                          sparc-validate#194 (include / aggregate)
                                                                              │
                                                                              ▼
                                                          sparc-iac OSCAL → ATO evidence
```

## Retrieval methods

Both MITRE tools convert AWS Config → HDF and are on PATH in `risksentinel/sparc-auditor` v0.1.5.

| Method | Command | Status |
|---|---|---|
| **hdf CLI `fetch aws-config`** | `hdf fetch aws-config --region <region> -o <out>.hdf.json` | **Active — primary** (hdf-libs 3.2.0). Richer **target** attribution + **failed-result enrichment**; emits the `baselines[]/requirements[]` HDF schema. |
| **SAF CLI `aws_config2hdf`** | `saf convert aws_config2hdf -r <region> -o <out>.saf.hdf.json` | **Active — cross-check.** Emits the `profiles[].controls[]` HDF schema; a second source during transition. |

The **hdf CLI is primary** for its target + enrichment; SAF runs in parallel as a cross-check. Validate either with `hdf validate <file>`; inspect with `hdf list <file>` / `hdf query <file> --status failed`.

- hdf CLI: <https://github.com/mitre/hdf-libs/blob/main/hdf-cli/README.md#fetch-aws-config>
- SAF CLI: <https://saf-cli.mitre.org/#aws-config-to-hdf>

## Output

- **Per-run artifacts** `aws-config-<account>-<region>.hdf.json` (hdf-cli) + `…​.saf.hdf.json` (SAF), 90-day retention.
- **Rolling `latest` GitHub release** whose assets are replaced each run — a **stable URL** for sparc-validate to `gh release download latest`.

Account + region are stamped into the filename (from the OIDC role's account id); the hdf-CLI output also carries them in the HDF **target** (inspect with `hdf list`).

## Prerequisites

| Need | Where | Status |
|---|---|---|
| **AWS Config enabled + recording** with rules deployed (Commercial account) | sparc-iac | required — empty fetch otherwise |
| **OIDC role** trusting `repo:risk-sentinel/aws-config:*` with AWS Config read (`config:Describe*`/`Get*`/`List*`) + `sts:GetCallerIdentity`, exposed as the `AWS_ROLE_ARN` secret | sparc-iac | required |
| **`DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`** (authenticated image pulls) | org secrets | present |
| **`hdf` + `saf` CLIs** in the scanner image | container-build-sign | present (`sparc-auditor` v0.1.5; hdf 3.2.0) |

## Downstream note

The primary (hdf-cli) output uses the `baselines[]/requirements[]` HDF schema, vs the SAF / cinc-auditor `profiles[].controls[]` schema. The consumer (`sparc-validate` `failure_export.py`) and the OSCAL converter (`sparc-iac hdf_to_oscal.py`) must ingest the chosen schema — tracked in sparc-validate#194.

## Scope

AWS Commercial only (account `752531709667`); GovCloud deferred, consistent with the rest of the SPARC compliance pipeline.

## References

- Tracking issue: [aws-config#1](https://github.com/risk-sentinel/aws-config/issues/1) · pin: sparc-validate#195
- Consumer: [sparc-validate#194](https://github.com/risk-sentinel/sparc-validate/issues/194)
