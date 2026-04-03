---
module: http4k-tools-verify
license: http4k Commercial
---

# http4k-tools-verify Reference

A Gradle plugin that verifies cosign signatures on http4k artifacts downloaded from the http4k Enterprise Repository. Adds a `verifyHttp4kDependencies` task that runs automatically before compilation, ensuring supply chain integrity.

## Setup

Apply the plugin in your `build.gradle.kts`:

```kotlin
plugins {
    id("org.http4k.verify")
}
```

No further configuration is required. The plugin downloads the http4k public key from `https://http4k.org/cosign.pub` automatically.

## Configuration

```kotlin
http4kVerify {
    failOnError.set(true)                    // default: true — throw on verification failure
    publicKey.set(file("cosign.pub"))        // optional: use a local key file instead
}
```

- **`failOnError`** — when `true`, a failed signature check throws `GradleException` and stops the build
- **`publicKey`** — local PEM file path; if omitted, the public key is fetched from `https://http4k.org/cosign.pub`

## What It Verifies

For each `org.http4k` artifact on the `runtimeClasspath`, the plugin checks up to four sigstore bundles:

| Check | Classifier |
|-------|-----------|
| JAR signature | `jar-sigstore` |
| SBOM (CycloneDX) | `cyclonedx-sigstore` |
| SLSA provenance | `provenance-sigstore` |
| License report | `license-report-sigstore` |

Bundles are only available from the http4k Enterprise Repository (`maven.http4k.org`). If no bundles are found, the plugin logs a notice and skips verification without failing.

## Caching

Verified artifacts are recorded in `~/.gradle/caches/http4k-verify/verified.txt` keyed by artifact SHA-256. Already-verified artifacts are skipped on subsequent builds.

## Task

```
./gradlew verifyHttp4kDependencies
```

The task runs in the `verification` group and is wired as a dependency of `compileKotlin` / `compileJava`.

## Output Example

```
Verifying 3 http4k module(s)...
  org.http4k:http4k-core:6.40.0.0           jar ✓   sbom ✓   provenance ✓   license ✓
  org.http4k:http4k-server-jetty:6.40.0.0   jar ✓   sbom ✓   provenance -   license -
Verified: 3 modules, 6 signatures
```

A `-` means the artifact type was not available (not published with that classifier).

## Gotchas

- **Enterprise Repository required**: Sigstore bundles are only published to `maven.http4k.org`. Artifacts from Maven Central have no bundles — the plugin skips them silently.
- **`failOnError = false` for graceful degradation**: If you want warnings without breaking the build when bundles are unavailable, set `failOnError.set(false)`.
- **Public key is fetched on first run**: If the build environment has no internet access, supply a local `publicKey` file.
- **EC key format**: The PEM public key must be an EC key (`-----BEGIN PUBLIC KEY-----`). Verification uses `SHA256withECDSA`.
