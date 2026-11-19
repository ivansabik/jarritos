# License policy

Jarritos is planned to depend only on free, self-hostable software. Once code exists, CI will check declared and installed dependency licenses against the allowlist below.

## Allowlist

| License | Scope |
|---------|--------|
| MIT | Default for application deps (reladiff, LangGraph, LangChain, sqlglot, …) |
| Apache-2.0 | Allowed |
| BSD (2-Clause / 3-Clause) | Allowed |
| ELv2 | **Soda Core public packages only** (`soda-*` free packages). Accepted for self-hosted use. |
| MPL-2.0 | **OpenTofu only** (IaC runtime). Terraform modules must stay OpenTofu-compatible. |

## Excluded

- Soda Cloud, Soda Cloud Runner, Soda AI, and any package that requires a Soda Cloud account.
- Paid Soda packages: `soda-groupby`, `soda-scopes`, reconciliation checks, `soda-mysql`, `soda-oracle`, failed-row **percentage** thresholds.
- Proprietary / paid SaaS SDKs that cannot run fully self-hosted without a license fee (except optional sinks the operator already licenses, e.g. Datadog).
- HashiCorp Terraform Business Source License as a **required** runtime — modules are written for OpenTofu (MPL-2.0); Terraform may still plan them, but CI prefers OpenTofu.

## ELv2 note

Jarritos is self-hosted in each user's own AWS accounts. Offering jarritos to third parties as a hosted multi-tenant service that embeds Soda Core would breach ELv2. Jarritos never patches Soda Core or bypasses its license-key checks.

## Planned CI license check

A script in CI will run `pip-licenses` against the project environment and compare the result with the allowlist above. If optional extras (soda, reladiff) are not installed, it will still check the declared core and dev dependencies. A full audit of the Soda image licenses is an M0 spike item.

## Exit plans (per major dependency)

Each adapter port is planned to have a swap path. Triggers to start a replacement: license change, no release for ~1 year, or an unpatched security issue.

### Soda Core (`ContractEngine`)

- **Risk:** ELv2 / free boundary can move; pre-release API churn.
- **Exit:** Keep user contracts as plain free-subset YAML. Implement a native engine on `QueryRunner` for `schema`, `row_count`, `freshness`, `missing`, `invalid`, `duplicate`, `aggregate`, `metric`, `failed_rows`.
- **Size:** medium (weeks), scoped to free checks only.

### reladiff (`DiffEngine`)

- **Risk:** Small maintainer base; incomplete drivers.
- **Exit:** Athena join diff + bucketed checksum diff (planned for M3 anyway) become the fallback core.
- **Size:** mostly covered by the M3 work; finishing cross-engine is medium.

### sqlglot (`SqlBuilder`)

- **Risk:** Dialect gaps / churn.
- **Exit:** Small set of hand-written templates for counts, nulls, percentiles, metadata.
- **Size:** small.

### LangGraph / LangChain (`AgentRuntime`, `ChatModel`)

- **Risk:** Frequent API changes.
- **Exit:** Keep typed state and plain node and tool functions so the framework can be replaced with a tiny state-machine runner. Prompts stay as files.
- **Size:** small–medium.

### Bedrock / chat providers (`ChatModel`)

- **Risk:** Provider quirks on tools / structured output.
- **Exit:** Second implementation is any OpenAI-compatible endpoint; conformance tests cover structured output.
- **Size:** small.

### Streamlit (`UiFrontend`)

- **Risk:** Rerun model / hosting limits.
- **Exit:** Pages are presentation-only over the read API client; replace with another front end without platform changes.
- **Size:** UI rewrite only.

### Aurora Data API (`Store`)

- **Risk:** 1 MiB response cap; writer-only reads.
- **Exit:** SQLite for local use; later direct Postgres via RDS Proxy behind the same `Store` port.
- **Size:** medium (connection pooling; migrations are versioned).

### numpy / scipy (`Detector`)

- **Risk:** Heavy deps for simple stats.
- **Exit:** Detectors are planned as plain functions; MVP math can stay stdlib if needed.
- **Size:** small.

### AWS SDK (`Queue`, `ObjectStorage`, `Secrets`, `Scheduler`, `Notifications`)

- **Risk:** API / IAM churn.
- **Exit:** Ports are kept thin, and in-memory fakes will cover local tests. Swap SDK major versions inside `adapters.aws` only.
- **Size:** small.
