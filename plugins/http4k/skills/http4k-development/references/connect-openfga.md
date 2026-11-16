---
module: http4k-connect-openfga
license: Apache-2.0
---

# http4k-connect-openfga Reference

Client for the OpenFGA fine-grained authorization API (stores, authorization models, relationship tuples, checks).

## Construction

```kotlin
val openFga = OpenFga.Http(
    baseUri = Uri.of("http://localhost:8080"),
    http = JavaHttpClient()   // optional — defaults to JavaHttpClient
)
```

Actions are invoked via `openFga(Action(...))` or the generated extension functions (`openFga.check(...)`). All return `Result<R, RemoteFailure>`.

## Typical Flow

A store needs an authorization model before it can be written to or checked.

```kotlin
val user = TypeName.of("user")
val doc = TypeName.of("document")
val viewer = Relation.of("viewer")
val editor = Relation.of("editor")

val store = openFga.createStore(StoreName.of("orion")).successValue().id

openFga.writeAuthorizationModel(
    store,
    listOf(
        TypeDefinition.of(user),
        TypeDefinition.of(doc, editor to direct(user), viewer to computed(editor))
    )
).successValue()

val tuple = TupleKey(User.of("user:anne"), viewer, FgaObject.of("document:budget"))
openFga.write(store, writes = listOf(tuple)).successValue()

val allowed: Boolean = openFga.check(store, tuple).successValue().allowed
```

Relation definition builders (`RelationDefinition.Companion`): `direct`, `computed`, `from`, `union`, `intersection`, `butNot`.

## Actions

- Stores: `CreateStore`, `GetStore`, `ListStores`, `DeleteStore`
- Models: `WriteAuthorizationModel`, `ReadAuthorizationModel`, `ReadAuthorizationModels`
- Tuples: `Write` (`writes`, `deletes`, `onDuplicate`/`onMissing: OnConflict`), `Read` (with `TupleFilter`), `ReadChanges`
- Queries: `Check`, `BatchCheck` (list of `BatchCheckItem(tuple, CorrelationId)`; results keyed by correlation id), `ListObjects`, `StreamedListObjects`, `ListUsers` (with `UserTypeFilter`)
- Assertions: `WriteAssertions`, `ReadAssertions`

Query actions accept optional `contextualTuples` (apply to that request only), `authorizationModel` (pin to an earlier model id) and `consistency` (`ConsistencyPreference`).

## Value Types

`User`, `Relation`, `FgaObject`, `TypeName`, `StoreId`, `StoreName`, `AuthorizationModelId`, `CorrelationId` etc. are non-blank string value types — construct with `.of()`. Users and objects use OpenFGA's `type:id` format (`user:anne`); wildcards are `user:*`; usersets are `group:eng#member`.

## Gotchas

- **Model before tuples**: `Write` and `Check` on a store with no authorization model fail with `400 BAD_REQUEST`.
- **Operations on missing stores** fail with `404 NOT_FOUND`.
- **`TupleKey.obj`** serialises as JSON `object`.
- **`Write` omits empty lists**: pass only `writes` or `deletes` as needed.

## Testing

Use `http4k-connect-openfga-fake` (`FakeOpenFga().client()`).
