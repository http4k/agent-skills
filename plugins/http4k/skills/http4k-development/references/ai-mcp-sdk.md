---
license: http4k Commercial
module: http4k-ai-mcp-sdk
---

# http4k-ai-mcp-sdk Reference

MCP server implementation — expose tools, resources, and prompts over multiple transports.

## Server Setup

```kotlin
val tools = ServerTools(listOf(
    ToolCapability(myTool, myToolHandler),
    ToolCapability(otherTool, otherHandler)
))

val resources = ServerResources(listOf(
    ResourceCapability(myResource, myResourceHandler)
))

val prompts = ServerPrompts(listOf(
    PromptCapability(myPrompt, myPromptHandler)
))
```

## Creating MCP Server (Routing Function)

```kotlin
// Recommended: use mcp() routing function
val mcpApp = mcp(
    ServerMetaData("MyServer", "1.0.0"),
    NoMcpSecurity,
    tools, resources, prompts,
    path = "/mcp"  // default
)

mcpApp.asServer(Undertow(8080)).start()
```

## Custom Initialize Handler

The MCP handshake uses an `InitializeHandler` that can be customised for protocol version negotiation, capability filtering, or request-level auth:

```kotlin
// Default: SimpleInitializeHandler negotiates protocol version automatically
val protocol = McpProtocol(
    sessions,
    ServerInitializer(SimpleInitializeHandler(metadata)),
    tools = tools
)

// Custom handler — inspect the HTTP request or reject initialization
val customInitializer: InitializeHandler = { req: InitializeRequest ->
    if (req.protocolVersion < minVersion)
        InitializeResponse.Error("Unsupported protocol version")
    else
        InitializeResponse.Ok(
            serverInfo = metadata.entity,
            capabilities = metadata.capabilities,
            protocolVersion = req.protocolVersion
        )
}

val protocol = McpProtocol(
    sessions,
    ServerInitializer(customInitializer),
    tools = tools
)
```

`InitializeRequest` carries `clientInfo`, `capabilities`, `protocolVersion`, and the raw HTTP `connectRequest`.
`InitializeResponse` is a sealed interface with `Ok` and `Error` subtypes.

## Quick Server from Single Capability

```kotlin
// Convert any ServerCapability directly to a server
val tool = Tool("greet", "Say hello") bind { Ok(Text("Hello!")) }

tool.asServer(Undertow(8080))  // convenience extension
```

## Transports (Low-Level)

```kotlin
// HTTP Streaming (SSE-based, recommended)
val mcpHandler = HttpStreamingMcp(tools, resources, prompts)

// HTTP Non-Streaming (request/response)
val mcpHandler = HttpNonStreamingMcp(tools, resources, prompts)

// JSON-RPC over HTTP
val mcpHandler = JsonRpcMcp(tools, resources, prompts)

// WebSocket
val mcpHandler = WebsocketMcp(tools, resources, prompts)

// Standard I/O (for subprocess-based MCP)
StdIoMcpSessions(tools, resources, prompts).start()
```

## Embedding in http4k Server

```kotlin
val app = routes(
    "/mcp" bind mcpHandler
)

app.asServer(Undertow(8080)).start()
```

## Security

```kotlin
val mcpHandler = HttpStreamingMcp(
    tools, resources, prompts,
    security = BearerAuthMcpSecurity { token -> token == "my-secret" }
)

// Available security types:
ApiKeyMcpSecurity { key -> key == "secret" }
BasicAuthMcpSecurity { creds -> creds.user == "admin" }
BearerAuthMcpSecurity { token -> validate(token) }
OAuthMcpSecurity(oauthHandler)
NoMcpSecurity  // default
```

## Observable Capabilities (Change Notifications)

```kotlin
val tools = ServerTools(initialTools)

// Dynamically add/remove tools — clients are notified
tools.add(ToolCapability(newTool, newHandler))
tools.remove(ToolName.of("old-tool"))
```

## Capability Pack (Composing Capabilities)

```kotlin
val capabilities = CapabilityPack(tools, resources, prompts)
val mcpHandler = HttpStreamingMcp(capabilities)
```

## Logging from Server

```kotlin
val logger = Logger(sessions)
logger.log("Server started", LogLevel.info)
```

## OpenTelemetry Tracing

```kotlin
// Add MCP-level tracing (creates SERVER spans per JSON-RPC method)
// openTelemetry is the first parameter, spanModifiers is the second
val tracedMcp = McpFilters.OpenTelemetryTracing(openTelemetry)
    .then(mcpHandler)

// Uses GlobalOpenTelemetry by default
val tracedMcp = McpFilters.OpenTelemetryTracing().then(mcpHandler)
```

Each span is named after the JSON-RPC method (e.g. `tools/call`) and includes attributes:
- `mcp.method.name` — the JSON-RPC method
- `mcp.session.id` — the MCP session ID
- `mcp.protocol.version` — the negotiated protocol version
- `jsonrpc.request.id` — the request ID (when present)

### Span Modifiers

Default span modifiers add method-specific attributes:

- **`CallToolSpanModifiers`** — adds `gen_ai.operation.name=execute_tool`, `gen_ai.tool.name`; sets ERROR status on tool errors
- **`GetPromptSpanModifiers`** — adds `gen_ai.operation.name=get_prompt`, `gen_ai.prompt.name`
- **`ReadResourceSpanModifiers`** — adds `gen_ai.operation.name=read_resource`, `mcp.resource.uri`

Custom modifiers implement `McpOpenTelemetrySpanModifiers` (provides `method`, `request()`, `response()` hooks).

### Transport Span Linking

When nested inside `PolyFilters.OpenTelemetryTracing()`, the MCP span automatically links to the transport span:

```kotlin
val poly = PolyFilters.OpenTelemetryTracing(openTelemetry).then(
    PolyHandler(http = myHttpHandler)
)
```

### Error Handling

- Exceptions: span status set to ERROR, `error.type` set to exception class name
- JSON-RPC errors: span status set to ERROR, `error.type` set to the error code

## Structured Tool Responses

```kotlin
// Return structured JSON alongside text content
val jsonData = McpJson.asJsonObject(mapOf("result" to 42, "message" to "success"))
ToolResponse.Ok(jsonData)  // creates both structured and text representation

// Text-only (traditional)
ToolResponse.Ok("plain text result")
ToolResponse.Ok(Content.Text("hello"), Content.Text("world"))
```

## RenderMcpApp Extra Capabilities

`RenderMcpApp` accepts an `extraCapabilities` list for adding additional tools or resources alongside the rendered app:

```kotlin
RenderMcpApp(
    name = "myApp",
    description = "My MCP app",
    uri = Uri.of("/app"),
    extraCapabilities = listOf(myExtraTool)
) { req -> renderHtml(req) }
```

## Gotchas

- **`McpProtocol` constructor changed**: `ServerMetaData` is no longer the first parameter. Pass `ServerInitializer(SimpleInitializeHandler(metadata))` as the second argument after `sessions`. The old constructor style `McpProtocol(metadata, sessions, ...)` no longer compiles.
- **Use `mcp()` not `mcpHttpStreaming()`**: `mcpHttpStreaming()` is deprecated — use `mcp()` as a direct replacement.
- `HttpStreamingMcp` is recommended for most use cases (supports progress, notifications)
- `StdIoMcpSessions` is for subprocess-based MCP servers (e.g., CLI tools)
- Each transport has matching connection class for lower-level control
- Security is applied per-transport, not per-capability
- `McpFilters.OpenTelemetryTracing()` requires `http4k-ops-opentelemetry` dependency alongside `http4k-ai-mcp-sdk`
