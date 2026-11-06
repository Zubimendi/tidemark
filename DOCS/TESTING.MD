# Testing: Tidemark

**Status: this is a test plan, not a test report.** No code exists yet,
so nothing below has been run. Every "Result" field is empty on
purpose. When a test is run, record the date, commit, hardware, and
actual outcome. Do not copy targets into result fields.

## Philosophy

A monitoring system's worst failure is a confident wrong answer, so most
tests compare the system against an independent oracle: raw data
against summaries, brute-force search against the index, a pure
function against a state machine run. Crash safety is tested by killing
the process at named instructions. Time-dependent code takes an
injected clock, so tests advance time instead of sleeping.

## Test environment

- `docker compose up` brings up Postgres 16 and, optionally, the server,
  agent, and canary.
- Each integration test uses a fresh schema or prefixed names.
- `go test -race` everywhere; fuzz targets run for a bounded time in CI.
- **Failpoints:** `FAILPOINT=<name>` kills the process at a named
  instruction. Compiled in only when `ENABLE_FAILPOINTS=true`, and
  production config refuses that flag.
- Record Go and Postgres versions, CPU, RAM, native or VM Docker, and
  Postgres settings for every benchmark.

```bash
make up
make test               # unit tests, no services
make test-db            # raw SQL tests
make test-integration
make test-failpoints
make bench
make fuzz               # bounded fuzz run
```

## 1. Unit tests

**1.1 Series identity.** Canonical encoding is order-independent for
labels; any change in name, value, or metric changes the hash;
length-prefixing prevents `a="b,c"` colliding with two labels. A forced
fake collision is rejected by the label comparison.

**1.2 TQL lexer and parser.** Valid corpus parses to expected ASTs;
invalid corpus returns errors with positions; the type checker rejects
range vectors where instant vectors are required and vice versa.
**Fuzz target:** random input never panics or hangs.

**1.3 Counter semantics.** Table-driven `increase` over sequences:
monotonic, one reset, multiple resets, reset to non-zero, gap longer
than lookback (no boundary delta), series starting inside the window,
single sample, empty window, and the boundary-straddling delta
(20 intervals, not 19, for a 5 minute window at 15 s).

**1.4 `selectTier`.** Table over (start, step, window, now) to chosen
tier and degraded flag, including: fine step inside raw retention (raw),
old start with fine step (coarser tier, degraded), 1 y at 1 h (1 h tier),
window smaller than a tier's resolution (finer tier).

**1.5 Alert `Step` function.** Exhaustive table with a fake clock over
every state and observation:

- INACTIVE true goes to PENDING; `for = 0` goes straight to FIRING
- PENDING false goes to INACTIVE silently
- PENDING true past `for` goes to FIRING and emits one notification
- FIRING true emits repeat only after `repeat_interval`
- FIRING false with `keep_firing_for` stays FIRING (clearing), then
  RESOLVED
- RESOLVED true again starts a new episode
- missing data under each policy (`resolve`, `hold`, `no_data`)
- evaluation error leaves state unchanged and emits no notification
- gap longer than 3 intervals resets PENDING but not FIRING

**1.6 Notification keys.** Deterministic from (rule, fingerprint,
episode, kind, seq); different episodes never collide.

**1.7 Tokenizer.** Lowercasing, splitting, UUIDs, IPv4, hex ids kept
whole, unicode, empty input, very long tokens. The edge-token rule:
literal `"out after 3"` yields only interior tokens; `"timeout"` with
the whole-word operator yields the whole token.

**1.8 Redaction.** Each pattern (bearer token, JWT shape, key prefixes,
card-like number) redacts; near-miss strings are left alone; behavior
documented as heuristic.

**1.9 Template safety.** Rule save rejects templates calling disallowed
functions or exceeding size; rendering escapes into JSON.

**1.10 Backoff and signing.** Bounded exponential backoff with jitter;
signature verifies for the right secret and fails for altered body or
timestamp.

Result: _not yet run_

## 2. Database tests (raw SQL)

These bypass application code. If one passes when it should fail, the
guarantee is not really a database guarantee.

**2.1 Partition routing.** Rows land in the correct daily partition; a
row outside every partition fails with the expected error.

**2.2 Idempotent insert.** The same `(series_id, ts)` inserted twice with
`ON CONFLICT DO NOTHING` yields one row; a different value for the same
key is ignored, and the returned count shows it.

**2.3 Index-only scan.** `EXPLAIN` of a series-range read on a vacuumed
partition shows an index-only scan; record how the plan changes before
vacuum.

**2.4 Append-only tables.** UPDATE and DELETE on `alert_events` and
`partition_drops` are rejected.

**2.5 Role isolation.** The read-only query role cannot INSERT, UPDATE,
DELETE, or run DDL; `statement_timeout` cancels a long query.

**2.6 Rollup function equivalence with a naive SQL oracle.** For random
raw data, `rollup_5m(from, to)` equals a straightforward per-sample
computation written independently in the test.

**2.7 Detach and drop.** `DETACH PARTITION CONCURRENTLY` while a query
runs completes after the query; the registry state matches the catalog
at each step.

**2.8 Unique notification key.** Two inserts with the same key: one
row.

Result: _not yet run_

## 3. Ingestion integration tests

**3.1 Retry safety.** Send a batch, then send it again 10 times, and
concurrently 50 times. Row count equals unique samples; responses report
duplicates.

**3.2 Accounting identity.** Randomized batches with a mix of valid,
future, too old, over-limit, and malformed samples. For every response
and in aggregate, `received = accepted + duplicates + rejected`.

**3.3 Response contract.** Malformed gives 400; oversize 413; full queue
429 with `Retry-After`; stopped Postgres 503; a permanent reject inside
a 200 is not retried by the agent (verified in the agent test).

**3.4 Concurrent series creation.** 100 workers push the same new series
at once; exactly one series row exists; every worker gets the same id.

**3.5 Cardinality caps.** Exceeding label count, name or value length,
or the global series cap rejects with the right reason and does not
affect existing series.

**3.6 Poison isolation.** A coalesced batch containing one poisoned
request (forced insert error) commits the others and reports the bad
one.

**3.7 Group commit.** Many small concurrent requests share commits;
record commits per request.

**3.8 Time rules.** Future beyond 5 minutes rejected; older than raw
retention rejected; out-of-order within the window accepted.

Result: _not yet run_

## 4. Downsampling tests (the most important group)

### 4.1 Tier equivalence property

Generate random counters with resets, gaps, irregular intervals, and
duplicate timestamps. For random bucket-aligned windows, compute
`increase` from raw, from `rollup_5m`, and from `rollup_1h`. Assert all
three are equal (exact for integer counters, within a stated tolerance
for floats). Also assert gauge `avg`, `min`, `max`, `count`, `sum`,
`last` agree. Run thousands of random cases per commit.

Result: _not yet run_

### 4.2 Late data

Ingest, roll up, then ingest a late sample into a summarized bucket.
Assert the bucket is marked dirty, recomputed, and now matches raw;
neighbouring buckets' stored rows are byte-for-byte unchanged; the
hourly bucket is recomputed too.

Result: _not yet run_

### 4.3 Watermark consistency under concurrency

Run queries continuously while the rollup job advances the watermark.
Assert no query ever shows a gap or a double count at the boundary.

Result: _not yet run_

### 4.4 Atomic buckets (F3, F4)

Kill the rollup mid-bucket: watermark unchanged, no partial rows
visible. Kill after commit: nothing to redo. Restart converges to the
same rows as an uninterrupted run.

Result: _not yet run_

### 4.5 Drop gate (F12)

Stop the rollup job, advance the fake clock past raw retention, run the
partition manager. Assert no raw partition with un-rolled data is
dropped, disk-growth metric rises, and `RollupLagHigh` fires. Resume
rollups and assert partitions then drop.

Result: _not yet run_

### 4.6 Tail view

With a lagging watermark, queries over the most recent minutes return
the same values as after the rollup catches up.

Result: _not yet run_

### 4.7 Degraded resolution

Query last 30 days at 15 s step: result carries `degraded = true` and
the effective resolution; the values equal the coarser tier's.

Result: _not yet run_

## 5. Query engine tests

**5.1 Golden files.** Directory of (dataset, query) to expected result
JSON covering every function, aggregation, and operator, including
`offset`, `absent`, `histogram_quantile`, comparison filtering, and
`and`/`or` on identical label sets.

**5.2 Limits.** Max series, max points, max samples loaded (aborts
early), timeout, and concurrency cap each produce a clear error; none
exhausts memory.

**5.3 Staleness.** A series with no sample in the last 5 minutes is
absent from an instant query; `absent()` returns 1.

**5.4 Regex semantics.** `=~` and `!~` are fully anchored RE2; patterns
over the size limit are rejected; behavior on Postgres POSIX
differences is not reachable because regex never runs there.

**5.5 `explain`.** Plan lists tier, series count, estimated rows, tail
span; does not read samples.

**5.6 Optional differential suite.** Load the same synthetic data into
a real Prometheus and compare results for supported queries. Expect
differences for `rate` because of extrapolation, so compare `increase`
over aligned windows and document the deviation. Optional, requires a
Prometheus container.

Result: _not yet run_

## 6. Logs tests

### 6.1 Oracle property test

Generate random corpora (words, ids, UUIDs, punctuation, mixed case) and
random queries (substring, whole word, regex, negation, combined with
stream selectors). For each, compare the indexed plan's result with a
brute-force scan using only the verification predicate. Assert equality.
This enforces "false positives allowed, false negatives never".

Result: _not yet run_

### 6.2 Edge-token cases

Searches for fragments (`"out"` in `timeout`, `"3"` in `30s`,
`"order-123"` in `reorder-123`) all return the correct lines.

Result: _not yet run_

### 6.3 Head versus sealed

A query returns identical results before and after a partition is
sealed and indexed. `explain` shows a scan on the head and an index
plan after sealing.

Result: _not yet run_

### 6.4 Dedupe and ordering

Re-sent lines do not duplicate. Lines with identical timestamps keep a
stable order. Late lines appear in tail via the ingest-time cursor.

Result: _not yet run_

### 6.5 Limits and hostile input

Line over 64 KiB rejected or truncated per config; stream cardinality cap
enforced; hostile content (script tags, control characters, extremely
long tokens) stored and returned as text.

Result: _not yet run_

### 6.6 Log-derived alerts

`count_over_time({...} |= "x" [5m]) > 10` drives an alert through the
normal state machine.

Result: _not yet run_

## 7. Alerting integration tests

**7.1 Lifecycle.** Feed controlled metrics to a rule; assert exact tick
times of PENDING, FIRING, RESOLVED and the events recorded with rule
version and value.

**7.2 Target death.** Stop a scraped target. `up` becomes 0, then
`ServiceDown` fires after `for`. Variant: no agent `up` available, use
`absent()`.

**7.3 Errors are not data.** Make queries fail (stop Postgres or inject
timeouts). Assert no instance changes state, `RuleEvaluationFailing` is
raised, and after recovery states continue correctly.

**7.4 Evaluator restart.** Kill the evaluator while instances are
PENDING and FIRING; on restart, PENDING with an old last evaluation
resets, FIRING persists.

**7.5 Duplicate evaluators (F9).** Force two evaluators briefly; assert
one transition and one notification row.

**7.6 Outbox atomicity (F7).** Kill before state commit: neither state
nor notification exists. After commit: both exist.

**7.7 Delivery (F8, F13).** Kill after HTTP send and before marking
SENT: re-sent with the same `notification_id`. Receiver down for a long
period: retries with backoff, then DEAD, then a self-alert. Signatures
verify at the receiver.

**7.8 Grouping and repeat.** 40 instances changing in one tick produce
one message; repeat only after `repeat_interval`.

**7.9 Silences.** A silenced alert still transitions and is recorded;
the notification is marked suppressed with the silence id; expiry
resumes delivery.

**7.10 Backtest.** Run a rule's historical range with `Step`; compare
with a live run over the same synthetic data (replay determinism);
assert nothing is sent.

**7.11 SSRF guard.** Receiver URLs to metadata and link-local addresses
are rejected; allowed private ranges follow config.

Result: _not yet run_

## 8. Agent tests

**8.1 Scrape.** Parses counters, gauges, histograms, and malformed
exposition without crashing; writes `up`, `scrape_duration_seconds`,
`scrape_samples_scraped`; a failing target yields `up = 0`.

**8.2 Spool and outage.** Stop the server for N minutes; the agent
spools; after restart, all data arrives, none duplicated. Spool limit
reached: oldest dropped and counted.

**8.3 Response contract.** 400 is dropped without retry; 429 honors
`Retry-After`; 503 backs off with jitter; 401 stops and logs loudly.

**8.4 Log tail.** Rotation, truncation, partial lines, restart with
checkpoint, crash between ack and checkpoint (F10): lines re-sent, none
duplicated in storage.

Result: _not yet run_

## 9. Failpoint suite

One test per crash point from architecture section 16. Method: start
with `FAILPOINT=<name>`, drive the workload until the process dies,
restart without it, wait for convergence, then assert the end-state
invariants below and equality with an uninterrupted run.

| ID | Failpoint | Extra assertion |
|---|---|---|
| F1 | ingest, before commit | no ack; retry succeeds; no partial rows |
| F2 | ingest, after commit, before 200 | retry reports duplicates; no extra rows |
| F3 | rollup, mid-bucket | no partial rows; recompute matches |
| F4 | rollup, after watermark | nothing redone |
| F5 | partition manager, after DETACH | next run drops it |
| F6 | evaluator, mid-cycle | remaining rules run next tick |
| F7 | evaluator, before state commit | no state, no notification |
| F8 | delivery, after send | same id re-sent |
| F9 | two evaluators | one notification |
| F10 | agent, after ack, before checkpoint | lines re-sent, no duplicates stored |
| F11 | Postgres unavailable | ingest 503, agent spools, evaluator holds state |
| F12 | rollup stalled | raw not dropped |
| F13 | receiver down | retry, DEAD, self-alert |
| F14 | cardinality explosion | new series rejected, existing unaffected |

**End-state invariants (asserted after every failpoint run):**
every acknowledged sample is present exactly once; accounting identity
holds; watermarks are monotonic and never ahead of persisted buckets;
no raw partition is missing un-rolled data; every FIRING instance has a
notification; every notification id is unique; the reconciler reports
zero critical findings.

Result: _not yet run_

## 10. Reconciler tests

Inject each drift and assert detection and recording:

| Injected drift | Expected |
|---|---|
| a stored 5 m row altered | rollup verify finding |
| a rollup row deleted | coverage finding |
| watermark set ahead of reality | watermark finding |
| samples with a deleted series row | orphan finding |
| a partition dropped by hand | registry finding |
| FIRING instance with no notification | alert consistency finding |
| rule not evaluated for 3 intervals | liveness finding |

A clean run is persisted with zero findings.

Result: _not yet run_

## 11. Meta-monitoring tests

Stop the evaluator (or the whole server): the canary receives no
heartbeats and raises its alarm after the configured misses, through a
different channel. Self-alerts fire for rollup lag, evaluator stall, and
dead notifications.

Result: _not yet run_

## 12. UI security tests

Hostile log lines and label values (`<script>`, event-handler
attributes, huge strings) render as inert text in log and chart
panels; CSP blocks inline script; API key never appears in URLs or
logs.

Result: _not yet run_

## 13. Benchmarks and load

Report every number with hardware and configuration. Run each at least
three times and record the spread.

| Benchmark | Method | Target |
|---|---|---|
| Ingest throughput | `cmd/load`, 10k series, ramp to saturation | 20,000 samples/s |
| Batch commit latency | same | p99 under 100 ms |
| `unnest` insert vs COPY fallback | same workload | record both |
| Index-only vs plain index reads | 24 h range, 100 series | record |
| Instant query | 100 series | p95 under 50 ms |
| 24 h at 1 min step | 100 series, raw | p95 under 500 ms |
| 30 d at 5 min step | 100 series, 5 m tier | p95 under 1 s |
| 1 y at 1 h step | 100 series, 1 h tier | p95 under 1 s |
| Rollup job | one 5 m bucket at 10k series | record, well inside the interval |
| Log ingest | 5,000 lines/s, head partition | sustained |
| Log search | stream and term, 24 h sealed | p95 under 1 s |
| Log index mode | `sealed` vs `always` ingest and query | record, decide default |
| Evaluator | 200 rules at 15 s | lag p99 under 2 s |
| Storage | table sizes by tier after 48 h of reference load | compare with estimates |

Tools: Go benchmarks for pure code, `cmd/load` for end to end,
`pg_stat_statements` and `EXPLAIN (ANALYZE, BUFFERS)` for query tuning.

Result: _not yet run_

## 14. Manual demonstrations

Verify commands against the real schema once migrations exist.

**14.1 Start and generate data**

```bash
make up && make server &
make load
```

**14.2 Prove retries are safe**

```bash
curl -X POST localhost:8080/api/v1/write -H "Authorization: Bearer $WRITE_KEY" \
     -d @batch.json
curl -X POST localhost:8080/api/v1/write -H "Authorization: Bearer $WRITE_KEY" \
     -d @batch.json
# second response reports duplicates; row count unchanged
```

**14.3 Watch downsampling**

```sql
SELECT count(*) FROM samples_raw;
SELECT count(*) FROM rollup_5m;
SELECT * FROM rollup_state;
```

Query the same counter at three steps and compare `meta.tier` and
values.

**14.4 See the degraded flag**

Request 30 days at 15 s step; read `meta.degraded`.

**14.5 Prove the drop gate**

Stop rollups, set short raw retention, run the partition manager, and
confirm nothing is dropped and the alert fires.

**14.6 Search logs and read the plan**

```bash
curl -G localhost:8080/api/v1/logs/query_range \
  --data-urlencode 'query={service="api"} |=w "timeout"' \
  --data-urlencode 'explain=true'
```

**14.7 The kill-and-recover story**

Start `make portfolio`, kill one portfolio service, and watch the
dashboard: `up` drops, the alert goes PENDING then FIRING, a signed
notification arrives at the receiver, the log panel shows the last
errors, restart the service, and watch RESOLVED.

**14.8 Stop the watcher**

Stop Tidemark's evaluator and watch the canary alarm.

## 15. Known gaps in this plan

- **Synthetic data only for most tests.** Real service metrics have
  messier shapes than the generators.
- **No long soak test.** Memory and bloat over weeks are untested unless
  a soak run is added.
- **Single Postgres.** Failover behavior is not tested.
- **No network fault injection layer.** Faults are process kills and
  stopped dependencies.
- **Differential testing against Prometheus is optional** and limited by
  the documented `rate` deviation.
- **Redaction is heuristic**, so tests prove patterns, not completeness.
- **No security testing beyond the cases above.**
- **No CI yet.** Wire unit and DB tests first, then integration and
  failpoint suites with a Postgres service.

## 16. Test report (fill in after running)

| Field | Value |
|---|---|
| Date | |
| Commit | |
| Go and Postgres versions | |
| Hardware (CPU, RAM, native or VM) | |
| Unit tests | |
| Database tests | |
| Tier equivalence property (cases run, result) | |
| Ingestion and idempotency | |
| Log oracle property (cases run, result) | |
| Alerting integration | |
| Failpoint suite F1 to F14, pass count | |
| Reconciler injected drift | |
| Benchmarks (with spread) | |
| Storage vs estimates | |
| Anything that failed and what changed | |