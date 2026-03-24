# http4k-bridge-spring Reference

Bridge http4k handlers into Spring MVC as a fallback controller.

## Spring Controller

```kotlin
@Controller
class Http4kFallback : SpringToHttp4kFallbackController(myApp)
```

Catches all unmatched routes (`/**`) and delegates to the http4k handler. Spring-mapped routes take priority.

## How It Works

- `@RequestMapping(value = ["**"])` catches all HTTP methods on unmatched paths
- Delegates to the http4k handler via Jakarta servlet adapter
- Automatic `@ExceptionHandler` for `LensFailure` → `400 BAD_REQUEST` with `ProblemDetail`

## Gotchas

- **Fallback only**: Spring-defined routes take priority. The http4k handler only receives requests that Spring doesn't match.
- **Jakarta servlet**: Uses `http4k-bridge-jakarta` internally for request/response conversion.
- **LensFailure handling**: Lens validation failures are automatically mapped to `400 BAD_REQUEST` with a `ProblemDetail` response body.
