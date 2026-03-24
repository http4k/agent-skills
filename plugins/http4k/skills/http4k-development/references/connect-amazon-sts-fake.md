# http4k-connect-amazon-sts-fake Reference

In-memory fake STS server for testing role assumption.

## Setup

```kotlin
val fakeSts = FakeSTS()
val client = fakeSts.client()
```

## Custom Configuration

```kotlin
val fakeSts = FakeSTS(
    clock = Clock.systemUTC(),
    defaultSessionValidity = Duration.ofHours(1)
)
```

## Test Pattern

```kotlin
val sts = FakeSTS().client()
val result = sts.assumeRole(
    ARN.of("arn:aws:iam::123456789012:role/TestRole"),
    "test-session"
).successValue()

val creds = result.Credentials
// Use creds to create other fake clients with assumed credentials
```

## Chaos Testing

```kotlin
fakeSts.returnStatus(Status.SERVICE_UNAVAILABLE)
fakeSts.behave()
```

## Gotchas

- Extends `ChaoticHttpHandler`
- Returns deterministic fake credentials (not real AWS credentials)
- Expiry set based on `defaultSessionValidity` from the clock
- Does **not** validate role ARNs or permissions
