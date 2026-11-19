# Planned package layout and import boundaries

This is the planned layout for the Python package. No code exists yet.

```text
src/jarritos/
  ports/           # Protocols jarritos owns (no vendor imports)
  types/           # Dataclasses / enums returned across ports
  validate/        # Config + contract policy + source tiers
  client/          # Read API client (no Streamlit / boto3 in core paths)
  ui/              # Presentation pages (may import streamlit)
  cli.py           # click entrypoint (validate, init, ui, …)
  adapters/
    fake/          # In-memory implementations for tests / local
    soda/          # Only package allowed to import soda-core
    diff/          # Only package allowed to import reladiff
    sql/           # Only package allowed to import sqlglot
    query/         # QueryRunner implementations
    store/         # SQLite / Data API
    detectors/     # numpy/scipy if used
    sinks/         # CloudWatch, Slack, …
    agents/        # LangGraph / Bedrock
    ui/            # UiFrontend host (launches Streamlit)
    aws/           # boto3 wrappers
```

## Planned rules

1. Only adapter (or `jarritos.ui` presentation) packages import their vendor library.
2. Ports take and return jarritos types; no library objects cross a port.
3. Each port reports `capabilities()` for `jarritos validate`.
4. CI will run import-linter and fail on violations.

The import-linter contracts will live in `pyproject.toml` under `[tool.importlinter]`.
