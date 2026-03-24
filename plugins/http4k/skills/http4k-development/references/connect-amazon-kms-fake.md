---
license: Apache-2.0
module: http4k-connect-amazon-kms-fake
---

# http4k-connect-amazon-kms-fake Reference

In-memory fake KMS server with real cryptographic operations via BouncyCastle.

## Setup

```kotlin
val fakeKms = FakeKMS()
val client = fakeKms.client()
```

## Custom Key Storage

```kotlin
val fakeKms = FakeKMS(
    keys = Storage.InMemory<StoredCMK>()
)
```

## Test Pattern

```kotlin
val kms = FakeKMS().client()

// Create a key
val keyId = kms.createKey().successValue().KeyMetadata.KeyId

// Encrypt and decrypt
val ciphertext = kms.encrypt(keyId, plaintext).successValue().CiphertextBlob
val decrypted = kms.decrypt(ciphertext, keyId).successValue().Plaintext
assertThat(decrypted, equalTo(plaintext))
```

## Chaos Testing

```kotlin
fakeKms.returnStatus(Status.SERVICE_UNAVAILABLE)
fakeKms.behave()
```

## Gotchas

- **Uses real cryptography** (BouncyCastle) — key operations behave like real KMS
- Requires `org.bouncycastle:bcpkix-jdk18on` on the classpath
- Extends `ChaoticHttpHandler`
- Keys are stored in-memory and lost when the fake is discarded
- Signing/verification uses real RSA/EC algorithms
