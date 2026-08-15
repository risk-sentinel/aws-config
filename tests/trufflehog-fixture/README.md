# TruffleHog fixture

Synthetic, non-functional credential used ONLY to prove the secret scanner still
fires. If `secret-scan.yml`'s fixture-detection job stops failing here, the
scanner has regressed and the real gate is silently doing nothing.

Nothing in this directory is a real credential. It is excluded from the
verified-secrets gate via `.trufflehog-exclude-paths`.

Copied verbatim from `risk-sentinel/dev-sec-ops-baseline` so both repositories
exercise the same detector on the same input — a fixture invented per repository
would drift, and a fixture that quietly stopped matching would take the gate's
credibility with it.
