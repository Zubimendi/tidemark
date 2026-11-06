# Architecture: Tidemark

Same format as the rest of this portfolio: each section names a
principle, the problem it answers for a metrics and log platform, the
concrete mechanism, and the trade-off accepted. Numbers marked "target"
or "assumption" are design inputs to validate by benchmark, not
results. Nothing here has been built yet.

This is the final project, built last on purpose. It watches the
others. Almost every mechanism in it was established earlier in the
portfolio (idempotent writes, append-only history, outbox, state
machines with pure transition functions, reconciliation, constraint
backstops). What is new is the domain: time-series storage that gets
coarser as it ages, a query language, full-text log search on Postgres,
and an alerting engine whose hardest problem is "what if the data
stops?" Section 20 maps each mechanism to its origin.

## System shape

```
 portfolio services                               operators
   /metrics   JSON logs                         browser / API clients
        |                                               |
        v                                               v
 +----------------+  batched, idempotent   +--------------------------------+
 | tidemark-agent |----------------------->| tidemark (one image, ROLE flag) |
 | scrape + tail  |  POST /write /logs     |  ingest | query | rollup       |
 | spool + retry  |                        |  retention | eval | notify     |
 +----------------+                        |  reconcile | embedded UI       |
                                           +---------------+----------------+
                                                           | SQL
                                                           v
                                    +------------------------------------------+
                                    | PostgreSQL 16                            |
                                    |  series, samples_raw (daily partitions)  |
                                    |  rollup_5m, rollup_1h, rollup_state      |
                                    |  streams, logs (hourly partitions)       |
                                    |  rules, instances, events (append-only)  |
                                    |  notifications (outbox), silences        |
                                    +------------------------------------------+

 notify --> webhooks (generic, Slack-style, Dispatcher, canary)
 tidemark-canary (separate host): receives Watchdog heartbeats and
 raises its own alarm when they stop. Tidemark cannot report its own death.
```

Postgres is the only store. Roles (ingest, query, rollup, retention,
eval, notify, reconcile) are goroutine groups in one binary. Singleton
duties take a Postgres advisory lock, so extra replicas are harmless
standbys, the same technique as Counterweight's relay.

## Design goals, non-goals, assumptions

Goals: never lose an acknowledged sample or log line; make retries
always safe; keep a year of history affordable by aggregating as data
ages, without making old answers silently wrong; never let an alert
fail quietly because the data it watches stopped; make every result
say which resolution produced it.

Non-goals: replacing Prometheus, Loki, or Grafana at scale; tracing;
multi-tenancy; high availability of Postgres itself. See the end.

**Reference deployment (assumption):** 10,000 active series scraped
every 15 s (about 670 samples/s), 100 log lines/s across 50 streams, 7
days of logs, 15 evaluation rules per project. Target headroom: 20,000
samples/s and 5,000 log lines/s sustained on one node.

**Storage arithmetic (estimates, verify with `pg_total_relation_size`):**

| Tier | Resolution | Retention | Rows (10k series) | Approx bytes/row (heap + index) | Size |
|---|---|---|---|---|---|
| raw | 15 s | 48 h | 115M | 90 | ~10 GB |
| rollup_5m | 5 min | 30 d | 86M | 130 | ~11 GB |
| rollup_1h | 1 h | 400 d | 96M | 130 | ~12 GB |
| total | | | | | ~33 GB |

The same year held at raw resolution is 10k series times 2.1M samples
times 90 bytes, roughly 1.9 TB. Downsampling is a ~50x reduction, and
that ratio, not any single mechanism, is why "a year of history" is
affordable. Most of the saving comes from dropping raw early and
keeping 1 h rows for the long tail; a 15 s to 1 min rollup alone would
save little, because rollup rows are wider than raw rows.

---

## 1. Why Postgres, and where it stops

**Decision:** one Postgres, no separate time-series database, no
TimescaleDB. Building downsampling and retention by hand is the point
of the project, and an extension that provides them would hide it.

**Where this design stops being appropriate** (stated so the limit is
a decision, not a surprise): above roughly 100k samples/s sustained,
above a few hundred thousand active series, or when query latency over
high-cardinality metrics matters. Row-per-sample storage has no native
compression and write amplification from index maintenance.

**Escape hatch:** the engine talks to storage through a `SampleStore`
interface (`Write`, `SelectSeries`, `ReadRaw`, `ReadRollup`). Tests use
an in-memory implementation, and a different backend would replace the
Postgres one without touching the query engine or alerting.

## 2. Series identity and the label index

**Identity:** a series is a metric name plus a label set. The label set
is canonicalized (labels sorted by name, length-prefixed encoding) and
hashed with SHA-256 truncated to 128 bits: `series.key_hash bytea
UNIQUE`. A cryptographic hash is used because label values come from
senders and a non-cryptographic hash could be attacked with crafted
collisions. On a hash conflict the stored labels are compared, and a
mismatch rejects the series and increments a counter; with 128 bits
this is expected never to happen, but the check is cheap.

**Table:** `series(id bigserial, key_hash, metric, labels jsonb, kind,
first_seen, last_seen)`. `kind` is counter, gauge, or histogram-bucket,
taken from sender metadata, because rollups treat counters differently
(section 5). Indexes: btree on `metric`, GIN (`jsonb_path_ops`) on
`labels` for equality matchers.

**Resolution on the hot path:** each ingest worker holds an LRU cache
`key_hash -> series_id`. A miss goes to Postgres in a batch: `INSERT
... ON CONFLICT (key_hash) DO NOTHING RETURNING id, key_hash`, then a
SELECT for the keys that conflicted. Concurrent ingesters creating the
same series converge on one row.

**Regex matchers run in Go, not Postgres.** Postgres POSIX regex and Go
RE2 differ in syntax and semantics. The series lookup narrows by
metric name and equality matchers in SQL, then applies `=~` and `!~`
with RE2 in Go. A cap (`max_series_per_query`, default 5,000) bounds
the candidate set.

**Cardinality control (the classic way these systems die):**
- max labels per series (30), max name and value length (128 / 1024)
- global active-series cap (default 100,000), enforced at series
  creation; excess samples are rejected with reason `series_limit`
- `last_seen` is coalesced: updated at most once per 5 minutes per
  series from the in-process cache, never per sample, to avoid turning
  every write into two
- an admin endpoint lists top metrics by series count

**No foreign key from samples to series.** Partition drops and bulk
inserts are faster without it. The cost is that integrity is checked by
the reconciler (section 17) instead of the database, and series are
garbage collected only after their last data has aged out of every
tier.

## 3. Ingestion: a remote-write-style protocol

**Shape:** agents and services push batches over HTTP.
`POST /api/v1/write`, optional gzip, JSON body:

```
{ "metadata": [{"metric": "http_requests_total", "kind": "counter"}],
  "series": [{"metric": "http_requests_total",
              "labels": {"service": "api", "code": "200"},
              "samples": [[1730880000123, 1027.0], [1730880015123, 1031.0]]}] }
```

Timestamps are Unix milliseconds. JSON is chosen for debuggability
(curl, Postman); a protobuf encoding is a documented later option.

**Delivery semantics: at-least-once, idempotent.** The raw table's
primary key is `(series_id, ts)`, and inserts use `ON CONFLICT DO
NOTHING`. Re-sending a batch is always safe, so every ambiguous failure
(timeout, crash before 200) is resolved by "send it again".

**Response contract** (senders must follow it, and the agent does):

| Status | Meaning | Sender action |
|---|---|---|
| 200 + body | processed; body lists `accepted`, `duplicates`, `rejected` by reason | done; do not retry rejects |
| 400 | malformed | drop, never retry |
| 401, 403 | auth | stop and alert the operator |
| 413 | too large | split |
| 429 + `Retry-After` | backpressure | retry later |
| 503 | storage unavailable | retry with backoff and jitter |

Permanent rejects (`too_old`, `too_new`, `series_limit`, `invalid`) are
reported inside a 200 so a sender never loops forever on data that
cannot be accepted.

**Accounting identity:** for every batch,
`received = accepted + duplicates + sum(rejected by reason)`. This is
asserted in tests and exported as counters, so "where did my samples
go" always has an answer.

**Acknowledgement means durable.** A 200 is sent only after the batch
commits. Requests are coalesced by a batcher (up to 10,000 samples or
200 ms) so many small requests share one commit (group commit). Each
request waits on a completion handle for its own slice.

**Write statement:** `INSERT INTO samples_raw SELECT * FROM
unnest($1::bigint[], $2::timestamptz[], $3::float8[]) ON CONFLICT DO
NOTHING`. `COPY` into a temp table then insert-select is the documented
fallback if the benchmark shows it wins.

**Poison isolation:** if a coalesced batch fails, the batcher bisects
and retries requests individually so one bad request cannot fail its
neighbours. Ahead of that, timestamps are pre-validated against
existing partition ranges, because "no partition for row" is the most
likely poison.

**Time rules:** reject samples more than 5 minutes in the future
(`too_new`) and samples older than raw retention (`too_old`). Sender
clocks are untrusted; out-of-order samples within the window are fine
because storage is keyed by timestamp, not arrival order.

**Backpressure:** bounded queue plus bounded concurrency. A full queue
returns 429 with `Retry-After`; the process never buffers without
limit.

## 4. Raw storage layout

`samples_raw(series_id bigint, ts timestamptz, value float8)`,
partitioned by day on `ts`, `PRIMARY KEY (series_id, ts) INCLUDE
(value)`. The `INCLUDE` makes series-range reads index-only scans: rows
for one series are interleaved with every other series on the heap, so
heap fetches would be random. Append-only partitions get vacuumed by
the insert-driven autovacuum threshold, which sets the visibility map
that index-only scans need. Whether this beats a plain index is a
benchmark question, not an assumption.

**Retention is a partition operation, not a DELETE.** Expired
partitions are removed with `DETACH PARTITION ... CONCURRENTLY` and then
`DROP TABLE`, which avoids long locks on the parent and produces no
dead tuples or vacuum work. A `partition_registry` table records each
partition's range and state (ACTIVE, SEALED, DETACHED, DROPPED) so a
crash between detach and drop is resumable (failure F5). A partition
manager keeps three days of future partitions created ahead of time.

**The drop gate (section 5):** raw partitions are never dropped until
their range is fully rolled up.

## 5. Downsampling: aggregate as data ages

Raw resolution is expensive, so each tier keeps less detail for longer:
raw (15 s) for 48 h, `rollup_5m` for 30 d, `rollup_1h` for 400 d.

**Rollup row** (per series per bucket): `count, sum, min, max, first,
last, inc`. `count` and `sum` make averages exact at every tier
(avg of avgs is wrong; sum over count is right). For counters, `inc` is
the reset-adjusted increase between consecutive samples **inside** the
bucket. For gauges it is NULL.

**Counter semantics (the rule everything else depends on):** the
increase between consecutive samples `a` then `b` is `b - a`, or `b` if
`b < a` (a reset to zero is assumed). The increase over a window is the
sum of those deltas over the sequence formed by the latest sample at or
before the window start (within the staleness lookback) followed by
every sample inside the window. This includes the interval that
straddles the window start, so a 5 minute window at 15 s scrapes counts
20 intervals, not 19.

**Boundary deltas are computed at query time, not stored.** A bucket
stores only deltas inside itself. When a query spans buckets, the delta
across each boundary comes from `last` of one bucket and `first` of the
next (same reset rule, via `lag()` over a series' bucket rows).
Consequence: recomputing one bucket never changes a neighbour's stored
values, so late data needs no cascade.

**Exactness property (tested):** for a bucket-aligned window,
`increase` computed from raw equals `increase` from `rollup_5m` equals
`increase` from `rollup_1h`. It is exact for integer-valued counters and
within float tolerance otherwise. Hierarchical rollup follows from it:
a 1 h bucket's `inc` is the sum of its 5 m `inc` values plus the
interior boundary deltas; `first` and `last` come from the first and
last 5 m buckets; min, max, count, sum compose directly.

**One SQL definition, used twice.** Rolling up a range is a SQL
function `rollup_5m(from, to)` over raw. The persisting job and the
on-read tail (below) both call it, so there is no second implementation
to drift. Bucketing uses `date_bin`.

**Persisting, atomically:** a rollup job closes buckets older than
`now - lateness` (default 2 minutes). For each bucket, one transaction
inserts rows with `ON CONFLICT (series_id, bucket) DO UPDATE` and
advances `rollup_state.watermark`. A bucket is either fully rolled up
and counted by the watermark, or not at all (failure F3). The 1 h tier
rolls up from 5 m the same way.

**Late data:** ingestion checks whether any sample falls before the
current watermark. If so it records the affected buckets in
`rollup_dirty(tier, bucket)`, and the rollup job recomputes them, which
is safe because rollup is an idempotent pure function of raw data. A
dirty 5 m bucket also marks its containing hour dirty.

**The tail problem and its solution.** Buckets after the watermark are
not persisted yet, but recent data is what dashboards and alerts want.
Queries read a view `v_rollup_5m` that is `UNION ALL` of persisted rows
(bucket < watermark) and `rollup_5m(watermark, now)` computed on read
from raw. Both halves have the same shape, so consumers never know. The
on-read part is bounded to the lag (minutes of data). Each query runs
in one `REPEATABLE READ, READ ONLY` transaction, so it sees a consistent
watermark and neither a gap nor a double count at the boundary.

**Safety over tidiness:** the partition manager drops a raw partition
only if `watermark_5m >= partition_end`. If rollups stall, raw
retention extends (disk grows) and an alert fires (`RollupLagHigh`),
instead of silently destroying data that has no rollup yet (failure
F12).

**What cannot be downsampled:** percentiles of gauge values. A p99
cannot be recovered from min, max, sum, count. That is why latency is
modelled as histogram buckets (counters with an `le` label), which roll
up exactly and feed `histogram_quantile`. This limitation is surfaced
in docs, not hidden.

## 6. Query planning and tier selection

`selectTier(start, end, step, window, now, retention, watermarks)` is a
pure function. Rule: pick the coarsest tier whose resolution is at most
the query step and at most the range-function window, among tiers whose
retention covers `start`. Raw wins for fine steps inside 48 h.

**Degradation is explicit.** If `start` predates raw retention but the
requested step is finer than the next tier, the engine uses the coarser
tier anyway and returns `meta.degraded = true` with
`meta.resolution_seconds`. A chart of last month at 15 s steps does not
exist, and the response says so instead of returning a plausible-looking
lie.

**Rollup windows snap to bucket edges.** For an unaligned window on a
rollup tier the error is bounded by one bucket resolution. The tier rule
(resolution at most the window) keeps that to one bucket of a window, and
`meta` reports the tier so callers can judge.

**Query limits (all enforced, all configurable):** max series per query
5,000; max points per series 11,000; max samples loaded 20M (aborts
early); timeout 30 s; max concurrent queries 8; and the query role is a
read-only Postgres role with `statement_timeout` as defense in depth.
`?explain=true` returns the plan: chosen tier, series count, estimated
rows, and the on-read tail span.

## 7. TQL: a deliberately small query language

TQL (Tidemark Query Language) is a PromQL-style subset. It is named
differently on purpose: compatibility is not claimed, and every
deviation is listed.

| Area | v1 |
|---|---|
| Selectors | `metric{l="v", l!="v", l=~"re", l!~"re"}`, range `[5m]`, `offset 1h` |
| Range functions | `rate`, `increase`, `delta`, `avg_over_time`, `min_over_time`, `max_over_time`, `sum_over_time`, `count_over_time`, `last_over_time` |
| Other functions | `histogram_quantile`, `absent`, `vector`, `time`, `abs` |
| Aggregations | `sum`, `avg`, `min`, `max`, `count`, with `by` or `without` |
| Binary operators | `+ - * / %`, comparisons (filtering), `and`, `or` |
| Matching | vector-scalar; vector-vector only on identical label sets |
| Not in v1 | subqueries, `on`/`ignoring`/`group_left`, `topk`, `irate`, `@`, recording rules |

**Semantics:**
- An instant selector at time `t` returns the latest sample at or before
  `t` if it is within the staleness lookback (default 5 minutes);
  otherwise the series is absent at `t`.
- `rate(x[w])` is `increase(x[w]) / w`, with `increase` defined in
  section 5. This is **not** Prometheus's extrapolating `rate`. The
  difference is documented, and it is what makes raw and rollup tiers
  agree exactly.
- Comparisons filter, as in PromQL: `x > 5` keeps only series where it
  holds. This is what alert rules use.
- `absent(selector)` returns a single series of value 1 when the
  selector matches nothing, which is how "the data stopped" becomes
  something an alert can fire on (section 9).

**Implementation:** lexer, Pratt parser to an AST, a type checker
(scalar, instant vector, range vector) that rejects invalid shapes with
useful errors, then a planner that batches one fetch per selector and
evaluates per series over the step grid. The evaluator is a pure
function of (AST, data, `t`), which is what makes alert backtesting and
golden tests possible. The parser is fuzzed: it must return an error or
a result and never panic or hang.

## 8. Logs: index what you search, then verify

**Model:** a stream is a label set (like a Loki stream), stored in
`streams` with the same hash identity as series. Entries live in `logs(
ts, stream_id, entry_id, message, fields jsonb, tokens text[],
ingested_at)`, partitioned hourly, `UNIQUE (ts, entry_id)`. `entry_id`
is sender-supplied or a content hash, so a re-sent batch is a no-op.
Hourly partitions bound partition count (about 170 for 7 days) and keep
each partition's index small.

**Stream labels must be low cardinality.** Variable data (order ids,
user ids) goes in `fields`, not labels. Stream count is capped, and the
docs explain why, because it is the most common way log systems are
misused.

**Tokenizer (shared by writer and query planner):** lowercase, split on
non-alphanumerics, with UUIDs, IPv4 addresses, and long hex ids also
kept whole. Tokens go in `tokens text[]` with a GIN index. Postgres
`tsvector` was rejected as the default: its stemming and stop-word
handling are built for prose, and a stop-word list that drops "no" or
"not" is wrong for logs.

**Index-then-verify:** the GIN index (`tokens @> ARRAY[...]`) narrows
candidates, then the exact predicate (substring or RE2 regex, applied
in Go or SQL as appropriate) runs on the candidates. The index may
return false positives but **must never produce a false negative**.

**The edge-token rule:** for a substring search such as `"out after 3"`,
the first and last runs could be fragments of larger words (`timeout`,
`30s`). Only runs bounded by a delimiter inside the literal are
guaranteed whole tokens and usable for the index; edge runs are dropped.
So substring search is correct everywhere and index-accelerated where
possible, and a whole-word operator (`|=w "timeout"`) is fully
accelerated because every run is a whole token. `?explain=true` says
which tokens were used. This invariant is tested against a brute-force
oracle.

**Head and sealed partitions:** maintaining a GIN index under heavy
insert is expensive, so the current (head) hourly partition has no GIN
index, and queries against it filter by scan (bounded to one hour of
data). A seal job creates the GIN index with `CREATE INDEX CONCURRENTLY`
on the partition after the hour ends plus a grace period. A config flag
(`log_index_mode = sealed | always`) falls back to indexing everything.
Which wins is a benchmark question.

**Query language (LogQL-flavoured):**
`{service="counterweight-api", env="dev"} |= "timeout" != "retry" |~
"order=[0-9]+" | level="error"`. Selector narrows streams, `|=` and
`!=` are substring filters, `|~` and `!~` are RE2, `| field="x"` filters
structured fields. `count_over_time({...} |= "x" [5m])` returns a
vector, so log-derived conditions can drive alerts.

**Tail:** `GET /api/v1/logs/tail` (SSE) polls with a cursor on
`(ingested_at, id)` and a 2 second safety lag. The cursor is ingest
time, not event time, so a late-arriving old line still appears, and
the lag avoids missing rows whose transaction committed out of
sequence.

**Limits:** line max 64 KiB, query limit default 1,000 (max 5,000),
retention 7 days by partition drop.

**Redaction at ingest:** configurable regex scrubbing (bearer tokens,
JWT shapes, cloud key prefixes, card-like numbers) runs before storage.
It is a heuristic and the docs say so: it reduces risk and does not
guarantee that secrets are never logged.

## 9. Alerting: a state machine fed by queries

**Rule:** name, TQL expression, `for`, `keep_firing_for`, labels,
annotations (templates), interval, `missing_policy`, receivers. Rules
are versioned immutably (`rule_versions`); every event records the
version and the evaluated value, so "why did this fire" is answerable
after the rule changes.

**Aligned, pure evaluation.** Ticks are wall-clock multiples of the
interval (`:00`, `:15`, ...), and evaluation runs the expression "as of"
the tick time, not "now". The result is a deterministic function of
(rule, tick, stored data). Instances are keyed by a fingerprint of their
label set.

**Instance state machine:**

| State | Meaning | Notifies |
|---|---|---|
| INACTIVE | condition false or never seen | no |
| PENDING | true, `for` not yet elapsed | no |
| FIRING | true for at least `for` | on entry, on repeat |
| FIRING (clearing) | false, inside `keep_firing_for` | no |
| RESOLVED | was firing, now cleared | once |

Transitions: INACTIVE to PENDING when true; PENDING to INACTIVE when
false (silent); PENDING to FIRING when elapsed at least `for`; FIRING to
RESOLVED after false for at least `keep_firing_for`; RESOLVED to
PENDING or FIRING when true again, with `episode` incremented. `for`
guards entry flapping, `keep_firing_for` guards exit flapping.

`Step(state, observation, tick, rule) -> (state', effects)` is a pure
function, tested exhaustively with a fake clock, like Tessera's
transition function.

**"The data stopped" is the central problem.** If a service dies, its
series go stale and `up == 0` returns nothing, not a 0. A naive rule
therefore never fires on exactly the failure it was written for. Three
defences:
1. The agent synthesizes `up{job,instance}` (1 or 0) on every scrape, so
   a down target produces an explicit 0.
2. `absent(...)` turns "no series" into a value rules can match.
3. `missing_policy`: `resolve` (the instance disappears, treated as
   resolved, default), `hold` (stay firing until explicitly false), or
   `no_data` (a distinct NoData alert).

**Errors are not data.** If a query fails or times out, instance state
is left untouched, never resolved, never fired, and a `RuleEvaluationFailing`
condition is raised. Treating a failed query as "condition false" would
resolve alerts during exactly the outage you are trying to see.

**Evaluator restarts and gaps:** `PENDING` timers are measured in tick
time. If an instance's last evaluation is older than 3 intervals (the
evaluator was down), the pending clock restarts, because the system
cannot claim the condition held during the gap. FIRING instances are
kept until evaluated false. Missed ticks are not replayed.

**Persistence and concurrency:** each rule's tick is one transaction:
update `alert_instances` (compare-and-swap on `version`), append to
`alert_events` (append-only, trigger-enforced), insert any notification
rows. A leader lock keeps one evaluator active. If two overlap briefly,
both compute the same transition, and the notification key
`(rule_id, fingerprint, episode, kind, seq)` is `UNIQUE`, so one
notification results (failure F9).

**Backtesting:** `POST /v1/admin/rules/backtest` runs a candidate
expression and the same `Step` function over a historical range and
returns the timeline of transitions, sending nothing. It evaluates
against data as stored now (late data included), so it is a what-if,
not a replay of what the live system saw. Rules whose windows exceed raw
retention are rejected for live evaluation, because alerts must not run
on degraded resolution.

Example rule (illustrative metric names):

```yaml
groups:
  - name: portfolio
    interval: 15s
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels: {severity: page}
        annotations:
          summary: "{{ $labels.job }} on {{ $labels.instance }} is down"
      - alert: SagaNeedsAttention
        expr: max by (service) (counterweight_sagas{status="NEEDS_ATTENTION"}) > 0
        for: 2m
        keep_firing_for: 5m
        labels: {severity: page}
```

## 10. Notification delivery

**Outbox:** a state transition and its notification row commit in one
transaction, so there is no transition without a notification and no
notification without a transition (same discipline as Dispatcher's
outbox). A delivery worker claims rows with `FOR UPDATE SKIP LOCKED`.

**Delivery:** at-least-once with a stable `notification_id` sent as an
idempotency key, so receivers can dedupe. Retries use exponential
backoff with jitter (1 s up to 5 min, 10 attempts), then the row goes to
DEAD and a self-alert fires. If the process dies after the HTTP call and
before marking SENT, the notification is re-sent with the same id
(failure F8). Delivery is at-least-once by design.

**Grouping and repeats:** one message per (receiver, rule, tick)
containing all changed instances, so 40 pods failing together produce
one message, not 40. FIRING re-notifies every `repeat_interval` (default
4 h).

**Silences:** matcher-based, time-bounded. A silenced alert still
transitions and is recorded; only its notification is suppressed, and
the suppression is itself recorded, so silencing never loses the
timeline.

**Receivers:** generic JSON webhook, Slack-compatible payload, stdout
(demo), and a Dispatcher endpoint (reusing the earlier project's
retry and DLQ machinery as an optional second hop). Payloads are
HMAC-signed with a per-receiver secret and timestamp, the same scheme
Counterweight verifies inbound.

**Safety:** annotation templates use `text/template` with a minimal
function set, are compiled and validated when the rule is saved, and
render only into JSON (never HTML), because label values are untrusted.
Receiver URLs pass an SSRF guard: link-local and cloud-metadata
addresses are blocked, with private ranges configurable because
self-hosted receivers legitimately live there.

## 11. Who watches the watcher

1. **Self-scrape:** Tidemark exposes `/metrics` and its own agent ingests
   it through the same pipeline, so every internal health number is
   queryable and alertable.
2. **Watchdog:** an always-true rule (`vector(1)`) notifies the canary
   every minute.
3. **Canary:** a small separate binary, deployed on a different host,
   that receives Watchdog heartbeats and raises its own alarm (to a
   different channel) after 3 missed heartbeats. This is the only way to
   detect "Tidemark is down", because a dead evaluator cannot alert
   about itself.
4. **Self-alerts:** `RollupLagHigh`, `EvaluatorStalled`,
   `OutboxLagHigh`, `IngestRejectsSpike`, `DeadNotifications`,
   `PartitionsMissing`, `SeriesLimitNear`.

## 12. The agent: scraping and shipping

**Scrape:** pulls Prometheus text exposition from configured targets
(static list, docker-compose service names; no `docker.sock` mount by
default because it grants host-level control), attaches `job` and
`instance`, and writes `up`, `scrape_duration_seconds`, and
`scrape_samples_scraped` per target. The text format is parsed with the
`prometheus/common/expfmt` library; reimplementing it adds nothing to
the point.

**Ship:** a bounded in-memory queue with an optional on-disk spool (size
and age limited) so server outages and agent restarts do not drop data
within the limit. When limits are hit the oldest data is dropped and a
counter says how much. Sending follows the response contract in
section 3: retry only on 429, 5xx, and network errors, with exponential
backoff, jitter, and `Retry-After`. Because storage is idempotent,
duplicate sends are harmless.

**Log tail:** follows files with offsets checkpointed **after** server
acknowledgement, handling rotation and truncation (inode plus size). A
crash between ack and checkpoint re-sends lines (failure F10), and
`entry_id` derived from (file, offset) makes that a no-op. JSON log
lines are parsed into level, message, and `fields`.

## 13. Retention and partition management

One leader-elected manager owns all partitioned tables: create future
partitions (3 days ahead for samples, 3 days for logs), seal log
partitions (create GIN), detach then drop expired ones, and record each
step in `partition_registry`. Every step is idempotent and resumable.
Drops are gated: samples by rollup watermark (section 5), logs by age
only. Dropping is logged to an append-only table so "where did that data
go" has an answer.

## 14. Dogfooding: Tidemark watches the portfolio

The roadmap closes by pointing this at the projects built before it.

**Observability contract** every project follows (a one-page doc in the
repo): a `/metrics` endpoint in Prometheus text format, `service` and
`env` labels, and JSON logs to stdout with `level`, `msg`, and a
`correlation_id` where one exists. Tessera and Counterweight specify
`/metrics` already. The others need a small endpoint added, tracked as a
checklist in `CURSOR_CONTEXT.md`.

**Provisioned dashboard "Portfolio Health"** (dashboards as JSON files):
per service `up`, request rate, error ratio, p95 latency; Counterweight
sagas by status, NEEDS_ATTENTION count, outbox lag, critical
reconciliation findings; Tessera first-offer latency and NO_DRIVERS
rate; Tidemark's own ingest rate, rejects by reason, rollup lag,
evaluator lag, and table size by tier. Metric names are illustrative
until those projects expose them.

**Cross-service log search:** `{env="dev"} |= "saga_id=..."` follows one
order across services via `correlation_id`. It is not tracing, and it is
described as what it is.

**Demo script (the story):** kill the Counterweight worker; `up` drops
to 0, the alert goes PENDING then FIRING, a signed notification lands at
the receiver, the log panel shows the last errors for that service, the
worker restarts, and the alert resolves. Then stop Tidemark's own
evaluator and watch the canary raise the alarm.

## 15. Security

- API keys: high-entropy, stored as SHA-256 hashes (as in VaultKeep),
  scopes `write`, `read`, `admin`, per-key series limits; shown once at
  creation.
- Query API connects as a read-only Postgres role; all SQL is
  parameterized; user regexes run under RE2 (no catastrophic
  backtracking) with a pattern size limit.
- **UI treats all log lines and label values as hostile:** rendered with
  `textContent`, never `innerHTML`, under a strict CSP with no inline
  script.
- Outgoing webhooks signed; SSRF guard on receiver URLs; templates
  sandboxed (section 10).
- Ingest redaction (section 8). TLS terminates at a reverse proxy; the
  doc says so rather than implying built-in TLS.
- Request size limits, decompression bomb limit on gzip bodies, and
  concurrency caps on every public endpoint. Per-IP rate limiting is
  delegated to GateKeeper or the proxy.

## 16. Failure modes

| ID | Failure | What happens | Why it is safe |
|---|---|---|---|
| F1 | Ingester dies before commit | No ack; client retries | Idempotent insert |
| F2 | Dies after commit, before 200 | Client retries; duplicates counted | PK + `DO NOTHING` |
| F3 | Rollup dies mid-bucket | Transaction rolls back, watermark unchanged | Bucket recomputed |
| F4 | Dies after watermark commit | Nothing to do | State is atomic |
| F5 | Dies after DETACH, before DROP | Registry says DETACHED; next run drops | Resumable steps |
| F6 | Evaluator dies mid-cycle | Remaining rules run next tick | Per-rule transaction |
| F7 | Dies before state commit | Old state and no notification | Same transaction |
| F8 | Dies after HTTP send, before SENT | Same notification re-sent | Stable id; receiver dedupes |
| F9 | Two evaluators briefly | One transition, one notification | Version CAS + UNIQUE key |
| F10 | Agent dies after ack, before checkpoint | Lines re-sent | Deterministic `entry_id` |
| F11 | Postgres unavailable | Ingest 503; agent spools; evaluator holds state | Errors are not data |
| F12 | Rollups stall | Raw partitions not dropped; alert | Drop gate |
| F13 | Receiver down for hours | Retries, then DEAD, then self-alert | Bounded retry |
| F14 | Cardinality explosion | New series rejected with reason | Caps at creation |

## 17. Reconciliation: proving the stored data is what it claims

A scheduled job records every run, clean or not, in
`reconciliation_runs` with findings in `reconciliation_findings`:

1. **Rollup verify:** sample closed buckets still inside raw retention;
   recompute from raw and compare with stored 5 m rows; compare 1 h with
   aggregated 5 m. Exact for count, min, max, first, last; tolerance for
   float sums.
2. **Coverage:** every series with raw rows in a sampled closed bucket
   has a rollup row.
3. **Watermarks and dirty set:** monotonic, not ahead of
   `now - lateness`, no dirty buckets older than a threshold.
4. **Orphans:** samples whose `series_id` has no series row (no FK, so
   checked here).
5. **Partition registry vs catalog:** every registered partition exists,
   every partition is registered, future partitions exist.
6. **Alert consistency:** FIRING instances with no corresponding
   notification, notifications pending past threshold, rules not
   evaluated for 3 intervals.
7. **Series GC candidates:** series with no data in any tier.

This is the same "fast derived data, authoritative source, scheduled
proof they agree" shape as LedgerLine's balance check, applied to
rollups.

## 18. Observability of Tidemark itself and targets

Metrics: samples received, accepted, duplicates, rejected by reason;
batch commit latency; queue depth; active series; series creation rate;
rollup lag per tier; dirty bucket count; query latency by tier and
degraded count; samples loaded per query; evaluator lag; evaluations
failed; instances by state; notification outbox lag; deliveries by
result; log lines ingested and rejected; partition count; table size by
tier; reconciliation findings by type. Logs are structured and carry
`request_id`, `rule_id`, and `tick`.

Targets to validate, not claims: ingest 20,000 samples/s sustained on one
node; batch commit p99 under 100 ms; instant query over 100 series p95
under 50 ms; 24 h range at 1 min step, 100 series, p95 under 500 ms; 30 d
at 5 min step p95 under 1 s; 1 y at 1 h step p95 under 1 s; log search
with stream and term over 24 h of sealed partitions p95 under 1 s;
evaluator lag p99 under 2 s for 200 rules at 15 s; zero lost
acknowledged samples across the failpoint suite.

## 19. Data model summary

`series`, `samples_raw` (daily), `rollup_5m`, `rollup_1h`,
`rollup_state` (watermarks), `rollup_dirty`, `streams`, `logs` (hourly),
`api_keys`, `rules`, `rule_versions` (immutable), `alert_instances`,
`alert_events` (append-only), `notifications` (outbox), `receivers`,
`silences`, `dashboards`, `partition_registry`, `partition_drops`
(append-only), `reconciliation_runs`, `reconciliation_findings`.

## 20. Pattern reuse map (the series, closed)

| Mechanism here | Where it was established |
|---|---|
| Idempotent writes by unique key, safe retries | LedgerLine |
| Append-only history enforced by trigger | LedgerLine, Tessera, Counterweight |
| Cached or derived data checked against truth on a schedule | LedgerLine |
| Transactional outbox, SKIP LOCKED delivery, backoff | Dispatcher |
| Leader election via advisory lock | Counterweight, Dispatcher |
| Pure transition function plus CAS | Tessera, Counterweight |
| Incident-style state machine with notification | StatusForge |
| Failpoint crash testing | Counterweight |
| Hashed API credentials, shown once | VaultKeep |
| Signed webhooks, replay window | Counterweight |
| Constraint as backstop | SlotForge |
| Search with index narrowing then verification | SearchCraft |
| Rate limiting delegated | GateKeeper |
| Optional second-hop delivery | Dispatcher |

What is new: time as a storage dimension (retention that degrades
gracefully and says so), a query language with defined semantics, and an
alerting engine designed around absence and failure.

## What is out of scope, and why

- **Multi-tenancy, RBAC beyond three scopes.**
- **Distributed tracing.** Correlation-id log search is the substitute.
- **Prometheus compatibility.** An optional Prometheus-shaped read API
  (for pointing Grafana at it) is an extra, with documented gaps.
- **Stale markers.** Prometheus encodes them as special NaN payloads;
  Postgres does not preserve NaN payloads in `float8`, as far as I know,
  so the staleness lookback and synthesized `up` are used instead.
- **Recording rules, subqueries, topk, vector matching with `on`.**
- **Email and paging integrations.** Webhook, Slack-style, and the
  Dispatcher hop only.
- **High availability, clustering, long-term object storage.**

## What I'd change with more time

- **Columnar chunks for the raw tier** (one row per series per hour
  holding delta-of-delta timestamps and XOR-compressed values in
  `bytea`) for roughly an order of magnitude less storage.
- **Compact sealed raw partitions** by rewriting them in series order so
  range reads become sequential.
- **Protobuf write protocol** and a Prometheus remote-write receiver.
- **Recording rules** to precompute expensive expressions.
- **Alert inhibition and routing trees.**
- **Exemplars** linking a latency spike to a correlation id.
- **A TSDB backend** behind `SampleStore` for scale beyond this design.