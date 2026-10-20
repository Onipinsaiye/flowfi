# Indexer Operations Guide

This document describes the externally observable behavior of the indexer's
lease coordination, the database indexes backing stream/event reads, the
admin replay preview/execute contract, the Soroban RPC retry/backoff policy,
graceful shutdown and poll cancellation, and the indexer metrics. It is
written for operators and on-call engineers. All public interfaces described
here are additive; existing endpoints and configuration keys keep working
unless noted.

---

## 1. Poll ownership lease

### Behavior

The indexer uses a single renewable lease stored in the database to guarantee
that **at most one backend replica actively polls and advances the cursor** at
any time.

- **Single owner.** A worker acquires the lease before polling. Only the
  current lease owner may advance the indexer cursor. Non-owners idle and
  periodically attempt acquisition.
- **Owner identity.** Each worker has a unique owner ID (hostname + process
  identity + random suffix). The lease records the current owner ID and a
  monotonically increasing **fencing token** (generation number).
- **TTL and renewal.** The lease has a finite TTL. The owner renews it on a
  heartbeat well before expiry. If renewal fails (e.g., DB unavailable), the
  worker stops advancing the cursor and releases ownership locally.
- **Takeover.** If the lease expires without renewal (owner crash, network
  partition, or shutdown), another worker may acquire it. Acquisition
  increments the fencing token.
- **Fencing.** Every cursor write is conditional on the writer still holding
  the lease at the current fencing token. A stale owner whose lease has been
  taken over will have its writes **rejected** rather than overwriting the new
  owner's progress. Rejected writes are logged and counted (see metrics).

### Externally observable effects

- At most one replica advances the cursor at a time; the cursor is monotonic
  per fencing generation.
- After an owner is lost, another replica takes over within roughly one TTL
  plus the acquisition retry interval.
- Stale writes from a former owner are discarded; they never move the cursor
  backwards or skip ledgers.

### Configuration

| Key | Meaning | Notes |
| --- | --- | --- |
| `INDEXER_LEASE_TTL_MS` | Lease time-to-live | Defaults are safe; must exceed renewal interval |
| `INDEXER_LEASE_RENEW_INTERVAL_MS` | Renewal heartbeat interval | Must be well below TTL |
| `INDEXER_LEASE_ACQUIRE_RETRY_MS` | Retry interval for non-owners | Controls takeover latency |

### Rollout and migration

- The lease table is created by migration and is additive; no existing data is
  rewritten.
- **Rolling upgrade:** during a mixed-version rollout, older replicas that do
  not participate in the lease must not run concurrently with new replicas
  against the same cursor. Recommended sequence:
  1. Apply the migration.
  2. Scale the indexer to a single replica (or stop old replicas).
  3. Deploy the lease-aware version.
  4. Scale back up; the lease ensures a single active owner.
- **Rollback:** the lease table may remain in place; older versions ignore it.
  Ensure only one replica runs if rolling back to a non-lease version.

---

## 2. Database indexes for stream and event reads

### Added indexes

The following indexes were added after profiling representative PostgreSQL
query plans for list, history, analytics, and cursor queries. Each is
migration-safe (created with `CREATE INDEX`; no data rewrite) and preserves
result ordering and pagination semantics.

| Index | Table | Columns | Hot path served |
| --- | --- | --- | --- |
| `streams_created_at_id_idx` | `streams` | `(created_at DESC, id DESC)` | Stream list ordering + keyset pagination |
| `streams_sender_idx` | `streams` | `(sender, created_at DESC)` | Per-wallet stream history |
| `events_stream_id_ledger_idx` | `events` | `(stream_id, ledger DESC)` | Per-stream event history |
| `events_ledger_idx` | `events` | `(ledger)` | Cursor scan / ledger range reads |
| `events_type_ledger_idx` | `events` | `(type, ledger DESC)` | Analytics by event type |

### Query-plan evidence

Before/after `EXPLAIN (ANALYZE, BUFFERS)` was captured for each hot path on a
representative dataset. Summary:

- **Stream list (keyset pagination):** before — sequential scan + sort;
  after — index scan on `streams_created_at_id_idx`, no sort node, stable
  ordering by `(created_at, id)`.
- **Per-wallet history:** before — filter + sort over `streams`; after —
  index scan on `streams_sender_idx`.
- **Per-stream events:** before — filter on `stream_id` then sort; after —
  index scan on `events_stream_id_ledger_idx`.
- **Cursor scan / ledger range:** before — sequential scan; after — index
  scan on `events_ledger_idx`.
- **Analytics by type:** before — filter + sort; after — index scan on
  `events_type_ledger_idx`.

### Operational impact

- Index creation takes a brief lock; on large tables run during a low-traffic
  window or use the provided migration which creates indexes without blocking
  writes where supported.
- Additional write amplification on `events` and `streams` inserts is modest
  and was measured within acceptable bounds.
- No public query interfaces changed; ordering and pagination are preserved.

---

## 3. Admin replay: preview and execution

### Contract

Replay is exposed under the existing admin replay controls. Two operations:

- **Preview** (`preview`): computes the affected ledger range and the expected
  record changes **without performing any writes**. Safe to call repeatedly.
- **Execute** (`execute`): performs the replay and returns an execution report.
  Supports retry.

Both accept the same replay parameters as before (ledger range / target). The
response is a **report** with the following fields:

| Field | Description |
| --- | --- |
| `requestId` | Correlation ID for this preview/execute call |
| `mode` | `preview` or `execute` |
| `ledgerRange.from` / `ledgerRange.to` | Affected ledger range (inclusive) |
| `expectedChanges` | Preview only: counts of records that would be created/updated/deleted |
| `counts` | Execute only: records created/updated/deleted/skipped |
| `durationMs` | Wall-clock duration of the operation |
| `status` | `succeeded` / `failed` / `partial` |
| `failurePosition` | On failure: the ledger at which execution stopped (null on success) |
| `liveCursor` | The live indexer cursor after the operation |

### Guarantees

- **Preview performs no writes.** It is read-only and reproducible.
- **Execution reports are reproducible** for the same inputs and starting
  state; retries produce a fresh `requestId` and report.
- **Live cursor is preserved on failure.** If execution fails partway, the
  live cursor is not advanced past the last successfully applied ledger, and
  `failurePosition` identifies where it stopped. Retry resumes from there.

### Operational notes

- Preview before execute to confirm the affected range and expected changes.
- On `status: failed` or `partial`, inspect `failurePosition` and `counts`
  before retrying.
- No existing admin endpoints were removed; the report is additive to the
  replay response.

---

## 4. Soroban RPC retry and backoff policy

### Behavior

All Soroban RPC calls made by the indexer polling path go through a single
centralized retry/backoff policy. Ad-hoc per-call retry loops are replaced by
this policy so classification, bounds, and diagnostics are consistent.

- **Error classification.** Each failure is classified before deciding to
  retry:
  - **Retryable:** transient network failures (connection reset, timeout,
    DNS/transport errors), rate-limit responses (HTTP `429`), and server
    errors (HTTP `5xx`).
  - **Permanent (fail fast):** decoding failures (malformed/unexpected
    response payloads) and authorization failures (HTTP `401`/`403`). These
    are surfaced immediately without consuming retry budget.
- **Bounded attempts.** The policy caps the number of attempts per RPC call.
  Once exhausted, the call fails with a structured error rather than
  retrying indefinitely.
- **Exponential backoff with jitter.** Retries use exponential backoff with
  randomized jitter to avoid synchronized retry storms across replicas.
- **Cancellation-aware.** Retry waits observe the worker's shutdown signal;
  a shutdown aborts pending backoff sleeps and in-flight RPC calls promptly
  instead of waiting out the full retry budget.

---

## 5. Graceful shutdown and poll cancellation

### Behavior

The indexer handles `SIGTERM` and `SIGINT` by initiating a **graceful
shutdown** rather than terminating mid-batch. A single cancellation signal is
shared by the poll loop and the Soroban event worker.

- **Stop acquiring new ranges.** Once shutdown is requested, the poll loop
  stops acquiring new ledger ranges. It finishes (or safely abandons) the
  in-flight batch and then exits cleanly.
- **Cancellation-aware RPC fetch.** If shutdown occurs during an RPC fetch,
  the fetch is aborted and the batch is abandoned **without advancing the
  checkpoint**. The next start re-fetches the same range.
- **Cancellation-aware persistence.** If shutdown occurs during database
  persistence, the checkpoint is advanced **only after the transaction
  commits**. A batch that is rolled back leaves the cursor unchanged, so the
  next start re-processes the same range.
- **Idempotent restart.** Re-processing an abandoned range relies on existing
  unique constraints / upserts, so a restart after an abandoned batch does not
  produce duplicate side effects.

### Externally observable effects

- A shutdown during RPC fetch or during persistence restarts without skipped
  ledgers or duplicate side effects.
- The persisted cursor only ever reflects committed progress; it never points
  past a batch that was not fully committed.
- Shutdown completes within the configured grace period; in-flight work is
  abandoned safely rather than left half-applied.

### Operational notes

- Send `SIGTERM` (the default for orchestrator scale-down) and allow the grace
  period to elapse before `SIGKILL`.
- No configuration keys or public interfaces changed; shutdown behavior is
  additive and requires no migration.
- **Rollout:** safe to deploy in a rolling fashion; each replica drains its
  in-flight batch on shutdown and the lease ensures a single active owner.

---

## 6. Indexer metrics

Metrics are emitted for lease acquisition/renewal, cursor advances, rejected
stale writes, RPC retries by classification, and shutdown events (requested,
completed, and batches abandoned). Use these to alert on repeated stale-write
rejections, sustained RPC retry exhaustion, or shutdowns that do not complete
within the grace period.
