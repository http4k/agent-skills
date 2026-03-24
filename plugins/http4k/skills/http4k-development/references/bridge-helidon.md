---
license: Apache-2.0
module: http4k-bridge-helidon
---

# http4k-bridge-helidon Reference

Bridge http4k handlers into Helidon SE, with support for HTTP, SSE, and WebSocket.

## HTTP Handler

```kotlin
val handler = HelidonToHttp4kHandler(http = myApp, sse = null)
```

## SSE Handler

```kotlin
val handler = HelidonToHttp4kHandler(http = myApp, sse = mySseHandler)
```

Automatically detects SSE requests (via Accept header) and routes to the SSE handler.

## WebSocket Listener

```kotlin
val listener = HelidonToHttp4kWebSocketListener(myWsHandler)
```

## Gotchas

- **Dual-mode**: Supports both HTTP and SSE through the same handler, detecting the mode from the request.
- **SSE lifecycle**: Uses a `CountDownLatch` to synchronize the SSE consumer lifecycle with Helidon's `SseSink`.
- **Method validation**: Unsupported HTTP methods return `501 NOT_IMPLEMENTED`.
