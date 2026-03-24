# http4k-pro-wiretap Reference

Wiretap is an http4k Pro module — a development console that wraps an HTTP application, recording all traffic and OpenTelemetry spans so you can monitor, inspect, and test what happens inside your app in real time. Requires an http4k Pro license.

## Wrapping a Remote App (Proxy Mode)

```kotlin
// Point Wiretap at an already-running app
val wiretap = Wiretap(
    RemoteTarget(Uri.of("http://localhost:8080"))
)

// Start the Wiretap console (default port: 21000)
wiretap.asServer(Jetty(21000)).start()
// Open: http://localhost:21000/_wiretap
```

## Wrapping a Local App

```kotlin
// LocalTarget wraps your HttpHandler directly — no real server needed
val wiretap = Wiretap(
    LocalTarget { myApp() }
)
wiretap.asServer(Jetty(21000)).start()
```

`LocalTarget` vs `RemoteTarget`:
- `LocalTarget` — wraps an in-process `HttpHandler`; no network hop
- `RemoteTarget` — proxies to a real server URI; auto-starts the server

## Context in LocalTarget / RemoteTarget

Both target types receive a `Context` that provides test-friendly collaborators:

```kotlin
val wiretap = Wiretap(
    LocalTarget {
        val outboundClient = http()          // HttpHandler with outbound recording filter applied
        val clock = clock()                  // deterministic clock
        val otel = otel("my-service")        // OpenTelemetry instance wired to Wiretap stores
        MyApp(outboundClient, clock, otel)
    }
)
```

Use `ctx.http()` (not `JavaHttpClient()` directly) so outbound calls are recorded.
Use `ctx.otel(serviceName)` so spans appear in the Wiretap trace view.

## Full Configuration

```kotlin
val wiretap = Wiretap(
    target = RemoteTarget(Uri.of("http://localhost:8080")),
    transactionStore = TransactionStore.InMemory(),  // default
    traceStore = TraceStore.InMemory(),              // default
    security = BearerAuthSecurity { it == "secret" },// protect the console UI
    mcpOptions = WiretapMcpServerOptions(
        security = BearerAuthMcpSecurity { it == "secret" },
        path = "/mcp"                                // MCP endpoint path
    ),
    sanitise = { tx ->
        // return null to drop a transaction, or modify before recording
        if (tx.request.uri.path.startsWith("/health")) null else tx
    },
    bodyHydration = BodyHydration.All,  // record full bodies; use None for large payloads
)
```

## JUnit Integration: `Intercept`

`Intercept` is a JUnit 5 extension that wires up Wiretap for unit/integration tests. It records traffic and telemetry, and generates an HTML report on test failure.

### Quick setup with an existing HttpHandler

```kotlin
class MyTest {
    private val app = routes("/" bind GET to { Response(OK).body("hello") })

    @RegisterExtension
    @JvmField
    val intercept = Intercept(app)

    @Test
    fun `test passes the HttpHandler as a parameter`(http: HttpHandler) {
        val response = http(Request(GET, "/"))
        assertThat(response.status, equalTo(OK))
    }
}
```

### Full setup with Context (recommended for production apps)

```kotlin
class MyTest {
    @RegisterExtension
    @JvmField
    val intercept = Intercept {
        // `this` is a Context — provides http(), otel(), clock(), random()
        val outbound: HttpHandler = { Response(OK) }
        MyApp(http(outbound), otel("my-service"))
    }

    @Test
    fun `spans are recorded`(http: HttpHandler) {
        http(Request(GET, "/"))
        assertThat(intercept.traceStore.traces(Ascending).size, greaterThan(0))
    }
}
```

### Poly/MCP apps

```kotlin
@RegisterExtension
@JvmField
val intercept = Intercept.poly {
    // wrap a PolyHandler (HTTP + SSE/WebSocket)
    myPolyApp(http(), otel("my-service"))
}

@Test
fun `test gets McpClient`(client: McpClient) {
    // McpClient resolves to an in-process client hitting /mcp
    val tools = client.tools().list()
}
```

### RenderMode

```kotlin
Intercept(app)                      // OnFailure (default) — HTML report only on test failure
Intercept(app, RenderMode.Always)   // Always generate report
Intercept(app, RenderMode.Never)    // Never generate report
```

Reports are written to a temp directory. The path is printed to stdout after each test.

### Accessing recorded data in tests

```kotlin
intercept.traceStore.traces(Ascending)     // Map<TraceId, List<SpanData>>
intercept.transactionStore.list(Descending) // List<WiretapTransaction> with Direction
intercept.logStore                          // log records per span
intercept.capturedStdOut                    // stdout captured during the test
intercept.capturedStdErr                    // stderr captured during the test
```

`WiretapTransaction.direction` is `Direction.Inbound` (server received) or `Direction.Outbound` (client sent).

## Gotchas

- **Pro license required**: `http4k-pro-wiretap` requires an http4k Pro commercial license.
- **Use `ctx.http()` for outbound clients**: Passing `JavaHttpClient()` directly bypasses outbound traffic recording. Always use `http()` from `Context`.
- **Use `ctx.otel(name)` not `GlobalOpenTelemetry`**: `Intercept` resets `GlobalOpenTelemetry` before each test and injects a Wiretap-backed instance. Using `ctx.otel()` ensures spans are captured in `traceStore`.
- **`bodyHydration = BodyHydration.All` buffers bodies**: For streaming or large payloads, use `BodyHydration.None` to avoid memory pressure.
- **MCP auto-detection**: `RemoteTarget` checks `GET {mcpPath}` to detect whether the target supports MCP. Wiretap exposes MCP tools at `/_wiretap/mcp` regardless.
- **`sanitise` can drop transactions**: Return `null` from `sanitise` to exclude a transaction from recording (e.g., health checks, static assets).
- **Wiretap console at `/_wiretap`**: The UI is mounted under `/_wiretap`; all other paths proxy to the target.
