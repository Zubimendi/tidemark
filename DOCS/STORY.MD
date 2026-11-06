# The story: Tidemark

## Short version (bio line or post opener)

Built the last project in my backend series: a metrics and log platform
in Go and Postgres that downsamples history as it ages, answers queries
in its own small language, and alerts correctly even when the data it
watches disappears. Then I pointed it at everything else I built.

## Longer version (LinkedIn or Medium)

I spent a long stretch building backend systems one pattern at a time:
a ledger that cannot go unbalanced, a saga that survives crashes, a
dispatcher that never assigns one driver twice. The last project had to
answer a question all of them raised: how do I know they are working?

Tidemark is the answer. It ingests metrics and logs from other services,
stores them, answers queries, and alerts. The interesting parts were not
the ones I expected.

Storage first. A year of 15 second samples for 10,000 series is about
two terabytes of rows. So the system keeps full detail for two days,
five minute summaries for a month, and hourly summaries for over a year.
The part that took the most care was making old answers correct. A
counter that resets, a window that crosses a bucket boundary, a late
sample arriving after its bucket was summarized: each one can quietly
produce a wrong number. I defined counter semantics once, made the
summaries exact for aligned windows, and wrote a test that checks the
same question against raw data, the five minute tier, and the hourly
tier and demands the same answer. When a query asks for finer detail
than a tier can give, the response says so instead of pretending.

Then alerting, where the hardest problem is silence. If a service dies,
its metrics stop, and a rule like "alert when up equals zero" returns
nothing, so it never fires on the one failure it was written for.
Tidemark's agent writes an explicit up value for every scrape, rules can
test for absence, and a failed query never counts as "everything is
fine". The evaluator itself is a pure function over a tick time, which
means I can backtest a rule over last week and see when it would have
fired.

And the watcher needs a watcher. A heartbeat rule notifies a tiny
separate process every minute, and that process raises its own alarm
when the heartbeats stop.

The capstone is the demo: I killed a worker in one of my earlier
projects and watched the alert go pending, then firing, deliver a signed
notification, show the last error logs for that service, and resolve
when I brought it back.

[Add measured results after running the suites: sustained ingest rate,
query latencies by tier, rollup equivalence test results, and the
failpoint pass count. Do not publish targets as results.]

## Interview talking points

- **"Why Postgres instead of a time-series database?"** To build the
  hard parts myself: partitioning, rollups, retention. I can also say
  where it stops (above roughly 100k samples/s or very high
  cardinality) and I put a storage interface in so a real TSDB could
  replace it.

- **"How do you downsample counters without getting wrong rates?"**
  Each bucket stores the reset-adjusted increase inside itself, plus
  first and last values. Boundary deltas are computed at query time from
  neighbouring buckets. That keeps buckets independent, so late data
  never cascades, and aligned windows give identical answers from every
  tier.

- **"What happens to data that arrives late?"** Ingestion notes which
  already-summarized buckets it touched, and the rollup job recomputes
  them. Rollup is an idempotent function of raw data, so recomputing is
  always safe.

- **"How do you make sure retention never deletes data that has no
  summary yet?"** Raw partitions are dropped only when the rollup
  watermark has passed them. If rollups stall, disk grows and an alert
  fires, which is the better failure.

- **"How do alerts avoid flapping and silence?"** `for` and
  `keep_firing_for` for hysteresis, explicit `up` and `absent` for
  missing data, a no-data policy, and query errors that leave state
  untouched.

- **"What guarantees does ingestion give?"** At-least-once with
  idempotent writes, so retries are always safe, and an accounting
  identity: received equals accepted plus duplicates plus rejected, by
  reason.

- **"How do you search logs on Postgres?"** Tokenize at write time, GIN
  index on the tokens, then verify candidates with the exact predicate.
  The index can return extra rows but never miss any, and I test that
  against a brute-force oracle.

- **"What is explicitly not done?"** Multi-tenancy, tracing, recording
  rules, Prometheus compatibility, HA. All listed with reasons in the
  architecture doc.

## The series at a glance (closing post)

| Project | Pattern it proved | How Tidemark uses it |
|---|---|---|
| LedgerLine | Invariants and reconciliation | Rollups verified against raw data |
| Dispatcher | Outbox and delivery | Alert notifications |
| StatusForge | Incident state machine | Alert instance lifecycle |
| Tessera | Pure transition functions | Alert `Step` function |
| Counterweight | Crash-safe workflows | Failpoint test suite |
| VaultKeep | Hashed credentials | API keys |
| SearchCraft | Search over a relational store | Log index-then-verify |

Each project made the next one cheaper to design. The last one made the
rest observable.