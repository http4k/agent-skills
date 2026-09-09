---
module: http4k-postbox
license: http4k Commercial
---

# http4k-postbox Reference

Postbox implements the transactional outbox/inbox pattern: incoming requests are durably stored (in the same transaction as your own business writes, via `Transactional<Postbox>`) instead of being processed inline, then a background processor drains them against a target `HttpHandler` later. This gives idempotent, at-least-once, retryable request handling without losing requests to crashes between "accepted" and "processed". Requires an http4k Pro license.

## Core Interface

```kotlin
interface Postbox {
    fun store(requestId: RequestId, request: Request): Result<RequestProcessingStatus, PostboxError>
    fun status(requestId: RequestId): Result<RequestProcessingStatus, PostboxError>
    fun markProcessed(requestId: RequestId, response: Response): Result<Unit, PostboxError>
    fun markFailed(requestId: RequestId, delayReprocessing: Duration, response: Response?): Result<Unit, PostboxError>
    fun markDead(requestId: RequestId, response: Response? = null): Result<Unit, PostboxError>
    fun pendingRequests(batchSize: Int, atTime: Instant): List<PendingRequest>
    fun claim(batchSize: Int, atTime: Instant, lease: Duration): List<PendingRequest>
}

typealias TransactionalPostbox = Transactional<Postbox>
```

`RequestId` is a `StringValue` (1-64 chars) used as the idempotency key. `RequestProcessingStatus` is `Pending`/`Processing`/`Processed`/`Dead`.

## Wiring an App: `PostboxHandlers`

```kotlin
val transactor: TransactionalPostbox = PostboxTransactor(dataSource)

val handlers = PostboxHandlers(
    transactor,
    responseGenerator = PendingResponseGenerators.linkHeader("requestId") // default: PendingResponseGenerators.Empty (202, no Link)
)

val app = routes(
    "/orders/{requestId}" bind POST to handlers.intercepting(RequestIdResolvers.fromPath("requestId")),
    "/orders/{requestId}" bind GET to handlers.status(RequestIdResolvers.fromPath("requestId"))
)
```

- `intercepting(resolver)` stores the request and returns 202 (or the already-recorded response if it was processed before) — this is what your app exposes to callers.
- `status(resolver)` looks up the current status by id — pending returns the `responseGenerator` output, processed/dead return the stored response.
- `RequestIdResolvers`: `fromPath(name)`, `fromHeader(name)`, or `fromPath()` (whole URI path as the id).
- `PendingResponseGenerators`: `Empty` (plain 202), `linkHeader(pathName)` (202 + `Link` header pointing at the status endpoint), `redirect(pathName)` (302).

## Background Processing: `PostboxProcessing`

```kotlin
val processing = PostboxProcessing(
    transactor,
    target = myDownstreamHandler,       // HttpHandler that actually does the work
    batchSize = 10,
    maxFailures = 3,                    // after this many failures, mark Dead instead of retrying
    maxPollingTime = Duration.ofSeconds(5),
    lease = Duration.ofSeconds(30),     // claim exclusivity window
    events = StdOutEvents,
    successCriteria = { it.status.successful }
)

processing.start()  // runs the poll loop on a background (virtual) thread
processing.stop()   // graceful shutdown, waits up to shutdownGracePeriod for in-flight work
```

Each cycle: `claim()`s a batch of due `PendingRequest`s (atomically marking them `Processing` so other processors skip them), calls `target` for each, then finalises via `markProcessed`/`markFailed`/`markDead` depending on `successCriteria` and `maxFailures`. Failed requests are rescheduled with `backoffStrategy` (default: exponential with jitter — `2^failures * 5s + random(10)s`).

`events` receives `ProcessingEvent`s (`BatchProcessingSucceeded`, `RequestProcessingSucceeded`, `RequestScheduledForRetry`, `RequestMarkedDead`, `RequestProcessingFailed`, `PollWait`, `ShutdownTimedOut`) — wire these to your metrics/logging.

## Storage Backends

```kotlin
// In-memory (tests, or single-process non-durable use)
val postbox = InMemoryPostbox(timeSource)
val transactor: TransactionalPostbox = InMemoryTransactor(postbox, { postbox })

// JDBC (durable; participates in the same DB transaction as your writes)
JdbcPostboxSchema.create(dataSource)  // creates the `<prefix>_postbox` table + index, prefix defaults to "http4k"
val transactor: TransactionalPostbox = PostboxTransactor(dataSource, tablePrefix = "http4k")
```

`PostboxTransactor` wraps `JdbcTransactor` so `transactor.perform { postbox -> ... }` runs against a `Postbox` bound to a single JDBC connection/transaction — use this to make storing a postbox entry part of the same commit as your domain writes.

## ExecutionContext

`PostboxProcessing` delegates its poll/sleep loop to an `ExecutionContext` so it can be tested without real threads or sleeps:

```kotlin
interface ExecutionContext {
    fun isRunning(): Boolean
    fun start(runnable: Runnable)
    fun pause(duration: Duration)
    fun stop(): Boolean          // true if in-flight work finished within the grace period
    fun currentTime(): Instant
}
```

`DefaultExecutionContext(shutdownGracePeriod)` is the production implementation (virtual-thread executor, real `Thread.sleep`). Don't construct `PostboxProcessing` with a custom context in production code — the default is almost always right.

## Testing Patterns

**Contract test** — implement `PostboxContract` against any storage backend to get the full behavioural suite (idempotent store, claim/lease/reclaim semantics, status transitions, response retention rules) for free:

```kotlin
class InMemoryPostboxTest : PostboxContract() {
    val inMemoryPostbox = InMemoryPostbox(timeSource)   // timeSource is a FixedTimeSource provided by the base class
    override val postbox = InMemoryTransactor(inMemoryPostbox, { inMemoryPostbox })
}
```

**Driving `PostboxProcessing` deterministically** — use a `TestExecutionContext(timeSource, iterations)` to run the poll loop exactly `iterations` times then stop, instead of racing against real threads:

```kotlin
val processor = PostboxProcessing(transactor, testTarget, context = TestExecutionContext(timeSource, 1), events = StdOutEvents)
processor.start()
```

**Simulating shutdown timing** — `SimulatedExecutionContext(timeSource, shutdownGracePeriod)` models a busy worker via `thread.busyUntil` without real sleeps, letting you assert `stop()`'s grace-period behavior (`context.finished`, elapsed simulated time) precisely.

**Injecting storage failures** — wrap a real `Postbox` in a test-only decorator (see `PostboxFailureInjector`) that fails the next `store()` call, to exercise the "storage failed while intercepting" path through `PostboxHandlers` without touching production code.

## Gotchas

- **`store` is idempotent by requestId, not by payload**: storing a second, different `Request` under an id already in the postbox is silently ignored — the original stored request wins and its existing status is returned. Don't assume re-POSTing with the same id updates the body.
- **`claim` vs `pendingRequests`**: `pendingRequests` is a read-only, non-mutating peek (for status/monitoring UIs). Only `claim` atomically flips due requests to `Processing` and gives you an exclusive lease — always use `claim` from a processor loop, not `pendingRequests`.
- **Expired leases self-heal**: if a processor claims a batch and then dies before finalising, those requests are simply reclaimed by whichever `claim` call comes after `processAt + lease` — no separate reaper process is needed.
- **`markDead` response retention differs from `markProcessed`/`markFailed`**: `markProcessed`/`markFailed` always overwrite the stored response. `markDead` only fills in a response if none is already stored — calling `markDead` again on an already-dead request with a new response is a no-op if a response was already recorded.
- **Terminal states are terminal**: `markProcessed`/`markFailed`/`markDead` all fail with `PostboxError.RequestAlreadyProcessed` or `RequestMarkedAsDead` once a request has reached `Processed` or `Dead` — there's no way to reopen a finished request.
- **`RequestId` max length is 64 chars** (`RequestId.MAX_LENGTH`); longer values throw `IllegalArgumentException` from `RequestId.of`.
- **`PostboxHandlers.intercepting` needs a matching id resolver on both routes**: the id extracted by the resolver passed to `intercepting` must line up with the one used for `status`, otherwise a status check will 404 even though the request was accepted.
