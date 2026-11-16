---
module: http4k-connect-openfga-fake
license: Apache-2.0
---

# http4k-connect-openfga-fake Reference

In-memory fake of the OpenFGA API for testing code that uses `http4k-connect-openfga`. It evaluates authorization models (direct, computed, tuple-to-userset, union, intersection, exclusion, wildcards, usersets) against stored tuples, so checks, `ListObjects` and `ListUsers` give real answers.

## Basic Usage

```kotlin
val fake = FakeOpenFga()
val openFga = fake.client()   // in-process, no network port

val store = openFga.createStore(StoreName.of("orion")).successValue().id
openFga.writeAuthorizationModel(store, listOf(TypeDefinition.of(user), TypeDefinition.of(doc, viewer to direct(user)))).successValue()
openFga.write(store, writes = listOf(TupleKey(anne, viewer, budget))).successValue()

openFga.check(store, TupleKey(anne, viewer, budget)).successValue().allowed   // true
```

## Constructor

```kotlin
FakeOpenFga(
    clock: Clock = Clock.systemUTC(),
    stores: MutableMap<StoreId, FgaStore> = ConcurrentHashMap()  // pre-seedable / inspectable
)
```

Extends `ChaoticHttpHandler` for failure injection. Run standalone with `FakeOpenFga().start()` or in tests via `WithRunningFake(::FakeOpenFga)`.

## Behaviour to Know

- Stores must have an authorization model written before `Write`/`Check` (otherwise `400`); unknown stores return `404`.
- Contextual tuples apply to the single request (or batch item) only.
- Queries can be pinned to an earlier model via `authorizationModel`.

## Contract Testing

Share `OpenFgaContract` (from the client module's test fixtures) between the fake and a real OpenFGA server:

```kotlin
class FakeOpenFgaTest : OpenFgaContract { override val http = FakeOpenFga() }
```
