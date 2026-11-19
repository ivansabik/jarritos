# M0 spikes report

Date: 2026-11-18  
Repo: jarritos  
Scope: research and local readonly checks only. No full AWS deployment. Measured numbers that require AWS, live warehouses, or private Soda credentials are labeled **not yet measured**.

Planned helper scripts for the M0 spikes (not yet in the repo, no AWS required):

- Soda package check: public index lookups for free vs missing Soda packages.
- Percentile SQL printer: portable median/percentile SQL per dialect.
- Data API batch budget: estimate how many rows fit under the Data API 4 MiB / 1 MiB limits.
- Metadata sweep SQL: draft system-catalog queries for Tier A sources, unverified against live warehouses.

Related: [m0-open-questions.md](./m0-open-questions.md)

---

## 1. Soda Core free-package audit

### Install source

Two public install paths exist for free Soda Core v4 packages:

| Source | Docs / evidence | Result of local check (2026-11-18) |
| --- | --- | --- |
| **public PyPI** (`pypi.org`) | [sodadata/soda-core README](https://github.com/sodadata/soda-core): `pip install soda-postgres` | `soda-postgres` / `soda-core` / `soda-athena` / `soda-redshift` / `soda-snowflake` resolve; latest seen **4.31.0** |
| **Soda public index** `https://pypi.cloud.soda.io/simple` | [Soda Python Libraries](https://docs.soda.io/soda-documentation/connect/deployment-options/soda-python-libraries): `uv pip install -i https://pypi.cloud.soda.io/simple --pre -U "soda-postgres>4"` | Index is reachable without credentials; same free packages resolve |

**Recommendation for jarritos:** install free packages from **public PyPI** (simpler, no custom index). Pin versions with hashes and mirror wheels. Keep awareness that Soda’s own docs still advertise `pypi.cloud.soda.io` with `--pre`; as of this check, 4.31.0 is published as a normal release on public PyPI (no `--pre` required for that version).

Do **not** use private indexes (`team.pypi.*`, `enterprise.pypi.*`) or Soda API key credentials.

### Does `verify_contract_locally` need a Soda Cloud account or network call?

From the [Python API](https://docs.soda.io/apis/python-api):

- `verify_contract_locally(...)` runs local verification.
- `publish` defaults to off; `soda_cloud_file_path` is required **only if** `publish=True`.
- Fetching a contract via `dataset_identifier` talks to Soda Cloud; local `contract_file_path` does not.

**Conclusion:** local verify with local datasource YAML + local contract YAML does **not** require a Soda Cloud account. Publishing or fetching cloud contracts does. Jarritos should always pass local file paths and `publish=False` (or omit publish).

**Not yet measured:** whether the free package makes any opportunistic telemetry / version-check HTTP call at import or verify time. Worth a network-trace test in M1 (deny-all egress) before assuming air-gapped friendliness.

### Measurements API

Available on Soda Core local scans per the same Python API docs:

- `ContractVerificationResult.measurements: list[Measurement]`
- `CheckResult.diagnostic_metric_values: Optional[dict[str, float|int|str]]`

These are documented as available for Soda Core (local) results. Jarritos can map them into metric points behind `ContractEngine` without Soda Cloud.

### Free vs private packages (especially mysql / oracle / groupby / scopes / reconciliation)

Soda’s docs split features by install flow ([Soda Python Libraries](https://docs.soda.io/soda-documentation/connect/deployment-options/soda-python-libraries)):

| Package / feature | Public (free) | Private / licensed | Local check |
| --- | --- | --- | --- |
| `soda-core`, `soda-postgres`, `soda-redshift`, `soda-snowflake`, `soda-athena`, `soda-duckdb`, `soda-trino`, `soda-bigquery`, `soda-databricks`, … | Yes | — | Present on `pypi.org` at 4.31.0 |
| `soda-groupby` | No | Team / Enterprise private PyPI | Missing on `pypi.org`; soda public simple index does not serve a usable free listing (redirect/404 without private auth) |
| `soda-scopes` | No | Private PyPI | Same as groupby |
| `soda-reconciliation` | No | Enterprise private PyPI ([reconciliation docs](https://docs.soda.io/soda-documentation/developer-tools/contract-language-reference/reconciliation-checks)) | Same as groupby |
| `soda-mysql` | Docs: private for licensed Python use | Team/Enterprise private PyPI | **Absent** from `pypi.org` (404). `pypi.cloud.soda.io` lists the name, but `pip index` only shows **legacy 1.x**, not a free v4 line |
| `soda-oracle` | Docs: private | Enterprise private PyPI | Same as mysql: not on `pypi.org`; soda index legacy 1.x only |
| MariaDB / SAP HANA | Docs list as supported data sources; package freeness unclear | Unknown | No `soda-mariadb` / `soda-sap-hana` on `pypi.org`; soda public simple index has no usable free listing |

**Jarritos policy implication (unchanged):** Tier C for MySQL/Oracle via reladiff + `QueryRunner`; replace group-by / scopes / reconciliation with jarritos monitors, namespaces, and reladiff. Reject paid-only check types in `jarritos validate`.

---

## 2. QueryRunner options: reladiff connection reuse vs DB-API

### What reladiff offers

From the [reladiff Python API](https://reladiff.readthedocs.io/en/latest/python-api.html) and [PR #62](https://github.com/erezsh/reladiff/pull/62):

- `connect(db_conf, shared=True)` caches connections by config.
- `connect_to_table` accepts an existing `AbstractDatabase`, so one connection can serve preprocess queries and a later diff.
- Drivers wrap databases via sqeleton; supported drivers include postgresql, mysql, oracle, snowflake, bigquery, redshift, presto, databricks, trino, clickhouse, vertica — **not Athena**.
- The wrapper is **not** a raw DB-API connection; it exposes reladiff/sqeleton query helpers.

### Options

| Approach | Pros | Cons |
| --- | --- | --- |
| **A. Reuse reladiff `Database` for QueryRunner** | One connection pool per datasource; fewer auth prompts (esp. Snowflake browser auth); shared driver install set | Couples monitors/metadata to reladiff’s API and dialects; Athena still needs a separate path; harder to swap DiffEngine later |
| **B. Plain DB-API / vendor SDK behind QueryRunner** | Clean port; Athena/PyAthena fits naturally; no reladiff import in the query path | Second connection model beside DiffEngine; duplicate driver config |
| **C. Hybrid (recommended)** | QueryRunner owns a thin dialect+driver adapter; DiffEngine may reuse the same credentials and, when both sides are reladiff-backed, optionally an injected `AbstractDatabase` | Slightly more adapter code |

### Recommendation

Use **C (hybrid)**:

1. `QueryRunner` talks DB-API / official SDKs (psycopg, redshift-connector, snowflake-connector, PyAthena, DuckDB).
2. `DiffEngine` uses reladiff; pass a pre-built reladiff connection when the same job already opened one for L2, to avoid double auth.
3. Do **not** make QueryRunner call into reladiff for ordinary SQL — that would turn DiffEngine’s exit plan into a rewrite of monitors too.

---

## 3. Runner image / Lambda cold start

### Constraints (documented)

- Lambda container images: up to **10 GB** uncompressed ([AWS Lambda container images](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)).
- Cold start is dominated by **init** (load code + runtime + handler init), not full image size, because Lambda loads container data sparsely ([Brooker et al., ATC’23](https://www.usenix.org/system/files/atc23-brooker.pdf); AWS states cold starts often under 100 ms to over 1 s in general guidance — **workload-specific numbers not measured here**).
- Init phase without provisioned concurrency / SnapStart: **10 s** limit before retry quirks ([Lambda lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)).

### Dependency size signals (wheels only, not a built image)

Largest single wheels seen on PyPI during this spike (illustrative, not the final image):

| Artifact | Approx size |
| --- | --- |
| `scipy` manylinux wheel | ~30–38 MB |
| `snowflake-connector-python` manylinux | ~23 MB |
| `numpy` | ~15–21 MB |
| `botocore` sdist / wheels | ~10–17 MB |
| `psycopg2-binary` | ~4–6 MB |
| Soda datasource meta-packages / reladiff | tiny (&lt;0.1 MB); drivers pull the mass |

A **runner** image that includes Soda Core + reladiff + Postgres/Redshift/Snowflake/Athena drivers + numpy/scipy (if detectors share the image) will likely land in the **hundreds of MB uncompressed** range. Hub Lambdas (no warehouse drivers) should stay much smaller.

**Not yet measured on AWS:**

- Final multi-arch image size (ECR digest).
- Cold start p50/p95 for Runner Lambda (VPC vs no VPC; 1–3 GB memory).
- Import-time cost of soda + snowflake + pyathena together.
- Whether SnapStart is usable for the chosen base image / Python runtime features.

Measure in M2 with the real Dockerfile before setting reserved concurrency or timeouts.

---

## 4. Reladiff on Lambda vs Fargate

### Reladiff published benchmarks (vendor claims)

From [reladiff README / PyPI](https://pypi.org/project/reladiff/):

- ~**25M rows in under 10 seconds** and ~**1B rows in about 5 minutes** when there are **no differences** (hash/join work stays inside the DBs).
- Tables with tens of billions of rows claimed as possible.

These are **not** Lambda measurements. They assume sufficient client threads, network, and warehouse capacity.

### When Lambda stops being viable

Hard limits and design triggers:

| Trigger | Why Lambda fails | Plan response |
| --- | --- | --- |
| Wall clock → **15 minutes** | [Lambda max duration](https://docs.aws.amazon.com/decision-guides/latest/fargate-or-lambda/fargate-or-lambda.html) | Soft deadline below 15m → requeue as heavy / Fargate |
| Large full table L3 with many mismatches | Client must download mismatched segments; memory ≤ 10 GB; time grows with diff size | Cap differences; Fargate for `mode: full` |
| High thread count / long-lived connections | Lambda freezes between invokes; connection reuse is within one invocation | Prefer Fargate for multi-hour diffs |
| Athena same-engine join that scans huge partitions | Query time + Athena concurrency, not CPU | Still may fit Lambda if Athena finishes in time; cost/quotas matter more |

**Rule of thumb for M3 (unmeasured):** windowed / incremental L3 on Lambda; full / unbounded / high-diff jobs on Fargate via `HeavyRunner`.

### Fargate cold start

AWS decision guide cites Fargate cold start roughly **35 seconds to 2 minutes**, driven largely by **pulling the full image every task** (no layer cache on the single-tenant instance) ([Fargate vs Lambda](https://docs.aws.amazon.com/decision-guides/latest/fargate-or-lambda/fargate-or-lambda.html), [Fargate pull behavior](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-pull-behavior.html)). SOCI lazy-loading helps most for images **&gt; ~250 MB** ([SOCI blog](https://aws.amazon.com/blogs/containers/under-the-hood-lazy-loading-container-images-with-seekable-oci-and-aws-fargate/)).

**Not yet measured:** jarritos runner image pull time on Fargate in the target region/VPC.

---

## 5. Percentile and median SQL portability

Exact continuous percentile syntax is **not** portable to Athena. Prefer dialect templates in `SqlBuilder` and record `approximation: exact | approximate` on the metric point.

| Engine | Median | Percentile `p` (0–1) | Notes |
| --- | --- | --- | --- |
| **PostgreSQL** | `percentile_cont(0.5) WITHIN GROUP (ORDER BY col)` | `percentile_cont(p) WITHIN GROUP (ORDER BY col)` | Exact continuous ([PG docs](https://www.postgresql.org/docs/current/functions-aggregate.html)). No built-in `MEDIAN()`. |
| **Redshift** | `MEDIAN(col)` or `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY col)` | `PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY col)` | Exact. Also `APPROXIMATE PERCENTILE_DISC(p) WITHIN GROUP (ORDER BY col)` ([docs](https://docs.aws.amazon.com/redshift/latest/dg/r_MEDIAN.html)). Multiple sort-based aggregates in one SELECT must share ORDER BY. |
| **Snowflake** | `MEDIAN(col)` | `PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY col)` | Exact. Approx: `APPROX_PERCENTILE(col, p)` ([docs](https://docs.snowflake.com/en/sql-reference/functions-aggregation)). |
| **Athena** (Presto/Trino engine) | no `MEDIAN` | `approx_percentile(col, p)` | **Approximate only** for efficient aggregates ([Presto/Trino aggregate functions](https://trino.io/docs/current/functions/aggregate.html)). Exact median needs window/`row_number` patterns (expensive). |

A planned M0 helper script will print these templates per dialect.

---

## 6. Data API batch-insert throughput and 1 MiB response limit

Sources: [BatchExecuteStatement API](https://docs.aws.amazon.com/rdsdataservice/latest/APIReference/API_BatchExecuteStatement.html), [Data API limitations](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/data-api.limitations.html), [ASv2 Data API announcement](https://aws.amazon.com/blogs/database/introducing-the-data-api-for-amazon-aurora-serverless-v2-and-amazon-aurora-provisioned-clusters/).

### Documented limits

| Limit | Value | Implication for jarritos |
| --- | --- | --- |
| HTTP request body | **4 MiB** (headers + JSON) | Caps `parameterSets` per `BatchExecuteStatement`. Row count depends on column width. |
| Response size | **1 MiB** | Call is **terminated** if exceeded. Prefer inserts that return little/`RETURNING` sparingly; paginate all reads. |
| Writer-only | Queries run on writer | Reader endpoint cannot offload Data API reads. |
| ASv1 RPS | 1,000 req/s (legacy) | **Does not apply** to Aurora Serverless v2 / provisioned Data API — RPS limited by instance capacity, not a fixed 1k cap. |
| Per-row result | 64 KB (design note in ASv2 blog) | Keep metric/history rows narrow. |

### Throughput

- Batching many parameter sets in one `BatchExecuteStatement` is the intended path for bulk insert.
- **Rows/sec and optimal batch size: not yet measured** on the target ACU setting. A planned batch-budget helper will size batches under 4 MiB before live tests.
- At ~3.6M metric points/day (~42/sec average), even modest batching should be fine if ingest Lambda buffers; bursts need the results queue (already in the plan).

### Read path

Any API/list that can exceed 1 MiB **must** paginate (`LIMIT`/`keyset`). Detectors should read baselines, not full history, as planned.

---

## 7. Athena diff options (M3)

Reladiff has **no Athena driver** ([supported drivers list](https://reladiff.readthedocs.io/en/latest/python-api.html)).

| Option | Effort | Fits same-engine? | Fits cross-engine? | Notes |
| --- | --- | --- | --- | --- |
| **A. Join diff in Athena** (`FULL OUTER JOIN` via sqlglot, `UNLOAD`/CTAS to S3) | Medium | Yes | No | Matches plan for two Athena tables. Cost = data scanned; require partition filters; respect Athena concurrency quotas. |
| **B. Contribute Athena driver to reladiff** | High | Yes (via JoinDiffer once driver exists) | Yes (HashDiffer) | MIT allows contribution. Need: SQL dialect hooks, hashing, type mapping, schema introspect, auth (IAM). Ongoing upstream risk (small maintainer base). |
| **C. Native bucketed checksum diff** | High | Possible | Yes | Reimplements reladiff’s hash-bisect algorithm against Athena SQL + S3. Becomes the DiffEngine exit plan anyway. |

### M3 recommendation

1. **Ship same-engine Athena join diff in M3** (option A) behind `DiffEngine` — unconditional in the plan.
2. **Defer cross-engine Athena** unless a spike prototype of option C (or an accepted upstream Athena driver) lands; keep it **conditional** as the plan states.
3. Prefer **native checksum (C) over waiting on upstream (B)** for cross-engine: jarritos already needs DiffEngine ownership, and Athena’s execution model (async queries, S3 results, no persistent temp tables like OLTP) fits a first-party implementation better than forcing Athena into sqeleton’s assumptions.

**Not yet measured:** join-diff scanned bytes / $ on a large partitioned table; Athena concurrent-query headroom under runner leases.

---

## 8. Terraform modules vs OpenTofu (hub + runner)

Plan: hub module + runner module, CI with both Terraform and OpenTofu.

### Compatibility notes

- Write modules in standard HCL using the `terraform` block for `required_providers` (`hashicorp/aws`, etc.). Both CLIs understand this.
- For dual CLI version constraints, OpenTofu docs recommend **`versions.tf`** (Terraform `required_version`) plus **`versions.tofu`** (OpenTofu constraint / `language` block). OpenTofu ignores `.tf` version files when `.tofu` counterparts exist; Terraform ignores `.tofu` ([OpenTofu language settings](https://opentofu.org/docs/language/settings/)).
- Stick to features available in **both** at the pinned minimum (avoid Terraform-only or OpenTofu-only language features until CI proves them).
- Provider lock files: generate/test with both CLIs; document which lockfile consumers should commit if hashes diverge.
- State and module registries: consumers may use either CLI against the same module source (git/tag); do not assume Terraform Cloud-only APIs in the modules.

### Licensing

- Terraform CLI: **Business Source License (BSL)** — not OSI.
- OpenTofu CLI: **MPL-2.0**.
- Modules authored by jarritos should be licensed under an OSI license (e.g. Apache-2.0) so either CLI can consume them. CI running both keeps the “free software only” story honest for the IaC toolchain choice.

**Not yet measured:** `terraform test` / `tofu test` on the real hub/runner modules (modules do not exist yet).

---

## 9. License audit of intended dependencies

| Dependency | License | OSI? | Jarritos stance |
| --- | --- | --- | --- |
| Soda Core public packages | **Elastic License 2.0 (ELv2)** ([LICENSE](https://github.com/sodadata/soda-core/blob/main/LICENSE)) | No | Allowed per plan for self-hosted use. **Cannot** offer jarritos-as-SaaS that exposes substantial Soda Core functionality to third parties. Do not bypass license-key checks. |
| reladiff | **MIT** | Yes | Allowed |
| LangGraph / LangChain (`langgraph`, `langchain-core`, `langchain-aws`) | **MIT** | Yes | Allowed; no LangSmith/Platform |
| Streamlit | **Apache-2.0** | Yes | Allowed |
| sqlglot | **MIT** | Yes | Allowed |
| numpy | BSD-style (multi) | Yes | Allowed |
| scipy | BSD-style | Yes | Allowed |
| Terraform CLI | **BSL 1.1** | No | Tooling only; modules OpenTofu-compatible; CI runs both |
| OpenTofu CLI | **MPL-2.0** | Yes | Preferred free CLI path |

PyPI metadata cross-check (2026-11-18): `reladiff`, `sqlglot`, `langgraph`, `langchain-core` report MIT; `streamlit` Apache-2.0; `soda-core` reports Proprietary on PyPI while the GitHub LICENSE is ELv2 — **trust the GitHub LICENSE / Soda announcement**, and ensure README legal text matches ELv2.

Planned follow-on: a CI license allowlist and a contract validator that rejects private-index packages and paid-only checks (see [licenses.md](./licenses.md)).

---

## Metadata sweep queries (draft)

M0 lists these as planned work. They will start as **draft templates** printed by a helper script, and none are live-verified:

- PostgreSQL: `information_schema` and `pg_catalog` sweeps for tables, columns, row estimates and last-modified signals.
- Redshift: `svv_table_info` and `svv_columns`.
- Snowflake: `information_schema.tables` and `columns`.
- Athena: Glue Data Catalog metadata and partition listings.

Verification against real PostgreSQL, Redshift, Snowflake, and Athena belongs in M1/M2 integration tests ([open questions](./m0-open-questions.md)).

---

## Summary of recommendations

1. Install free Soda packages from **public PyPI**; never private indexes. Local verify needs **no** Cloud account when using local files + `publish=False`.
2. Build **QueryRunner on DB-API/SDKs**; optionally inject a reladiff connection into DiffEngine for reuse.
3. Expect a **large** runner image; **measure** cold start on AWS before SLOs.
4. Lambda for light/windowed diffs; Fargate when approaching **15 minutes**, large full diffs, or heavy mismatch download.
5. Dialect-specific percentile SQL; Athena uses **approx_percentile** and must be labeled approximate.
6. Size Data API batches under **4 MiB** request / avoid fat **1 MiB** responses; paginate reads; ASv2 has no fixed 1k RPS cap.
7. M3: **Athena join diff** for same-engine; cross-engine **conditional** via native checksum (prefer over waiting on upstream).
8. Dual **`versions.tf` / `versions.tofu`**; CI both Terraform and OpenTofu.
9. Licenses OK for the self-hosted plan; ELv2 and Terraform BSL need README/CI honesty.
