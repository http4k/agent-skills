---
license: Apache-2.0
module: http4k-security-oauth
---

# http4k-security-oauth Reference

Full OAuth 2.0 implementation — both client (consumer) and server (authorization server). Supports authorization code flow, PKCE, OpenID Connect, CSRF protection, refresh tokens, and client credentials.

## OAuth Client (Consumer)

### Setup

```kotlin
val oauthProvider = OAuthProvider(
    providerConfig = OAuthProviderConfig(
        authBase = Uri.of("https://auth.example.com"),
        authPath = "/authorize",
        tokenPath = "/oauth/token",
        credentials = Credentials("client-id", "client-secret")
    ),
    client = httpClient,
    callbackUri = Uri.of("https://myapp.com/callback"),
    scopes = listOf("openid", "email", "profile"),
    oAuthPersistence = InsecureCookieBasedOAuthPersistence("oauth")
)

val app = routes(
    oauthProvider.callbackEndpoint,                           // handles redirect from provider
    "/protected" bind GET to oauthProvider.authFilter.then {  // redirects to provider if no token
        Response(OK).body("authenticated content")
    }
)
```

### Preconfigured Providers

```kotlin
OAuthProvider.google(client, credentials, callbackUri, persistence, listOf("openid", "email"))
OAuthProvider.gitHub(client, credentials, callbackUri, persistence, listOf("user:email"))
OAuthProvider.auth0(auth0Uri, client, credentials, callbackUri, persistence)
OAuthProvider.facebook(client, credentials, callbackUri, persistence, listOf("email"))
OAuthProvider.discord(client, credentials, callbackUri, persistence)
```

### With PKCE

```kotlin
val oauthProvider = OAuthProvider(
    providerConfig = config,
    client = httpClient,
    callbackUri = Uri.of("https://myapp.com/callback"),
    scopes = listOf("openid"),
    oAuthPersistence = persistence,
    pkceGenerator = PkceChallengeAndVerifier.SECURE_PKCE
)
```

### OAuthPersistence

Stores CSRF tokens, access tokens, and PKCE state between redirect and callback:

```kotlin
// Dev/testing only — stores in cookies
InsecureCookieBasedOAuthPersistence("cookiePrefix", cookieValidity = Duration.ofDays(1))

// Production — implement the interface with secure storage
interface OAuthPersistence {
    fun assignCsrf(redirect: Response, csrf: CrossSiteRequestForgeryToken): Response
    fun retrieveCsrf(request: Request): CrossSiteRequestForgeryToken?
    fun assignToken(request: Request, redirect: Response, accessToken: AccessToken, idToken: IdToken?): Response
    fun retrieveToken(request: Request): AccessToken?
    fun assignOriginalUri(redirect: Response, originalUri: Uri): Response
    fun retrieveOriginalUri(request: Request): Uri?
    fun assignNonce(redirect: Response, nonce: Nonce): Response
    fun retrieveNonce(request: Request): Nonce?
    fun assignPkce(redirect: Response, pkce: PkceChallengeAndVerifier): Response
    fun retrievePkce(request: Request): PkceChallengeAndVerifier?
}
```

## OAuth Server (Authorization Server)

### Setup

```kotlin
val server = OAuthServer(
    tokenPath = "/oauth2/token",
    authRequestTracking = InsecureCookieBasedAuthRequestTracking(),
    authoriseRequestValidator = myClientValidator,
    accessTokenRequestAuthentication = ClientSecretAccessTokenFetcherAuthenticator(config),
    authorizationCodes = InMemoryAuthorizationCodes(clock),
    accessTokens = myAccessTokenStore,
    clock = Clock.systemUTC(),
    json = Jackson,
    idTokens = IdTokens.Unsupported,       // enable for OpenID Connect
    refreshTokens = RefreshTokens.Unsupported  // enable for refresh token support
)

val app = routes(
    server.tokenRoute,                                           // POST /oauth2/token
    "/login" bind GET to server.authenticationStart.then {       // shows login page
        Response(OK).body("Please log in")
    },
    "/login" bind POST to server.authenticationComplete          // completes auth, issues code
)
```

### AuthorizationCodes

```kotlin
interface AuthorizationCodes {
    fun create(request: Request, authRequest: AuthRequest, response: Response):
        Result<AuthorizationCode, UserRejectedRequest>
    fun detailsFor(code: AuthorizationCode): AuthorizationCodeDetails
}

// In-memory implementation for testing
class InMemoryAuthorizationCodes(clock: Clock) : AuthorizationCodes
```

### AccessTokens

```kotlin
interface AccessTokens {
    fun create(clientId: ClientId, tokenRequest: AuthorizationCodeAccessTokenRequest):
        Result<AccessToken, AccessTokenError>
    fun create(clientId: ClientId, tokenRequest: TokenRequest):
        Result<AccessToken, AccessTokenError>
}
```

## Client Credentials Flow

```kotlin
val tokenRequest = ClientFilters.OAuthClientCredentials(
    clientCredentials = Credentials("client-id", "client-secret"),
    scopes = listOf("service:read")
).then(httpClient)

val tokenResponse = tokenRequest(Request(POST, "https://auth.example.com/token"))
```

## Refresh Token Flow

```kotlin
val refreshRequest = ClientFilters.OAuthRefreshToken(
    clientCredentials = Credentials("client-id", "client-secret"),
    token = RefreshToken("refresh-token-value"),
    scopes = listOf("openid")
).then(httpClient)

val tokenResponse = refreshRequest(Request(POST, "https://auth.example.com/token"))
```

## Response Types

```kotlin
ResponseType.Code          // Authorization code (default, most secure)
ResponseType.Token         // Implicit flow (deprecated)
ResponseType.CodeIdToken   // Code + OpenID ID token
ResponseType.CodeToken     // Code + access token
```

## CSRF Protection

Built into the OAuth flow automatically:

```kotlin
// CSRF token is generated during redirect, stored via OAuthPersistence,
// carried in the OAuth state parameter, and validated on callback.
val provider = OAuthProvider(
    // ...
    generateCrsf = CrossSiteRequestForgeryToken.SECURE_CSRF  // default
)
```

## Complete End-to-End Example

```kotlin
// Authorization server
val authServer = OAuthServer(
    tokenPath = "/oauth2/token",
    authRequestTracking = InsecureCookieBasedAuthRequestTracking(),
    authoriseRequestValidator = myValidator,
    accessTokenRequestAuthentication = myAuthenticator,
    authorizationCodes = InMemoryAuthorizationCodes(clock),
    accessTokens = myTokenStore,
    clock = clock
)

val authApp = routes(
    authServer.tokenRoute,
    "/login" bind GET to authServer.authenticationStart.then { Response(OK).body("Login form") },
    "/login" bind POST to authServer.authenticationComplete
)

// Client application
val oauthProvider = OAuthProvider(
    OAuthProviderConfig(Uri.of("http://auth-server"), "/login", "/oauth2/token", credentials),
    authApp,
    Uri.of("/callback"),
    listOf("read", "write"),
    InsecureCookieBasedOAuthPersistence("oauth")
)

val clientApp = routes(
    oauthProvider.callbackEndpoint,
    "/protected" bind GET to oauthProvider.authFilter.then { Response(OK).body("secret") }
)
```

## Gotchas

- **InsecureCookieBasedOAuthPersistence is for dev only**: It stores tokens in plain cookies. Implement `OAuthPersistence` with secure storage for production.
- **CSRF is automatic**: The OAuth state parameter carries the CSRF token. Don't disable it.
- **authFilter redirects on missing token**: `oauthProvider.authFilter` returns a `307 TEMPORARY_REDIRECT` to the provider when no token is found in the request.
- **callbackEndpoint must be routed**: The callback URI must be bound in your routes — use `oauthProvider.callbackEndpoint` which binds to the callback path.
- **Token exchange is server-side**: The authorization code is exchanged for an access token on the server side (not in the browser), keeping the client secret safe.
- **PKCE is optional**: Pass `pkceGenerator = PkceChallengeAndVerifier.SECURE_PKCE` for public clients that can't keep a client secret.
