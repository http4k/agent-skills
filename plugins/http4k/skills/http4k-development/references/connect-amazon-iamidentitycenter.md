---
license: Apache-2.0
module: http4k-connect-amazon-iamidentitycenter
---

# http4k-connect-amazon-iamidentitycenter Reference

IAM Identity Center client — two clients in one module: SSO for credential federation and OIDC for device authorization.

## SSO Client

```kotlin
val sso = SSO.Http(
    region = Region.of("us-east-1"),
    http = JavaHttpClient()   // optional — no credentials needed (uses token)
)
```

### Get Federated Credentials

```kotlin
val credentials = sso.getFederatedCredentials(
    accessToken = AccessToken.of("access-token-from-device-auth"),
    accountId = AccountId.of("123456789012"),
    roleName = RoleName.of("DeveloperAccess")
).successValue().RoleCredentials
```

## OIDC Client

```kotlin
val oidc = OIDC.Http(
    region = Region.of("us-east-1"),
    http = JavaHttpClient()   // optional
)
```

### Device Authorization Flow

```kotlin
// 1. Register the client application
val registration = oidc.registerClient(
    clientName = ClientName.of("my-cli-tool"),
    clientType = ClientType.PUBLIC
).successValue()

// 2. Start device authorization
val deviceAuth = oidc.startDeviceAuthorization(
    clientId = registration.ClientId,
    clientSecret = registration.ClientSecret,
    startUrl = StartUrl.of("https://my-org.awsapps.com/start")
).successValue()

println("Open: ${deviceAuth.VerificationUriComplete}")

// 3. Poll for token (after user authorizes)
val token = oidc.createToken(
    clientId = registration.ClientId,
    clientSecret = registration.ClientSecret,
    deviceCode = deviceAuth.DeviceCode,
    grantType = GrantType.DEVICE_CODE
).successValue().AccessToken
```

## Gotchas

- SSO and OIDC are two separate clients in the same module
- SSO client does NOT use AWS SigV4 — it uses bearer token auth
- OIDC device flow: poll `createToken` until user approves (handle `AuthorizationPendingException`)
- Access tokens expire — re-run device flow to get new token
- SSO credentials have short TTL — cache and refresh before expiry
