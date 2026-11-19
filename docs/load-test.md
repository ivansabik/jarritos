# Load test

Plan for a synthetic harness for the ~5,000-dataset scale assumption. The harness is not written yet. It is planned to run locally without AWS first, then be pointed at a real home/runner deploy at the end of M2.

**Do not put the 5,000-dataset scale claim in the README until this load test has passed on AWS.**

## What it will cover

| Piece | Local harness (planned) | AWS (end of M2) |
| --- | --- | --- |
| Config size | Generator builds namespaces, runners (two accounts), datasources, monitors, and ~N datasets as a compact manifest | Deploy the same shape via `jarritos deploy` into two accounts |
| Schedule fan-out | Groups datasets by `(namespace, cron)` — same unit as EventBridge → dispatcher | Real dispatcher Lambda + SQS |
| Jitter | Deterministic per-dataset jitter so jobs do not start on the minute | Same idea on enqueue |
| Lease contention | Per-datasource `max_concurrent` pool; misses requeue with delay | DynamoDB leases on the runner |
| Store / Lambda / Aurora | Estimated metric-point volume only | Measure for real (see below) |

## Planned layout

```text
tools/loadtest/          # standalone harness, not required by the jarritos package
  generate.py            # synthesize a compact manifest
  simulate.py            # local dispatcher-like simulation
  __main__.py            # CLI
tests/tools_loadtest/    # pytest, small N in CI
```

## Planned local workflow

- `generate` builds a compact manifest for a given dataset count (5,000 by default, with placeholder account IDs).
- `fanout` prints the schedule grouping without running the event loop.
- `simulate` runs a number of hours of dispatch, jitter and lease behavior, either from a manifest or in memory, and can emit JSON.
- CI would use a small dataset count (about 50). A full 5,000-dataset run would be opt-in.

### Planned manifest outputs

`generate` would write:

- `manifest.yaml` and `manifest.json` with compact `dataset_groups` rather than 5,000 separate files.
- `sample_config.yaml` with runners, namespaces, datasources and a few sample dataset docs for inspection.

Placeholder `account_id` values and secret ARNs must be replaced before any AWS deploy.

## What to measure on AWS (end of M2)

After home + runner modules are deployed to two accounts and the synthetic (or equivalent) manifest is deployed, exercise hourly schedules and record:

1. **SQS queue age / depth** — job queue and results queue; watch for sustained growth across ticks.
2. **Errored-run rate** — `errored` outcomes vs completed runs (platform alarm target).
3. **Store insert rate** — Ingest Lambda → Aurora Data API batch inserts (metric points/sec, throttles, 1 MiB read limits on the read path).
4. **Aurora** — ACU, commit latency, whether Serverless v2 stays warm under the hourly burst (~5k datasets × ~30 metrics ≈ 3.6M points/day at full hourly fire).
5. **Lambda concurrency** — runner reserved concurrency, dispatcher duration, cold starts; confirm bursts do not exhaust the account quota.
6. **Lease contention** — DynamoDB conditional-write failures / requeue rate per datasource vs configured `max_concurrent`.
7. **Missed runs** — datasets with no completed run within a multiple of their schedule (platform health alarm).

Pass criteria for README: the AWS run completes a multi-hour window at the target config size without unbounded queue growth, with errored-run and missed-run rates inside the alarm thresholds you ship in the Terraform modules.

## Pointing at AWS later

The local harness will not call AWS. When M2 exists:

1. Generate the manifest (or export expanded YAML once `jarritos deploy` accepts it).
2. Map runners to real account IDs / queue URLs in Terraform.
3. Deploy home + runner; publish the manifest per namespace.
4. Drive schedules (or invoke the dispatcher) and collect the metrics above.
5. Only then reference the scale target in the README.
