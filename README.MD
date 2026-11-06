# Tidemark

A metrics and log platform in Go and PostgreSQL, built around the
question every running system raises: **how do I know it is working,
and what happens to that knowledge as it gets old?**

**Status: design phase.** This repository currently holds architecture
and build specifications only. See `docs/CURSOR_CONTEXT.md` for the
build plan. The sections below describe what the finished system is
designed to do, not what already runs.

## What makes this different from "a metrics table"

- **History that degrades honestly.** Full detail for 2 days, 5 minute
  summaries for 30 days, hourly summaries for over a year, roughly a 50x
  storage reduction. Summaries are exact for counters, and a query that
  asks for finer detail than exists says so.
- **Safe retries everywhere.** Idempotent at-least-once ingestion; a 200
  means committed; received equals accepted plus duplicates plus
  rejected, always.
- **Alerts that survive silence.** Explicit `up`, `absent()`, no-data
  policies, and query errors that never resolve an alert.
- **A watcher for the watcher.** A heartbeat rule and an external canary
  that raises its own alarm if Tidemark stops.
- **Logs on Postgres.** Tokenized GIN index narrows, exact match
  verifies, never missing a line.
- **Dogfooded.** Built to watch the other projects in this portfolio.

## Designed features

- Remote-write-style push protocol with a strict retry contract
- Partitioned raw storage, hierarchical rollups, partition-drop retention
  gated on rollup progress
- TQL, a small PromQL-style query language with explicit semantics and
  `explain`
- Scrape agent with spool, backoff, and log tailing
- Alert rules with `for`, `keep_firing_for`, silences, grouped signed
  notifications, and backtesting
- Embedded dashboards (provisioned from JSON) and log search
- Scheduled reconciliation of rollups, partitions, and alert state

## Quickstart (planned, not yet working)

Requires Go 1.22+, Docker, Docker Compose.

```bash
cp .env.example .env
make up          # Postgres, migrations
make server      # all roles in one process
make agent       # scrape targets from config/agent.yml
make load        # synthetic metrics and logs
make portfolio   # compose profile: agent pointed at the portfolio
make canary      # dead man's switch (run on a different host in real use)
```

## Documentation

- [`docs/PRD.md`](docs/PRD.md): problem, users, scope
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md): every design decision,
  its mechanism, its trade-off, and where it came from
- [`docs/TESTING.md`](docs/TESTING.md): test plan and demonstrations
- [`docs/STORY.md`](docs/STORY.md): narrative and interview talking points
- [`docs/CURSOR_CONTEXT.md`](docs/CURSOR_CONTEXT.md): build spec and phases

## Repo layout

```
cmd/            tidemark, agent, canary, load
internal/       series, ingest, store, rollup, retention, tql, logs,
                alert, notify, reconcile, api, web, obs
migrations/     schema, partitions, triggers
provisioning/   dashboards, rules, receivers
postman/        collection and environment
docs/
```

## License

MIT, see `LICENSE`.