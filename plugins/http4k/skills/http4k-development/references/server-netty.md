# http4k-server-netty Reference

Netty server backend. Supports HTTP and WebSocket. Does NOT support SSE.

## Construction

```kotlin
val server = app.asServer(Netty(8000)).start()

// With explicit graceful shutdown timeout
val server = app.asServer(Netty(8000, StopMode.Graceful(Duration.ofSeconds(10)))).start()
```

Default stop mode is `Graceful(5 seconds)`. Default port is `8000`.

## WebSocket

Netty implements `PolyServerConfig` with WebSocket support:

```kotlin
Netty(8000).toServer(http, ws).start()
```

## Stop Modes

| Mode | Supported |
|---|---|
| `StopMode.Immediate` | No — throws `UnsupportedStopMode` |
| `StopMode.Graceful(duration)` | Yes (default, 5s) |

## Gotchas

- **Graceful only**: Netty throws `ServerConfig.UnsupportedStopMode` if you pass `StopMode.Immediate`. This is enforced in the constructor.
- **No SSE**: Passing an `SseHandler` throws `UnsupportedOperationException("Netty does not support sse")`.
- **No HTTP/2**: The stock Netty config does not include HTTP/2 codec setup.
- **Aggregates full request body**: Uses `HttpObjectAggregator(Int.MAX_VALUE)` — the entire request body is buffered in memory before the handler runs.
- **WebSocket backpressure**: The WebSocket channel handler uses a fixed-capacity buffer (1000 frames). When the buffer is full, auto-read is disabled on the Netty channel (backpressure). Auto-read resumes when the buffer drains below 50%. This prevents memory exhaustion from fast producers.
- **WebSocket message ordering**: Messages are always buffered and drained with an exclusive lock on the app executor — order is guaranteed even under concurrent frame arrival.
