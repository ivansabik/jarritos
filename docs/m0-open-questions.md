# M0 open questions (need live AWS / accounts)

Items from the M0 spikes that documentation and local index checks cannot close. Track these into M1–M3 work.

## Needs live warehouses or Soda install

1. **Soda egress:** Does `verify_contract_locally` (or import of `soda-*`) make any network call when Cloud publish is off? Test with deny-all egress.
2. **Measurements fidelity:** Confirm `measurements` / `diagnostic_metric_values` cover every free check type jarritos will store (schema, row_count, freshness, missing, invalid, duplicate, aggregate, metric, failed_rows count).
3. **MariaDB / SAP HANA package freeness:** Still unknown for v4; no public package found. Keep tier Unknown until a real package + license path is confirmed.
4. **Metadata sweep SQL:** The planned draft metadata sweep SQL must be verified for permissions, performance, and column semantics on PostgreSQL, Redshift, Snowflake, and Athena.
5. **Percentile correctness:** Spot-check exact vs approximate values on each Tier A engine with a known fixture.

## Needs AWS account / deployment

6. **Runner image size** (uncompressed + ECR) with the final driver set.
7. **Lambda cold start** p50/p95 for Runner (with and without VPC ENI attachment) and home-account Lambdas.
8. **Reladiff on Lambda:** time-to-complete for representative windowed L3 jobs; where soft deadline should sit below 15 minutes.
9. **Fargate task start time** for the runner image (with/without SOCI).
10. **Data API ingest:** rows/sec and safe `parameterSets` count for metric_points inserts at chosen ACU; behavior when response approaches 1 MiB.
11. **Aurora pause resume** latency after idle (plan cites ~15–30s) if small installs auto-pause.
12. **Athena same-engine join diff:** scanned bytes, cost, and runtime on a large partitioned table with required partition filters.
13. **Athena concurrency** under per-datasource leases (account quota interaction).
14. **Cross-engine Athena checksum prototype:** feasibility and cost vs contributing an Athena driver upstream.
15. **Terraform + OpenTofu:** `plan`/`apply`/`test` on home and runner modules in two accounts; lockfile differences.

## Needs product / legal confirmation (non-engineering alone)

16. Enterprise legal review of **Soda ELv2** for self-hosted distribution of jarritos.
17. Confirm jarritos never ships private-index Soda packages even if a customer already has keys.
