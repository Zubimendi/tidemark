# CURSOR_CONTEXT: Tidemark

Read this first, then `docs/ARCHITECTURE.md` in full before writing any
code. This is a build spec.

## Status

Design only. No implementation code exists. Everything below is work to
be written. Do not claim anything works until it has been run. Where
tests are listed, they must pass before a phase is marked done.

## Working agreement

- When modifying an existing file, show only the changed blocks with
  exact placement instructions (file, function, before/after anchor),
  not full-file rewrites.
- All time-dependent logic takes an injected clock. No bare `time.Now()`
  in code under test.
- A 200 from the write path means the data is committed.
- Query errors are never treated as "condition false".
- No user-supplied regex runs in Postgres. Use RE2 in Go.
- UI renders untrusted text with `textContent` only.
- Do not change a decision under "Decisions not to relitigate" without
  raising it first.

## Stack

Go 1.22+, PostgreSQL 16 (needs `date_bin`, `DETACH PARTITION
CONCURRENTLY`), Docker Compose. Suggested libraries: `pgx/v5`,
`prometheus/common/expfmt` (text parsing), `prometheus/client_golang`
(self metrics), `cespare/xxhash` is NOT used for series identity
(SHA-256 truncated), `fsnotify` (log tail), `testcontainers-go`
(optional), stdlib `net/http` with Go 1.22 method patterns. Frontend:
no framework, no build step; vanilla JS plus uPlot, embedded with
`go:embed`.

## Repo layout

```
cmd/tidemark/        server (ROLE: ingest,query,rollup,retention,eval,notify,reconcile,all)
cmd/agent/           scraper, log tailer, spool, shipper
cmd/canary/          dead man's switch receiver
cmd/load/            synthetic metrics and log load generator
internal/series/     identity, hashing, cache, cardinality caps
internal/ingest/     protocol, batcher, accounting, backpressure
internal/store/      SampleStore interface, pgx and in-memory impls
internal/rollup/     SQL functions, watermarks, dirty buckets, tail view
internal/retention/  partition manager, registry, drop gate
internal/tql/        lexer, parser, types, planner, evaluator, tier selection
internal/logs/       tokenizer, ingest, stream index, query, seal job, tail, redaction
internal/alert/      rules, Step function, evaluator, instances, silences
internal/notify/     outbox worker, receivers, signing, SSRF guard, templates
internal/reconcile/  checks and runs
internal/api/        HTTP handlers, auth, limits
internal/web/        embedded UI
internal/obs/        self metrics, logging
migrations/          0001_core.sql, 0002_logs.sql, 0003_alerting.sql
provisioning/        dashboards/, rules/, receivers/
docs/ postman/ docker-compose.yml
```

## Build order (each phase has an acceptance gate)

**Phase 1: schema and partitions.** `series`, `samples_raw`,
`partition_registry`, partition manager (create ahead, registry),
append-only triggers, roles (writer, reader, retention). Gate: DB tests
for partition routing, PK idempotency, read-only role cannot write,
append-only tables reject UPDATE/DELETE.

**Phase 2: series resolution and write path.** Canonical hashing, LRU
cache, batch upsert, collision check, cardinality caps, protocol
handler, batcher with group commit, accounting identity, poison
bisection, 429 backpressure, time rules. Gate: retry and duplicate
tests, concurrent series creation converges, accounting identity holds
under random inputs, first ingest benchmark recorded.

**Phase 3: rollups.** `rollup_5m(from, to)` SQL, persisting job with
atomic watermark, `rollup_dirty`, hierarchical 1 h, tail view, drop gate
in the partition manager. Gate: **rollup equivalence property** (raw =
5 m = 1 h for aligned windows, including resets and gaps), late data
recompute, no gap or double count at the watermark under concurrent
rollup, stalled rollups never lose raw data.

**Phase 4: TQL and planner.** Lexer, parser, type checker, evaluator,
`increase`/`rate` per ARCHITECTURE section 5, `selectTier`, degraded
metadata, limits, `explain`. Gate: golden files, parser fuzz (no panic,
no hang), tier-equivalence tests, limit enforcement.

**Phase 5: query API and UI.** Instant and range endpoints, series and
label endpoints, embedded UI (charts, query box, dashboards from JSON,
stat and table panels, macros `$__interval`, `$__range`), dashboard
provisioning. Gate: a dashboard renders from a provisioned file; XSS
test with hostile label values.

**Phase 6: alerting.** Pure `Step`, aligned evaluator, instance and
event tables, CAS, outbox, grouping, repeats, silences, receivers,
signing, SSRF guard, templates, Watchdog, backtest endpoint. Gate:
exhaustive `Step` table test, all failpoints F6 to F9 converge, errors
leave state untouched.

**Phase 7: agent.** Scrape and parse, synthesized `up` and scrape
series, bounded queue, disk spool, backoff following the response
contract. Gate: server outage and restart lose nothing within spool
limits; permanent rejects are not retried.

**Phase 8: logs.** Tokenizer, ingest with `entry_id` dedupe, streams,
query parser, index-then-verify planner with the edge-token rule, seal
job, tail with lag, redaction, log-derived `count_over_time`. Gate:
**oracle property test** (indexed result equals brute-force result),
sealed and head partitions return identical results, redaction cases.

**Phase 9: reconciler, self-monitoring, canary.** The seven checks in
ARCHITECTURE section 17, self-scrape, canary binary, self-alert rules.
Gate: each injected drift detected; stopping the evaluator triggers the
canary.

**Phase 10: failpoints, chaos, benchmarks.** Failpoint harness (named
crash points via env var), F1 to F14, load generator, benchmark report.
Gate: zero lost acknowledged samples; benchmarks recorded with hardware.

**Phase 11: dogfood the portfolio.** Add `/metrics` and JSON logging to
projects that lack them (checklist below), provision the "Portfolio
Health" dashboard and rules, write the demo script. Gate: the kill and
recover demo works end to end.

**Phase 12: extras** (below).

## Portfolio instrumentation checklist

For each project: `/metrics` in Prometheus text format, `service` and
`env` labels, JSON stdout logs with `level`, `msg`, `correlation_id`
where one exists. Track here: Tessera [ ], Counterweight [ ],
Dispatcher [ ], LedgerLine [ ], VaultKeep [ ], others [ ]. Metric names
used in provisioned dashboards and rules are illustrative until each
project actually exposes them.

## API surface

- `POST /api/v1/write`, `POST /api/v1/logs/write`
- `GET /api/v1/query`, `GET /api/v1/query_range` (+ `explain=true`),
  `GET /api/v1/series`, `GET /api/v1/labels`,
  `GET /api/v1/label/{name}/values`
- `GET /api/v1/logs/query_range`, `GET /api/v1/logs/tail` (SSE)
- `GET/POST /v1/rules`, `POST /v1/admin/rules/backtest`,
  `GET /v1/alerts`, `GET /v1/alerts/{id}/events`
- `GET/POST /v1/silences`, `GET/POST /v1/receivers`
- `GET/POST /v1/dashboards`
- Admin: `GET /v1/admin/cardinality`, `GET /v1/admin/reconciliation`,
  `POST /v1/admin/keys`
- `GET /healthz`, `GET /metrics`

## Tests that must exist (and be run)

1. Rollup equivalence: raw, 5 m, 1 h agree for aligned windows.
2. Ingest idempotency and accounting identity under random input.
3. Concurrent series creation converges to one row.
4. Late data recompute; watermark boundary consistency under concurrent
   rollup.
5. Drop gate: stalled rollups never lose raw data.
6. TQL goldens; parser fuzz.
7. Degraded-resolution flag when requested step is finer than the tier.
8. Alert `Step` exhaustive table with fake clock; errors do not change
   state; no-data policies; `for` and `keep_firing_for`.
9. Duplicate evaluators produce one notification (F9).
10. Notification redelivery is idempotent (F8).
11. Log oracle property test (no false negatives).
12. Edge-token rule cases.
13. Redaction and UI XSS tests.
14. Failpoints F1 to F14.
15. Reconciler detects each injected drift.
16. Canary fires when the evaluator stops.

## Decisions not to relitigate

- Postgres only, no TimescaleDB; `SampleStore` interface stays.
- Row-per-sample raw tier; tiers raw 48 h, 5 m for 30 d, 1 h for 400 d.
- Buckets store in-bucket `inc` only; boundary deltas computed at query
  time; one SQL definition shared by persistence and tail view.
- `increase`/`rate` defined as in ARCHITECTURE section 5, not
  Prometheus extrapolation.
- Raw partitions dropped only behind the rollup watermark.
- SHA-256 (128-bit) series identity; regex in Go.
- Idempotent at-least-once ingestion; 200 means committed.
- Errors are not data in alerting; `up` is synthesized by the agent.
- Notification outbox with deterministic ids.
- Tokenizer shared by writer and query planner; the index may add false
  positives, never false negatives.
- Watchdog plus external canary.

## Open decisions to confirm with the owner

- Whole-word operator syntax (`|=w` is a placeholder).
- Log index mode default (`sealed` vs `always`) after benchmarking.
- Whether to ship the Prometheus-shaped read API in v1 (default: extra).
- Whether to keep the name TQL or call it a PromQL subset.

## Extras (optional, do after Phase 11)

### A. Postman collection

Create `postman/Tidemark.postman_collection.json` (Collection v2.1) and
`postman/Tidemark.local.postman_environment.json`.

- Variables: `baseUrl`, `writeKey`, `readKey`, `adminKey`, `ruleId`,
  `silenceId`, `receiverId`.
- Folders: Ingest (metrics JSON, logs JSON), Query (instant, range,
  explain), Logs, Rules and Alerts (create, backtest, list, events),
  Silences, Receivers, Dashboards, Admin, Health.
- Pre-request script generates sample timestamps (`Date.now()`), so
  ingestion requests are always in the accepted window.
- Test scripts assert status and store ids with `pm.environment.set`.
- Negative cases: malformed body (400), future timestamp (reported in
  `rejected`), over-limit labels, invalid TQL (error message), read key
  used on write (403).
- The SSE tail endpoint is not Postman-friendly; document `curl -N`
  instead. Verify current Postman behavior before promising otherwise.
- Validate by importing into Postman before committing.

### B. Medium thumbnail

- Sizes: about 1400x788 (16:9) for Medium and 1200x627 for LinkedIn.
  Verify current platform recommendations.
- Concept: dark background; a line chart where the right side (recent)
  is crisp and detailed and the left side (old) becomes progressively
  blocky stair-steps, like data losing resolution with age; a thin
  horizontal "tide line" across the chart; a small bell with a red dot
  at the top right. Title "Tidemark", subtitle "Watching everything I
  built". Large text, key content in the central safe area, no logos.
- Implementation: SVG or HTML exported to PNG with a headless browser,
  or an image-generation prompt from this concept.

### C. Tags

Medium (max 5): `Observability`, `PostgreSQL`, `Golang`,
`System Design`, `Monitoring`.

LinkedIn: `#observability` `#golang` `#postgresql` `#systemdesign`
`#backendengineering` `#monitoring` `#devops` `#softwareengineering`
`#buildinpublic`

## Priority order if time is short

1. Phases 1 to 4: storage, downsampling with the equivalence test, TQL.
2. Phase 6 and 7: alerting and the agent.
3. Phase 11: the dogfood demo, since it is the story.
4. Everything else.