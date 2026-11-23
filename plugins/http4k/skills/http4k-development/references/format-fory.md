---
license: Apache-2.0
module: http4k-format-fory
---

# http4k-format-fory Reference

Apache Fory binary serialization. Unlike the JSON formats this produces bytes, not text, and
the default content type is `application/octet-stream`.

## Construction

`Fory { ... }` is a factory function (not a singleton object) that returns a `ConfigurableFory`.
**Every type you marshal must be registered** with `register<T>()`; unregistered types are
refused, so untrusted payloads cannot name arbitrary classes to instantiate.

```kotlin
val fory = Fory {
    register<MyObject>()
    register<MyEnum>()
    value(MyValue)
    text(BiDiMapping({ MyType.parse(it) }, MyType::toString))
}
```

## Direct Use

```kotlin
val bytes: ByteArray = fory.asBytes(myObject)
val back = fory.asA<MyObject>(bytes)
val back = fory.asA(bytes, MyObject::class)
val back = fory.asA<MyObject>(inputStream)
```

## Message Bodies

```kotlin
with(fory) {
    // write and set the content type in one call
    val request = Request(GET, "/").binary(myObject)

    // read back
    val myObject: MyObject = request.binary()
}
```

## Lenses

```kotlin
with(fory) {
    val bodyLens = Body.auto<MyObject>().toLens()

    val response = Response(OK).with(bodyLens of myObject)
    val myObject = bodyLens(request)

    // websockets
    val wsLens = WsMessage.auto<MyObject>().toLens()
}
```

## Gotchas

- Class registration is required: every top-level type and every nested/contained user type
  must be registered via `register<T>()`, otherwise marshalling fails. Forgetting a type is
  the most common failure
- Content type is `OCTET_STREAM`, not JSON — override with the `contentType` argument to
  `auto`/`autoBody` when a more specific binary type is wanted
- Cross-language mode (`withXlang`) is off
- `Fory` is backed by a `ThreadSafeFory` and is safe to use concurrently from many threads
- Value types need registering via `value(MyValue)` inside `Fory { }` before they roundtrip
  inside maps and other containers — the same registration story as the JSON formats
- Common JDK and http4k types (`Instant`, `UUID`, `Uri`, `Status`, `ZoneId`, `Locale`, ...)
  roundtrip out of the box via the standard mappings
- Binary payloads are not human-readable — prefer a JSON format for public APIs and debugging
