# PRD: Tidemark

**Status:** design phase. Nothing described here has been built or
measured. Targets are design inputs to validate by benchmark.

## Problem

Every system needs to answer "is it working?" and, later, "what was it
doing last month?" Running Prometheus, Loki, Grafana, and Alertmanager
is the standard answer, and it hides how each piece works. Small
home-grown versions usually fail in the same ways: storage grows without
bound, old data is silently wrong after being compacted, retries
duplicate or drop data, alerts stop working exactly when the monitored
system dies, and nobody notices when the monitor itself is down.

Tidemark is a from-scratch answer to a narrower question than "replace
Prometheus": what does a small monitoring platform look like when those
failure modes are designed against on purpose? Specifically:

- history is affordable because it is aggregated as it ages, and the
  aggregation is provably consistent
- ingestion is safe to retry and never loses an acknowledged sample
- alerts fire on absence and failure, not only on bad values
- the platform monitors itself and is monitored from outside
- every result states the resolution that produced it

It is also the final project of a series. It is meant to be pointed at
the earlier projects and used to watch them.

## Users

Portfolio and reference project, not a product with real users.

- **Me**, as the artifact for backend, observability, and
  database-design interviews, and as the live demo that closes the
  series.
- **A reviewer** judging whether the downsampling, delivery, and
  alerting claims hold up and whether limits are stated honestly.

Actors within the system:

- **Producer:** a service or the agent, pushing metrics and logs.
- **Operator:** queries data, manages rules, silences, receivers,
  dashboards, and keys.
- **Receiver:** a webhook endpoint (chat, Dispatcher, canary).
- **Canary:** an external process that watches Tidemark's heartbeat.

## Scope

**In scope** (mechanism for each is in `docs/ARCHITECTURE.md`):

1. Push ingestion of metrics with a strict retry contract.
2. Series identity, label index, cardinality caps.
3. Partitioned raw storage with partition-drop retention.
4. Hierarchical downsampling (5 m, 1 h) with counter-aware summaries,
   late-data recomputation, and an on-read tail.
5. A drop gate: raw data is never dropped before it is rolled up.
6. TQL query language with tier selection, degraded-resolution metadata,
   limits, and `explain`.
7. Log ingestion, stream index, tokenized GIN search with
   index-then-verify, sealed-partition indexing, tail, redaction.
8. Alert rules with `for`, `keep_firing_for`, no-data policies, and a
   pure state machine; backtesting.
9. Notification outbox, grouping, repeats, silences, signed delivery.
10. Agent: scrape, synthesized `up`, spool, log tail.
11. Meta-monitoring: self-scrape, Watchdog, canary, self-alerts.
12. Scheduled reconciliation.
13. Embedded dashboards and log search UI.
14. Instrumenting the earlier portfolio projects and a provisioned
    "Portfolio Health" dashboard.

**Explicitly out of scope:**

- Multi-tenancy and fine-grained RBAC.
- Distributed tracing.
- Prometheus protocol and PromQL compatibility (an optional
  Prometheus-shaped read API is an extra).
- Recording rules, subqueries, `topk`, vector matching modifiers.
- Email and paging integrations.
- High availability and clustering.
- Long-term object storage.

## Functional requirements

| ID | Requirement |
|---|---|
| FR-1 | A producer can push batches of samples with metadata and receive a response stating accepted, duplicate, and rejected counts by reason. |
| FR-2 | Re-sending any batch never creates duplicate rows. |
| FR-3 | A 200 response means all accepted samples are committed. |
| FR-4 | Malformed batches are rejected with 400; backpressure returns 429 with `Retry-After`; storage outage returns 503. |
| FR-5 | Samples too far in the future or older than raw retention are rejected with a stated reason. |
| FR-6 | New series beyond the configured cap are rejected with `series_limit`. |
| FR-7 | Raw data is kept 48 h, 5 m summaries 30 d, 1 h summaries 400 d (configurable). |
| FR-8 | Counter summaries preserve increase across bucket boundaries and resets. |
| FR-9 | A late sample causes the affected summary buckets to be recomputed. |
| FR-10 | Raw partitions are not dropped until their range is fully summarized. |
| FR-11 | A query returns results with the tier used, effective resolution, and a degraded flag. |
| FR-12 | TQL supports the selectors, functions, aggregations, and operators listed in the architecture. |
| FR-13 | `explain` returns the plan without executing at full cost. |
| FR-14 | Queries enforce series, point, sample, time, and concurrency limits. |
| FR-15 | A producer can push log lines grouped by stream labels; re-sent lines do not duplicate. |
| FR-16 | Logs can be searched by stream selector, substring and regex filters, field filters, and time range, with index acceleration where correct. |
| FR-17 | Logs can be tailed live with bounded delay. |
| FR-18 | Configured redaction patterns are applied before logs are stored. |
| FR-19 | An operator can define alert rules with `for`, `keep_firing_for`, labels, annotations, and a missing-data policy. |
| FR-20 | Alert instances move through the documented states, and every transition is recorded with the rule version and value. |
| FR-21 | A failed rule evaluation never changes instance state and is itself surfaced. |
| FR-22 | Notifications are delivered at least once with a stable id, signed, with retries and a dead state. |
| FR-23 | Silences suppress notifications without suppressing state tracking. |
| FR-24 | A candidate rule can be backtested over a historical range without sending notifications. |
| FR-25 | The agent scrapes Prometheus-format targets, writes `up`, spools during outages, and ships logs with offset checkpoints. |
| FR-26 | The system emits a heartbeat to an external canary, which alarms when it stops. |
| FR-27 | A scheduled reconciliation records every run and its findings. |
| FR-28 | Dashboards are defined as JSON and provisioned from files. |

## Non-functional requirements

All targets are to be **validated by benchmark** before being stated as
results anywhere.

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Metric ingest | 20,000 samples/s sustained, one node |
| NFR-2 | Write batch commit latency | p99 under 100 ms |
| NFR-3 | Instant query, 100 series | p95 under 50 ms |
| NFR-4 | 24 h range, 1 min step, 100 series | p95 under 500 ms |
| NFR-5 | 30 d at 5 min step, and 1 y at 1 h step, 100 series | p95 under 1 s each |
| NFR-6 | Log ingest | 5,000 lines/s peak |
| NFR-7 | Log search, stream and term, 24 h sealed | p95 under 1 s |
| NFR-8 | Evaluator lag, 200 rules at 15 s | p99 under 2 s |
| NFR-9 | Storage reduction at reference deployment | about 50x vs raw-only for one year (estimate) |
| NFR-10 | Durability | zero lost acknowledged samples across the failpoint suite |
| NFR-11 | Consistency | raw, 5 m, and 1 h agree for aligned windows |

## Success criteria

The project is "done" when each is demonstrated, not asserted:

1. **Rollup equivalence:** raw, 5 m, and 1 h tiers agree on aligned
   windows, including counter resets and gaps.
2. **Ingestion safety:** the accounting identity holds under random
   input, retries create no duplicates, and failpoints F1 and F2 lose
   nothing.
3. **Late data:** a late sample changes the relevant summary and
   nothing else.
4. **Drop gate:** with rollups stalled, no raw partition is dropped.
5. **Honest resolution:** a query finer than a tier can serve returns the
   degraded flag.
6. **Alerting:** the `Step` table passes; failed queries leave state
   unchanged; a killed target fires `ServiceDown`.
7. **Delivery:** failpoints F8 and F9 produce one logical notification
   with at-least-once delivery.
8. **Logs:** the indexed search equals the brute-force oracle across
   random corpora.
9. **Meta-monitoring:** stopping the evaluator triggers the canary.
10. **Dogfood demo:** kill and recover a portfolio service and watch the
    full alert lifecycle on the dashboard.
11. **Benchmarks:** NFR-1 to NFR-8 measured on stated hardware and
    recorded in `TESTING.md`.

## Constraints and assumptions

- One Postgres 16 primary; one region; no clustering.
- Reference deployment: 10,000 series, 15 s scrape, 100 log lines/s,
  7 days of logs.
- Sender clocks are roughly correct (NTP); skew beyond 5 minutes into
  the future is rejected.
- Byte-per-row estimates are to be verified with real table sizes.
- The other portfolio projects can be extended to expose `/metrics` and
  JSON logs.

## Risks and open questions

| Risk or question | Mitigation or default |
|---|---|
| Row-per-sample is storage heavy | Short raw retention; columnar chunks listed as upgrade |
| Postgres write throughput below target | Group commit, `unnest` insert, COPY fallback, benchmark early |
| GIN maintenance slows log ingest | Unindexed head partition, seal job, `always` fallback mode |
| Counter semantics bugs in rollups | One definition, equivalence property test |
| Query engine scope creep | Closed v1 feature list, parser fuzzing |
| Alerts fail when data stops | `up`, `absent`, no-data policy, errors are not data |
| Tidemark cannot detect its own death | External canary |
| Cardinality explosion | Caps at creation, admin cardinality endpoint |
| Whole-word operator syntax | Open |
| Log index mode default | Open, decide after benchmark |
| Prometheus-shaped read API | Open, default is an extra |

## Milestones

Aligned with the build phases in `docs/CURSOR_CONTEXT.md`.

1. Schema and partitions
2. Series and write path
3. Rollups
4. TQL and planner
5. Query API and UI
6. Alerting
7. Agent
8. Logs
9. Reconciler, self-monitoring, canary
10. Failpoints, chaos, benchmarks
11. Dogfood the portfolio
12. Extras

Priority if time is short: phases 1 to 4 prove the storage and
downsampling claim, 6 and 7 make it useful, 11 makes it the story.