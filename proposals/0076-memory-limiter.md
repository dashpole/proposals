# Memory Limiter

* **Owners:**
  * @dashpole

* **Implementation Status:** `Partially implemented (proof of concept)`

* **Related Issues and PRs:**
  * https://github.com/prometheus/prometheus/issues/17109
  * https://github.com/prometheus/prometheus/issues/13939
  * https://github.com/prometheus/prometheus/issues/11306
  * https://github.com/prometheus/prometheus/issues/16917
  * https://github.com/prometheus/prometheus/issues/18284

* **Other docs or links:**
  * Promcon 2025 - Scrape Trolley Dilemma talk (credit to @bwplotka)
    * [YouTube Recording](https://www.youtube.com/watch?v=ulHQUCarjjo)
    * [Slides](https://docs.google.com/presentation/d/1jKrUklPdAor9292HrPWtJkIa6ruUhOGo9IFO7fNj-DE/edit?slide=id.p#slide=id.p)
  * Proof of concept: [`dashpole/prometheus@memory_limiter_simple`](https://github.com/dashpole/prometheus/tree/memory_limiter_simple),
    [`dashpole/prometheus@memory_limiter_ai_poc`](https://github.com/dashpole/prometheus/tree/memory_limiter_ai_poc)

> TL;DR: This proposal introduces a Memory Limiter for Prometheus. It measures **live heap
> relative to `GOMEMLIMIT`** and, as that ratio approaches 1, applies mitigations in two
> tiers: first ones that only *delay* work (throttling scrape admission, pausing on-disk
> block compaction, shedding reads), then ones that *discard* work (skipping scrapes,
> rejecting writes, pausing recording rules). The goal is to trade a bounded, observable
> degradation for the unbounded, global outage that an OOM kill causes.

---

<!--
EDITOR'S NOTE — what changed relative to the first draft of this document.
Delete this block before merging; it exists so reviewers who read the earlier revision can
find the deltas without diffing.

1. The measured signal is now specified: `/gc/heap/live:bytes` from `runtime/metrics`, not
   `runtime.MemStats.Alloc`. Rationale in "Choosing the signal"; the rejected options are in
   Alternatives #5.
2. Limits are now expressed as ratios of `GOMEMLIMIT`, not percentages of total memory. The
   old config shape (copied from the OpenTelemetry Collector) could not be set correctly at
   any value — see Alternatives #6.
3. `GOMEMLIMIT` is no longer derived from the limiter's limits. The dependency now runs the
   other way. This removes the ~27-percentage-point silent reduction in effective
   `GOMEMLIMIT` that the previous draft's defaults produced.
4. "Pause compaction" is now scoped to on-disk block compaction only, explicitly excluding
   head compaction, OOO head compaction, and WAL truncation. The previous wording described
   an action that increases memory.
5. "Pause recording rules" moved from the soft tier to the hard tier, and is now
   dependency-aware. The soft/hard boundary is now defined as "does this permanently lose
   data", which also moves remote read to the soft tier.
6. New soft-tier scrape mitigation: byte-budget scrape admission. This delays rather than
   drops, and subsumes what were previously two separate future enhancements ("Gradual
   Degradation" and "Fairness Mechanisms").
7. Skipped scrapes no longer emit per-series staleness markers. The previous design's skip
   path cost one append per series, per target, in the first skipped interval.
8. Added: hysteresis and minimum engagement duration; a defined saturation state and exit
   path; limiter self-observability metrics; remote write receiver and federation as
   in-scope surfaces; "How we test and verify"; Risks; Open Questions.
9. `prometheus_rule_group_iterations_missed_total` is no longer cited as covering paused
   rules — it does not increment in this case.
-->

## Why

Memory exhaustion is a common cause of Prometheus crashes (OOM kills). It can be triggered by:

- Spikes in scrape volume or metric cardinality (e.g. new workloads spun up in Kubernetes).
- Expensive PromQL queries, recording rules, or federation requests.
- High volume of incoming OTLP or remote write traffic, or expensive remote read requests.
- TSDB block compaction, which merges on-disk blocks and holds index structures in memory.

When Prometheus is OOM-killed, the blast radius is total: every target and every consumer
loses monitoring at once, usually at the moment monitoring is most needed. Worse, the
conditions that caused the kill are frequently still present on restart, so the process enters
a crash loop and each restart pays an increasingly expensive WAL replay.

The core asymmetry motivating this proposal: **an OOM kill discards all data from all targets
for an unbounded period, and Prometheus gets no say in it.** Almost any deliberate, bounded,
observable degradation is preferable. Today Prometheus has no way to express that preference.

### Pitfalls of the current solution

Current mitigations are fragmented, static, and per-surface:

- `sample_limit` and `body_size_limit` are per-scrape-config and require knowing target sizes
  in advance. They cannot react to a target that grows.
- `--query.max-samples`, `--query.max-concurrency`,
  `--storage.remote.read-concurrent-limit`, `--storage.remote.read-sample-limit`, and
  `--rules.max-concurrent-evals` each bound one surface in isolation. None of them knows how
  much memory the process is actually using, so they must be provisioned for the worst case
  of every surface simultaneously — which means provisioning for a peak that never occurs, and
  still being wrong when it does.
- `GOMEMLIMIT` (set automatically by `--auto-gomemlimit`) makes the Go GC work harder as
  memory grows, but it cannot shed load. When live heap approaches `GOMEMLIMIT`, Go's only
  remaining option is to spend more CPU collecting, which degrades into a GC death spiral and
  then an OOM anyway.
- Relying on the OS or cgroup boundary guarantees the worst available outcome: SIGKILL, with
  no WAL flush and no graceful shutdown.

There is no mechanism by which Prometheus can notice it is running out of memory and do
something about it.

## Goals

- Prevent OOM kills by shedding load before the process becomes unrecoverable.
- Define a single, global memory pressure signal that all subsystems consult, rather than
  another per-surface limit.
- Distinguish mitigations that **delay** work from mitigations that **discard** it, and reach
  for the former first.
- Let operators enable and disable individual mitigations, because each has a different cost
  profile and different deployments will accept different costs.
- Make it unambiguous, from metrics alone, when the limiter is active, why, and what it did.
- Degrade proportionally: a small, well-behaved target should not be punished for a large
  target's cardinality explosion.

### Audience

Prometheus operators running in memory-constrained environments — most commonly a container
with a fixed memory limit — whose servers scrape targets they do not control.

## Non-Goals

- **Fixing a Prometheus that is genuinely too small for its workload.** If live heap exceeds
  the limit because the head legitimately holds that many series, no amount of load shedding
  helps. This proposal aims to survive *transients* and to *fail visibly and recoverably*
  rather than silently, in the sustained case. See "Saturation".
- **Long-term cardinality growth.** Slow series growth is a different problem with a different
  fix (see Complementary Ideas #3). This proposal protects the live heap against bursts.
- **Memory leaks.** A leak will eventually saturate the limiter; the limiter will report that
  it has, and will not repair it.
- **Fairness and per-job QoS beyond proportional cost.** The byte-budget mechanism below is
  proportional to predicted scrape cost, not to configured priority. Explicit priority is a
  future enhancement.
- **Bounding non-heap memory.** Page cache from mmap'd chunks, goroutine stacks, and Go
  runtime overhead are out of scope; see "What this does and does not bound".

## How

The limiter is a single global component. A background goroutine samples one number every
`check_interval` and publishes a **pressure state** — `ok`, `soft`, `hard`, or `saturated` —
as an atomic value. Subsystems read that value; they never sample memory themselves.

```
                     ┌──────────────────────────────┐
   runtime/metrics ──▶│  memory limiter (1 goroutine)│
   /gc/heap/live      │  every check_interval:       │
   /gc/limiter/...    │    sample → state + hysteresis│
                     └──────────────┬───────────────┘
                                    │ atomic state
        ┌───────────────┬───────────┼────────────┬──────────────┐
        ▼               ▼           ▼            ▼              ▼
   scrape loops    block compactor  rule mgr   OTLP / RW     remote read
   (throttle,       (pause)         (pause     receivers     / federation
    then skip)                       recording  (503)         (503)
                                     rules)
```

### Choosing the signal

This is the load-bearing decision in the design, so it is stated explicitly.

The limiter measures **live heap**, via `runtime/metrics`' `/gc/heap/live:bytes`, and compares
it to **`GOMEMLIMIT`**, read from `/gc/gomemlimit:bytes`. The control variable is the ratio:

```
pressure = live_heap / GOMEMLIMIT
```

Three properties motivate this.

**Live heap, not allocated heap.** Under the default `GOGC=100`, the Go heap deliberately
oscillates to roughly twice the live heap between collections. Comparing instantaneous
allocation (`runtime.MemStats.Alloc`, which is what the proof of concept does) against a fixed
threshold therefore fires on garbage that the next GC will reclaim. Measured against a
synthetic ~205 MiB live heap with scrape-shaped allocation churn, `MemStats.Alloc` swung
between 215 MiB and 509 MiB — a 2.37x range. A threshold set anywhere below ~2.4x live heap
trips permanently. Live heap does not have this problem: it is the quantity the GC itself
pivots on, and it changes only at collection boundaries.

**`GOMEMLIMIT` as the denominator, not total or container memory.** Go treats the entire
`GOMEMLIMIT` budget as available for garbage; memory-in-use approaching `GOMEMLIMIT` is
*normal, healthy operation by design*, not a warning sign. So an absolute byte threshold below
`GOMEMLIMIT` fires constantly during healthy operation, and one at or above `GOMEMLIMIT` never
fires at all, because Go will not let memory get there. There is no correct absolute
threshold. What *is* meaningful is how much of the budget is spoken for by live data:
`live_heap / GOMEMLIMIT` is the fraction of the budget Go cannot reclaim no matter how hard it
tries. As that ratio approaches 1, the effective achievable `GOGC` approaches 0, GC runs
continuously, and the process is dying. The ratio is also scale-free, so a single set of
defaults is meaningful from a 1 GiB Prometheus to a 200 GiB one.

**Sampling is cheap and non-invasive.** `runtime/metrics.Read` costs roughly 0.5–0.9 µs and
does not stop the world. `runtime.ReadMemStats` costs roughly 50–67 µs and does, roughly
independent of heap size. At `check_interval: 1s` neither is expensive in aggregate, but
`runtime/metrics` is two orders of magnitude cheaper, is not a stop-the-world pause, and
exposes live heap directly. There is no reason to prefer `ReadMemStats`.

Because live heap only changes when a GC completes, `check_interval` is an upper bound on
reaction latency rather than a sampling rate; sampling more often than the GC cycle is
harmless but returns the same value.

#### Secondary signal: the Go GC CPU limiter

If `/gc/limiter/last-enabled:gc-cycle` advances, the Go runtime's own GC CPU limiter has
engaged, meaning garbage collection is consuming more than 50% of available CPU. That is an
unambiguous statement, from the runtime, that the process cannot keep up — and it requires no
tuning. When this happens the limiter enters the `hard` state immediately, regardless of the
ratio.

#### What this does and does not bound

The limiter bounds Prometheus's **Go heap**. It does not bound:

- **Page cache from mmap'd head chunks and block files.** These are charged to the cgroup but
  are *reclaimable*: under memory pressure the kernel drops them rather than OOM-killing. A
  limiter that read cgroup `memory.current` would therefore engage constantly on a healthy,
  idle Prometheus. Letting the kernel reclaim file cache while Prometheus bounds its own
  anonymous memory is the correct division of labour, and is why cgroup accounting is rejected
  (Alternatives #5).
- **Go runtime overhead** outside the heap — goroutine stacks, mspan/mcache, GC metadata.
  These are typically 10–20% of heap for a large Prometheus, and they are *inside*
  `GOMEMLIMIT`, so `GOMEMLIMIT` already accounts for them. That is a further argument for
  `GOMEMLIMIT` as the denominator.
- **Memory already committed to in-flight work** at the moment the limiter engages. Overshoot
  past the configured limit is expected; see "Overshoot and headroom".

Consequently the limiter reduces the *probability* of an OOM kill; it does not eliminate it.
Any claim otherwise would be false.

### Pressure states and hysteresis

| State | Entered when | Meaning |
|---|---|---|
| `ok` | `pressure < soft_limit_ratio` | Normal operation. |
| `soft` | `pressure >= soft_limit_ratio` | Apply mitigations that delay work. No data is lost. |
| `hard` | `pressure >= hard_limit_ratio`, or the Go GC CPU limiter engaged | Additionally apply mitigations that discard work. |
| `saturated` | continuously in `hard` for `saturated_after` | Mitigations were insufficient. See "Saturation". |

Two rules prevent flapping, which a bare threshold comparison at `check_interval: 1s` would
otherwise produce at 1 Hz:

- **Asymmetric thresholds.** A state is exited only when `pressure` falls below its entry
  threshold minus `release_hysteresis`.
- **Minimum engagement.** Once entered, a state is held for at least
  `min_engaged_duration`, even if pressure drops immediately.

State is published as a single atomic value. Subsystems perform one atomic load; they do not
take a lock and do not sample memory. This matters on the scrape path, where the proof of
concept's global write mutex would serialize every scrape loop in the process.

### Mitigations

Mitigations are tiered by a single criterion: **does this permanently destroy data that
cannot be recovered?**

- **Soft tier — delays work.** Requests fail in ways clients can retry, or work is deferred
  and catches up. Nothing is permanently lost.
- **Hard tier — discards work.** A gap is created in stored data that nothing will backfill.

This criterion is what determines the tiering, not how disruptive the action feels. Rejecting
a remote read is highly visible but loses nothing; skipping a recording rule evaluation is
invisible and loses data forever.

#### Soft tier

**1. Throttle scrape admission (byte budget).**

Transient scrape memory is roughly `concurrent_scrapes × body_size × parse_amplification`.
Prometheus has no global bound on this today — there is no scrape concurrency or byte limit,
only per-target `body_size_limit`.

In the `soft` state, a scrape loop must reserve budget from a global semaphore before issuing
its HTTP request. The reservation is `lastScrapeSize × parse_amplification`; for a target that
has not yet been scraped it is `body_size_limit` if configured, otherwise a small default.
Total budget is derived from remaining headroom, `(hard_limit_ratio − pressure) × GOMEMLIMIT`.
If the reservation cannot be satisfied, the loop waits, bounded by `scrape_timeout`; on
expiry it proceeds anyway. **In the soft tier a scrape is never dropped, only delayed.**

This is deliberately proportional rather than equal-share. A 5-series target reserves almost
nothing and is therefore almost never delayed, while a 50,000-series target waits for
meaningful headroom. That is the desired behaviour and the reason this replaces the
equal-quantum scheduling schemes explored in the proof of concept (Alternatives #7).

It also provides graceful, continuous degradation for free: as pressure rises the budget
shrinks and scrapes are progressively delayed, rather than all being dropped at one threshold.

**2. Pause on-disk block compaction.**

Scoped precisely: **only the merging of existing on-disk blocks** (`DB.compactBlocks`). Head
compaction, OOO head compaction, and WAL truncation must continue.

This scoping is not a detail. Head compaction is what *frees* head memory, and WAL truncation
is what keeps the WAL — and therefore the next startup's replay cost — bounded. Pausing either
would make memory grow monotonically and would worsen the crash loop this proposal exists to
break. The existing `DB.DisableCompactions()` disables all of it and is therefore **not** the
correct mechanism. `BlockCompactionExcludeFunc` (already used by
`--storage.tsdb.delay-compaction-file`) is the closer precedent.

Note that `compactBlocks` already yields to head compaction when the head becomes compactable,
so pausing it cannot starve head compaction.

**3. Shed read traffic: remote read, federation, and optionally PromQL.**

Remote read (particularly the non-streaming path) and federation both materialize large series
sets in memory and are among the largest single allocations a Prometheus server makes. Both
respond `503` with `Retry-After`.

PromQL queries are covered by the same mechanism but default to **off**, because intermittent
query failures make an incident much harder to diagnose, and because `--query.max-samples`
already bounds per-query memory. Operators who run meta-monitoring (a second Prometheus
watching this one) can safely enable it; operators who debug from the affected server's own UI
should not.

#### Hard tier

**4. Skip scrapes.**

A skipped scrape must **not** be implemented as a failed scrape. Prometheus currently treats a
failed scrape as an *empty* scrape and appends one staleness marker per series the target
previously exposed. Measured against the real scrape loop, a skipped scrape of a 5,000-series
target performs 5,000 appends — the same number as the successful scrape it replaced. The
HTTP body and the parse are avoided; the entire append path, including head chunk writes and
WAL records, is not. Since the limiter's state is global, every target would skip in the same
interval, producing a synchronized `Σ(series)` append burst at the exact moment the process is
closest to OOM. The first act of the mitigation would be to make the spike worse.

Therefore a skipped scrape:

- **Does not** append staleness markers, and **does not** advance the scrape cache. The cache
  is only advanced from within the append path, so "this scrape did not happen" is the natural
  result.
- **Does** report `up = 0` and the usual per-scrape report series, which is a separate,
  bounded append of a handful of samples per target.
- **Does** attach a descriptive error to the target on the `/targets` page.

The consequence is that the target's series carry forward under the 5-minute lookback rather
than reading as absent. **This is intentional**: a skipped scrape means the data is delayed,
not gone, and for the transient case this proposal targets, that is both cheaper and more
truthful. It does mean the "`up = 0` is an unambiguous signal" argument is weaker than it
appears — `up = 0` is emitted, but the target's own metrics do not go stale until lookback
expires. Operators should alert on the limiter's own metrics, not infer memory pressure from
`up`.

**5. Reject writes: OTLP and the remote write receiver.**

Both respond `503` with `Retry-After`, signalling clients to back off. Clients will retry, so
this is only lossy once client-side buffers overflow — but they are finite, so it is treated as
lossy.

Rejection must occur at **handler entry, before the request body is read.** The OTLP handler
currently calls `DecodeOTLPWriteRequest` as its first statement, which reads and unmarshals
the whole body; the OTLP decode plus remote-write conversion is a multiple of the wire size.
Rejecting after that point has already paid the allocation the rejection was meant to avoid.

The remote write receiver (`--web.enable-remote-write-receiver`) has the same memory profile as
OTLP and is included for the same reasons.

**6. Pause recording rule evaluation.**

This is in the hard tier because a missed recording rule evaluation is a **permanent gap in a
derived series**. Nothing backfills it. The evaluator resumes cleanly; the data does not come
back. It was previously classified as non-destructive, which was wrong.

Two constraints:

- **Alerting rules continue to evaluate**, which creates a hazard: an alerting rule reading a
  paused recording rule's output will first evaluate against carried-forward values and appear
  healthy, then — once lookback expires — see an empty result and silently **resolve**.
  Silently resolving alerts during a memory incident is a worse outcome than the OOM. The
  limiter therefore **will not pause a recording rule that has dependent rules**, using the
  dependency information the rule manager already computes (`buildDependencyMap` /
  `isIndependent`).
- Pausing is implemented via the existing `evalIterationFunc` hook, which is the least
  invasive insertion point.

### Saturation

If the limiter remains in `hard` continuously for `saturated_after`, mitigations have failed:
live heap exceeds what the process can support and shedding incoming work is not reducing it.
This is the sustained-growth case that Non-Goals excludes from the *prevention* scope, but it
must still have a defined behaviour, because the alternative is a silent permanent brownout
with no recovery path — invisible to Kubernetes, invisible to liveness probes, and strictly
worse than an OOM kill, which at least restarts the process and is loudly visible.

On entering `saturated`:

1. `prometheus_memory_limiter_state` reports `3`, and a `WARN`-level log line is emitted at a
   bounded rate. This is the signal operators should alert on.
2. The configured `saturation_action` is taken:
   - `none` (default): all mitigations remain engaged. Prometheus stays up, degraded and
     clearly labelled as such. This preserves query availability, which is often what an
     operator most wants during an incident.
   - `release`: mitigations are released and Prometheus is allowed to take the OOM kill. For
     operators who would rather crash and restart than serve a degraded server, and who have
     an orchestrator that will restart it.
   - `shutdown`: Prometheus exits cleanly. This is strictly better than SIGKILL — the WAL is
     flushed and the next startup's replay is cheaper — and is the recommended setting for
     deployments where an OOM crash loop is the failure being fought.

`none` is the default because it is the least surprising, not because it is the best choice for
most deployments. Documentation should say so.

### Overshoot and headroom

The limiter cannot reclaim memory already committed to in-flight work, and cannot force the GC
to return memory to the OS instantly. Overshoot past `hard_limit_ratio` is therefore expected.
The proof of concept observed peak RSS approximately 28% above its configured limit under a
deliberately extreme synthetic load.

Because limits here are ratios of `GOMEMLIMIT`, and `GOMEMLIMIT` already defaults to 90% of the
container limit, the headroom is structural rather than something operators must compute:
`hard_limit_ratio: 0.85` corresponds to roughly 76% of the container limit. Operators who
raise `--auto-gomemlimit.ratio` toward 1.0 lose that headroom and should expect the limiter to
be less effective. This interaction must be documented.

### Configuration

The limiter is configured under the existing `runtime` section, alongside `gogc`, because it
is a process-behaviour knob rather than a data-pipeline knob.

```yaml
runtime:
  memory_limiter:
    # Fraction of GOMEMLIMIT of *live heap* at which non-destructive mitigations engage.
    soft_limit_ratio: 0.70

    # Fraction of GOMEMLIMIT of live heap at which destructive mitigations engage.
    hard_limit_ratio: 0.85

    # Time between measurements. Live heap only changes when a GC completes, so this is an
    # upper bound on reaction latency rather than a sampling rate.
    check_interval: 1s

    # A state is exited only when pressure falls this far below its entry threshold.
    release_hysteresis: 0.05

    # Minimum time a state is held once entered, to prevent flapping.
    min_engaged_duration: 30s

    # Time continuously in the hard state after which the limiter declares saturation.
    saturated_after: 15m

    # What to do on saturation: none | release | shutdown.
    saturation_action: none

    # Optional: use an explicit byte budget instead of GOMEMLIMIT as the denominator.
    # Required if --auto-gomemlimit=false. Prometheus fails to start if neither is available.
    # limit_bytes: 4GiB

    # Assumed transient memory amplification of a scrape body during parse and append,
    # used to size byte-budget reservations. See Open Questions.
    parse_amplification: 4.0

    enforcement:
      # Soft tier.
      throttle_scrapes: true
      pause_block_compaction: true
      reject_remote_read: true
      reject_federation: true
      reject_queries: false          # off by default; see Soft tier #3
      # Hard tier.
      skip_scrapes: true
      reject_otlp: true
      reject_remote_write: true
      pause_recording_rules: true
```

Defaults for `soft_limit_ratio` and `hard_limit_ratio` follow from GC headroom arithmetic: at
`pressure = 0.70` the maximum achievable `GOGC` is about 43, and at `0.85` about 18, at which
point the collector is running nearly continuously. They are starting points and must be
validated empirically before this feature graduates — see Open Questions.

#### Relationship to `GOMEMLIMIT`

`GOMEMLIMIT` is an **input** to the limiter, not an output. Prometheus continues to set it
from `--auto-gomemlimit` / `--auto-gomemlimit.ratio` exactly as it does today, and the limiter
reads the result. Enabling the memory limiter does not change `GOMEMLIMIT`, does not change GC
behaviour, and cannot cause a GC CPU regression.

This is a reversal from the previous draft, which derived `GOMEMLIMIT` from the limiter's soft
limit. That direction had two problems: it silently reduced effective `GOMEMLIMIT` from 90% to
roughly 63% of the container limit under the documented defaults — handing a quarter of the
memory budget to the GC as a side effect of enabling a stability feature — and it created two
sources of truth for `GOMEMLIMIT`, one a startup flag and one a reloadable config value,
requiring a `debug.SetMemoryLimit` call on every SIGHUP.

If `--auto-gomemlimit=false` and no `limit_bytes` is configured, the limiter has no
denominator and Prometheus **fails to start** with a clear error, rather than starting with a
silently inert limiter.

#### Relationship to existing limits

The limiter composes with, and does not replace, `sample_limit`, `body_size_limit`,
`--query.max-samples`, `--query.max-concurrency`,
`--storage.remote.read-{sample,concurrent}-limit`, and `--rules.max-concurrent-evals`. Those
bound the cost of a single unit of work; the limiter bounds the aggregate. Both are needed:
per-unit limits keep any one request from being pathological, and the limiter keeps a
legitimate aggregate from being fatal.

#### Agent mode

In agent mode there is no compaction, no rule evaluation, and no query path, so only the
scrape and write-receiver mitigations apply. The limiter is supported; irrelevant
`enforcement` keys are accepted and ignored, and a warning is logged if one is explicitly set
to `true`.

### Feature Flag

Gated behind `--enable-feature=memory-limiter` while experimental, following the usual
graduation process.

If the flag is absent and a `runtime.memory_limiter` block is present, Prometheus logs a
`WARN` on startup and on each reload stating that the block is being ignored. Silently
accepting memory limits and providing no protection is a failure mode operators cannot
detect.

### Debuggability and User Experience

Three audiences need different things.

**1. The Prometheus operator — "is the limiter active, and is it right?"**

This is the primary audience, and the previous draft's most significant omission was that the
limiter did not observe itself. Given the difficulty of choosing a correct signal, an operator
must be able to see the limiter's own inputs in order to distinguish a true engagement from a
false positive.

| Metric | Type | Purpose |
|---|---|---|
| `prometheus_memory_limiter_state` | gauge | 0 `ok`, 1 `soft`, 2 `hard`, 3 `saturated`. |
| `prometheus_memory_limiter_live_heap_bytes` | gauge | The measured input. |
| `prometheus_memory_limiter_limit_bytes{limit="soft\|hard"}` | gauge | Resolved absolute thresholds. |
| `prometheus_memory_limiter_pressure_ratio` | gauge | `live_heap / GOMEMLIMIT`. |
| `prometheus_memory_limiter_state_seconds_total{state}` | counter | Time spent per state — reveals flapping and slow burns that a gauge scrape misses. |
| `prometheus_memory_limiter_state_transitions_total{from,to}` | counter | Transition counts. |
| `prometheus_memory_limiter_scrape_budget_bytes` | gauge | Current byte-budget size. |

The intended alert is on `prometheus_memory_limiter_state >= 2` sustained, and on
`state == 3` immediately. Both should be in the documentation, since an operator who alerts on
`up == 0` instead will get a storm that tells them nothing.

**2. The application owner — "why is my target not being scraped?"**

- `up` records `0` for a skipped target.
- The `/targets` page shows a descriptive error distinguishing "skipped: server memory limit"
  from "skipped: server memory limit, scrape delayed past timeout".
- New: `prometheus_target_scrapes_skipped_total` — scrapes dropped in the hard tier.
- New: `prometheus_target_scrape_admission_wait_seconds` (histogram) — soft-tier delay, which
  is the signal that throttling is active before anything is dropped.

Distinguishing *this* scrape failure from the many other reasons a scrape can fail remains a
broader gap, tracked in prometheus/prometheus#18284. This proposal does not solve it, but the
new counter means the memory-limiter case at least has an unambiguous signal of its own.

**3. Remote clients.**

`503` with `Retry-After` for OTLP, remote write, remote read, and federation. Counted by the
existing `prometheus_http_requests_total`. Note that remote read clients (Thanos, Grafana)
generally surface a `503` to a human as a failed query; this is intended, and is why the read
mitigations are separately toggleable.

**Corrections to the previous draft's observability claims:**

- `prometheus_rule_group_iterations_missed_total` does **not** cover paused recording rules. It
  increments only when the tick loop falls behind an interval; pausing via a no-op keeps the
  ticker on time and the counter at zero. A new
  `prometheus_rule_group_iterations_skipped_total{reason="memory_limit"}` is required.
- `prometheus_tsdb_compactions_skipped_total` already exists and already means "skipped due to
  disabled auto compaction", so it covers paused block compaction. A new
  `prometheus_tsdb_block_compaction_paused` gauge is added for current state; the previously
  proposed `prometheus_tsdb_compaction_pending_blocks` is dropped as redundant.

### How we test and verify

The previous draft had no verification plan. Given that the central risk is a limiter that
engages when it should not, testing is not optional detail — it is how the design is
falsified.

**The false-positive test is the one that matters most.** A Prometheus at realistic steady-state
utilization (50–60% of `GOMEMLIMIT` in live heap), under continuous scrape load and rule
evaluation, with the limiter enabled at default settings, must remain in state `ok` for the
duration of the run. `prometheus_memory_limiter_state_seconds_total{state="soft"}` must be
zero. A design that cannot pass this is not shippable at any threshold, and both rejected
signals (`MemStats.Alloc`, cgroup `memory.current`) fail it.

Also required:

1. **Unit tests** for state transitions, hysteresis, and minimum engagement, driving the
   sampler from an injected fake so tests are deterministic and do not allocate.
2. **Skip-path test** asserting that a skipped scrape performs `O(1)` appends, not one per
   series. This is a regression test for the specific defect described in Hard tier #4.
3. **Compaction scoping test** asserting that with block compaction paused, head compaction and
   WAL truncation still occur — i.e. that head memory and WAL size remain bounded.
4. **Rule dependency test** asserting that a recording rule with dependents is not paused.
5. **End-to-end OOM survival test.** A container with a fixed memory limit, a target whose
   cardinality steps up sharply, and an assertion that the process survives, plus a record of
   peak RSS relative to the limit to quantify overshoot.
6. **Recovery test.** After the load spike ends, the limiter must return to `ok` and all
   targets to `up == 1` within a bounded time. Measure and document it.
7. **Data-loss accounting.** Compare total samples ingested against an unlimited control run
   on a larger memory budget. The previous proof of concept's control run produced no data, so
   the cost of the feature was never measured. It must be.
8. **prombench** run to characterise CPU and latency overhead in the `ok` state, which should
   be indistinguishable from baseline.

## Risks

1. **A false positive is worse than the disease.** A limiter that engages on a healthy server
   degrades it permanently for no reason, and the degradation (paused rules, skipped scrapes)
   is quiet. This is the dominant risk and the reason for the false-positive test, the
   self-observability metrics, and the `runtime/metrics`-based signal.
2. **Live heap is not RSS.** The limiter can be well inside its limits while the process is
   OOM-killed for non-heap reasons. This is a bounded, documented limitation, not a bug, but it
   will generate bug reports.
3. **Alert storms.** Any mitigation that sets `up = 0` at scale will fire target-down alerts
   across a fleet. Documentation must steer operators to the limiter's own metrics.
4. **Silently resolving alerts** if recording rules with dependents are ever paused. Mitigated
   by the dependency check; a bug there is a silent monitoring failure, so it needs a test.
5. **The mitigations may simply not be enough**, in which case the failure mode is a degraded
   server rather than a crashed one. `saturation_action` makes that a deliberate operator
   choice rather than an accident.
6. **Configuration surface growth.** Twelve new keys, nine of them enforcement toggles. Some
   of this is irreducible — the mitigations genuinely have different costs, as @SuperQ noted —
   but if evidence shows the tiers are always enabled or disabled together, the toggles should
   collapse before graduation.
7. **Complexity in the scrape hot path.** The byte-budget reservation adds an atomic load and,
   under pressure, a wait to every scrape. It must be a no-op in the `ok` state.

## Open Questions

1. **Are `0.70` / `0.85` the right defaults?** They come from GC headroom arithmetic, not
   measurement. The false-positive and OOM-survival tests should set them.
2. **What is `parse_amplification` in practice?** The byte-budget mechanism needs a usable
   estimate of transient memory per scrape body. It plausibly varies by exposition format
   (OpenMetrics vs. protobuf vs. native histograms) enough to need per-format constants, or
   enough that a measured feedback loop beats a constant.
3. **Should skipped scrapes emit staleness markers after some duration?** Carrying values
   forward is right for a transient skip and wrong for a long one. A time-based transition
   would be more truthful but reintroduces the append burst at an unpredictable moment.
4. **Is `none` the right `saturation_action` default?** It preserves query availability but
   also preserves an invisible brownout. `shutdown` is arguably the better default for the
   crash-loop case this proposal targets.
5. **Should the limiter act on WAL replay at startup?** Replay is a common OOM point and no
   mitigation here applies — there are no scrapes yet. This may need to compose with
   prometheus/prometheus#11306 and #13939.
6. **Does the ratio hold at very large heaps?** `live_heap / GOMEMLIMIT` is scale-free in
   theory, but GC mark cost scales with the object graph, so a 200 GiB head may enter the death
   spiral at a lower ratio than a 2 GiB one. Worth measuring before graduation.

## Alternatives

1. **Do nothing.** The status quo: OOM kills with total loss of monitoring, and crash loops
   whose WAL replay cost grows with each iteration. The linked issues show this is a recurring
   operational problem, and @bboreham noted at the dev summit that Mimir has a similar
   mechanism in production, so the general approach has real-world backing.

2. **Reject only new series** ([#16917](https://github.com/prometheus/prometheus/issues/16917),
   [PR #11124](https://github.com/prometheus/prometheus/pull/11124)). Accept updates for known
   series, reject allocation of new ones. Rejected because it violates scrape transactionality:
   a scrape should be ingested in full or not at all. Partial ingestion produces query skew — a
   success-rate query where the success metric is ingested but a newly created error metric is
   dropped reads as 100% healthy — which is a worse failure than missing data, because it is
   confidently wrong. Whole-scrape failure is also the established Prometheus response to an
   over-large scrape (`sample_limit`), so skipping follows precedent rather than inventing a
   new behaviour. Note that new series is nevertheless the right signal for *long-term* growth;
   see Complementary Ideas #3.

3. **Slow down scrapes** — dynamically back off the scrape interval for targets under pressure.
   Rejected as a *reconfiguration* of the interval, because that silently breaks assumptions
   users have built into alerts and recording rules. Note, however, that the byte-budget
   mechanism in Soft tier #1 is a bounded, sub-interval form of this idea: it delays a scrape
   by up to `scrape_timeout` without changing the advertised interval. That is the useful part
   of this alternative, and it is adopted.

4. **Reject only PromQL queries, and nothing else.** Queries are a real source of allocation
   churn, and @SuperQ observed that pausing them is recoverable in a way that dropping scrapes
   is not. Rejected as the *sole* mitigation because ingestion-driven growth is the more common
   OOM cause, and because intermittent query failures blind the operator during an incident.
   Adopted as an opt-in soft-tier mitigation instead.

5. **Other signals considered and rejected:**
   - `runtime.MemStats.Alloc` (what the proof of concept uses): oscillates to ~2.4x live heap
     under default `GOGC`, so any threshold below that multiple fires permanently. Also
     requires a stop-the-world `ReadMemStats`.
   - **Process RSS** from `/proc`: includes reclaimable page cache from mmap'd chunks, so it
     over-reads on a healthy Prometheus, and the kernel would reclaim that memory rather than
     OOM.
   - **cgroup `memory.current`**: same over-reading problem, for the same reason.
     `memory.current − inactive_file` is closer but still couples the limiter to cgroup v1/v2
     differences for a quantity Go already reports directly.
   - **cgroup PSI (`memory.pressure`)**: attractive because it measures the thing we actually
     care about — imminent reclaim failure — across heap and non-heap alike, and needs no
     tuning. Not chosen as the primary signal because it is Linux-only and cgroup-v2-only,
     which makes it unusable as *the* signal for a portable server, and because it indicates
     pressure without indicating headroom, so it cannot size the scrape byte budget. It is a
     good candidate for a future additional trigger.
   - **`GOGC` / GC CPU fraction** as the primary signal: self-calibrating and needs no
     thresholds, but it is a lagging indicator — by the time GC CPU is high, the process is
     already in trouble. Adopted as a secondary immediate-`hard` trigger via
     `/gc/limiter/last-enabled:gc-cycle`, which is the runtime's own unambiguous distress
     signal.

6. **Copy the OpenTelemetry Collector's `memory_limiter` configuration shape**
   (`limit_mib` / `spike_limit_mib` / `limit_percentage` / `spike_limit_percentage`). This was
   the previous draft's design, and it is rejected. Absolute byte limits and
   percentages-of-total-memory are not commensurable with `GOMEMLIMIT`: below it they fire
   during healthy operation, at or above it they never fire. Familiarity to Collector users is
   a real benefit and it is being given up deliberately. Prometheus differs from the Collector
   in ways that matter here — it has a large, long-lived heap rather than a mostly-transient
   one, and it sets `GOMEMLIMIT` by default — and the config should reflect that rather than
   inherit a shape that does not fit.

7. **Equal-share scheduling for scrape fairness** — probabilistic drop, token bucket, or
   Deficit Round Robin, all three of which were prototyped. Rejected in favour of the
   proportional byte budget. The prototype's own data does not support the fairness claim: DRR
   admitted roughly twice as much from the large target as the probabilistic strategy did, so
   the improvement in the small target's uptime (10% → 31%) largely reflects higher total
   throughput under the same limit rather than better isolation; the small target's *share*
   improved only about 1.6x. More decisively, 31% uptime for a five-series target is a design
   failure rather than a qualified success — a five-series target costs approximately nothing,
   so there is no memory justification for ever dropping it. Equal *shares* is the wrong
   objective; the right one is "shed the expensive, keep the cheap", which is what a
   cost-proportional budget does directly and with far less machinery. Weighted DRR remains
   the natural mechanism if and when explicit *priority* (as opposed to cost) is added.

8. **Independent `GOMEMLIMIT` configuration** — keep the limiter's limits and `GOMEMLIMIT`
   entirely separate. This is now what the proposal does, in the sense that the limiter reads
   `GOMEMLIMIT` and never writes it. The previous draft rejected this on the grounds that a
   `GOMEMLIMIT` above the limiter's limit is not something users would want; that reasoning
   assumed the two were comparable quantities, which they are not — one is a ceiling on total
   in-use memory, the other a threshold on live heap.

### Complementary Ideas

Compatible with this proposal; each addresses a source of memory pressure that a limiter does
not.

1. **Automated WAL deletion on OOM** ([#13939](https://github.com/prometheus/prometheus/issues/13939)):
   deleting the WAL when recovering from an OOM. Reactive — it lets the server eventually start
   again, but only after a crash and at the cost of recent data. Complementary because the
   limiter cannot help during replay.
2. **Force head compaction / WAL truncation before scraping** ([#11306](https://github.com/prometheus/prometheus/issues/11306)):
   pause scraping on startup until replay and compaction complete. Breaks a specific startup
   crash cycle that this proposal does not address, for the same reason.
3. **Limit label churn / new series over time** ([#17109](https://github.com/prometheus/prometheus/issues/17109)):
   a per-job limit on how many *new* series a target may introduce over a window. This is the
   correct tool for the long-term growth this proposal lists as a Non-Goal. The two are
   genuinely orthogonal: this proposal protects the live heap against bursts, a churn limiter
   protects the TSDB against slow growth. Together they cover both axes; separately, neither
   does.
4. **Early head compaction under pressure.** Proactively triggering head compaction to flush
   the head and free memory. Not adopted here because the driver of sudden-growth OOMs is new
   series, which would immediately re-grow the head — but it is a plausible
   `saturation_action` and deserves its own evaluation rather than being dismissed. Note that
   this points the *opposite* direction from Soft tier #2, which is precisely why that
   mitigation is scoped to on-disk block compaction only.
5. **Cardinality pre-flight on scrape** (@yeya24): extend `/metrics` to answer a `HEAD`
   request with the target's series count, so the server can decide before paying for the body.
   Attractive but requires ecosystem-wide client library changes; the byte budget approximates
   it using `lastScrapeSize`, which needs no cooperation from targets.
6. **Out-of-order backfill of skipped scrapes** (@yeya24): if a skipped scrape could be
   retried and ingested out of order, skipping would stop being lossy and could move to the
   soft tier. Promising, and it would meaningfully change this proposal's tiering, but it
   requires buffering the very data whose memory cost caused the skip — so it needs its own
   design.

## Action Plan

Staged so that each stage is independently reviewable and shippable. Stage 1 is entirely about
the correctness of the controller; every later stage is one independent policy decision about
one actuator. The previous draft interleaved these, which made the whole design hard to
evaluate at once.

**Stage 1 — the controller, and one mitigation.**

* [ ] Finalize the signal, the denominator, and the state machine (this document)
* [ ] `runtime.memory_limiter` configuration, validation, and reload behaviour
* [ ] Feature flag `--enable-feature=memory-limiter`, with a warning when the block is present
      but the flag is not
* [ ] Sampler on `runtime/metrics`, with hysteresis and minimum engagement, publishing an
      atomic state
* [ ] Limiter self-observability metrics
* [ ] Byte-budget scrape admission (soft tier)
* [ ] Skip scrapes (hard tier), with an `O(1)` skip path and
      `prometheus_target_scrapes_skipped_total`
* [ ] False-positive test, skip-path regression test, recovery test
* [ ] Documentation, including the alerting guidance and the
      `--auto-gomemlimit.ratio` interaction

**Stage 2 — deferrable mitigations (no data loss).**

* [ ] Pause on-disk block compaction only, plus `prometheus_tsdb_block_compaction_paused` and
      the scoping test
* [ ] Reject remote read and federation with `503` + `Retry-After`
* [ ] Optional PromQL rejection, default off

**Stage 3 — lossy mitigations.**

* [ ] Reject OTLP at handler entry, before the body is read
* [ ] Reject remote write receiver at handler entry
* [ ] Pause independent recording rules, with the dependency check,
      `prometheus_rule_group_iterations_skipped_total`, and the dependency test

**Stage 4 — saturation and graduation.**

* [ ] `saturation_action` (`none` / `release` / `shutdown`)
* [ ] End-to-end OOM survival test, data-loss accounting, prombench characterisation
* [ ] Revisit defaults and collapse enforcement toggles based on the above
* [ ] Graduation review

**Future enhancements (explicitly out of scope for the above).**

* [ ] Per-job overrides and explicit priority / criticality metadata, via weighted DRR
* [ ] cgroup PSI as an additional trigger
* [ ] Non-heap accounting, if the heap-only bound proves insufficient in practice
