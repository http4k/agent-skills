---
license: http4k Commercial
module: http4k-ai-mcp-client
---

# http4k-ai-mcp-client Reference

MCP client — connect to MCP servers and call tools, resources, and prompts.

## Creating a Client

```kotlin
// HTTP Streaming (SSE-based, recommended)
val client = HttpStreamingMcpClient(
    baseUri = Uri.of("http://localhost:8080/mcp"),
    name = McpEntity.of("my-client"),       // optional, defaults to "http4k-mcp-client"
    version = Version.of("1.0.0"),          // optional, defaults to "0.0.0"
    http = JavaHttpClient()                 // optional
)

// Minimal — URI is the only required parameter
val client = HttpStreamingMcpClient(Uri.of("http://localhost:8080/mcp"))

// HTTP Non-Streaming
val client = HttpNonStreamingMcpClient(Uri.of("http://localhost:8080/mcp"))

// WebSocket
val client = WebsocketMcpClient(Request(GET, "ws://localhost:8080/mcp"))

// SSE
val client = SseMcpClient(Request(GET, "http://localhost:8080/mcp"))
```

## Connecting

```kotlin
client.use { mcp ->
    val caps = mcp.start().getOrThrow()
    println("Connected. Tools: ${caps.tools != null}")

    // ... use the client
}
// AutoCloseable — client.stop() called on exit
```

## Tools

```kotlin
val tools = mcp.tools()

// List available tools
val toolList: List<McpTool> = tools.list().getOrThrow()

// Call a tool
val result: ToolResponse = tools.call(
    ToolName.of("calculator"),
    ToolRequest(mapOf("operation" to "add", "a" to 1, "b" to 2))
).getOrThrow()

// React to tool list changes
tools.onChange { println("Tool list updated") }
```

## Resources

```kotlin
val resources = mcp.resources()

val list: List<McpResource> = resources.list().getOrThrow()
val templates: List<McpResource> = resources.listTemplates().getOrThrow()

val content: ResourceResponse = resources.read(
    ResourceRequest(Uri.of("file:///data/config.json"))
).getOrThrow()

// ResourceResponse is a sealed interface — pattern-match on the result
when (content) {
    is ResourceResponse.Ok -> content.list.forEach { println(it) }
    is ResourceResponse.Error -> println("Error: ${content.message}")
}

resources.subscribe(Uri.of("file:///data/config.json")) {
    println("Resource updated")
}
```

## Prompts

```kotlin
val prompts = mcp.prompts()

val list: List<McpPrompt> = prompts.list().getOrThrow()

val result: PromptResponse = prompts.get(
    PromptName.of("summarize"),
    PromptRequest(mapOf("text" to "Hello world", "style" to "brief"))
).getOrThrow()

// PromptResponse is a sealed interface
when (result) {
    is PromptResponse.Ok -> result.messages.forEach { println(it) }
    is PromptResponse.Error -> println("Error: ${result.message}")
}
```

## Sampling (LLM from server → client)

```kotlin
val sampling = mcp.sampling()
sampling.onRequest { req ->
    // Server is requesting LLM generation from the client
    val response = myChat(ChatRequest(req.messages, ModelParams(...)))
    sequence { yield(SamplingResponse(...)) }
}
```

## LLM Tool Integration

```kotlin
// Convert MCP tools to LLM tools for use with Chat
val llmTools: List<LLMTool> = McpLLMTools.fromClient(mcp).getOrThrow()

val response = chat(ChatRequest(
    messages,
    ModelParams(modelName, tools = llmTools)
))
```

## Testing

```kotlin
val testClient = TestMcpClient()
// Configure with fake tools, resources, prompts for unit testing
```

## Gotchas

- **Constructor parameter order**: `baseUri`/`sseRequest`/`wsRequest` is the first parameter. Name and version are optional with defaults.
- Always use `client.use { }` or call `client.stop()` to release connections
- `start()` must be called before any operations
- Override timeouts per-call: `tools.list(overrideDefaultTimeout = 30.seconds)`
- `onChange` callbacks fire on the transport thread — don't block
- **`ResourceResponse`, `PromptResponse`, `CompletionResponse` are sealed interfaces**: Pattern-match on `Ok` and `Error` subtypes. Domain errors from the server are reconstructed as `*.Error` (not `McpError.Protocol`) when the server returns error code `-32050`.
