---
license: Apache-2.0
module: http4k-testing-webdriver-datastar
---

# http4k-testing-webdriver-datastar Reference

Headless datastar v1 testing support — a `DatastarWebDriver` that drives an http4k app as if a browser running datastar were connected. Maintains a signal store, evaluates `data-*` expressions, applies `patch-elements` and `patch-signals` SSE events, and sends non-local signals with every backend action.

## Construction

```kotlin
// Single entry point — wraps Http4kWebDriver with DatastarBehaviour
val driver = DatastarWebDriver(myApp)

// With custom clock (for deterministic testing)
val driver = DatastarWebDriver(myApp, clock = fixedClock)
```

## Navigation and Element Interaction

Works exactly like `Http4kWebDriver` via the Selenium `WebDriver` interface:

```kotlin
driver.get("http://localhost/")
driver.findElement(By.id("btn")).click()
driver.findElement(By.id("input")).sendKeys("hello")
driver.findElement(By.tagName("form")).submit()
driver.navigate().back()
driver.navigate().forward()
```

After any interaction that triggers a backend SSE response, element patches are applied to the live document before control returns — reads on elements reflect the post-patch state.

## Signal Store

datastar signals defined in `data-signals` are parsed into the store on page load. Elements with `data-bind` sync their value into the store on input. Non-local signals (no leading `_`) are sent with every backend action.

```html
<!-- Initialise signals from attribute -->
<body data-signals="{count: 1, user: {name: 'bob'}}">

<!-- Single signal via kebab-case attribute (camelCase in store) -->
<div data-signals-my-count="41">

<!-- Two-way binding to input value -->
<input data-bind="name" value="alice">
```

```kotlin
driver.get("http://localhost/")
driver.findElement(By.id("btn")).click()
// signals are serialised and sent with the request automatically
```

## Backend Actions

`@get`, `@post`, `@put`, `@patch`, `@delete` expressions in `data-on-*` send signals to the backend:

- **GET** — signals serialised as `?datastar=<json>` query param, with `datastar-request: true` header
- **non-GET** — signals sent as a JSON body (`Content-Type: application/json`)

Local signals (names starting with `_`) are excluded from transport.

## SSE Response Events

The driver processes `text/event-stream` responses inline:

- **`datastar-patch-elements`** — merges the returned HTML fragment into the live document via morphing
- **`datastar-patch-signals`** — deep-merges signal values into the store

```kotlin
// Server handler returning SSE events
fun handle(req: Request): Response {
    val events = sequenceOf(
        DatastarEvent.PatchSignals(Signal("count") to "42"),
        DatastarEvent.PatchElements(Selector("#counter"), "<span id='counter'>42</span>")
    )
    return Response(OK).datastarEvents(events)
}
```

## Visibility (data-show)

`data-show="<expr>"` sets `style="display:none"` when the expression is falsy. `element.isDisplayed` reflects this:

```kotlin
// <div id='panel' data-show="$open">
assertThat(driver.findElement(By.id("panel")).isDisplayed, equalTo(false))
```

## Reactivity (data-text, data-class, computed signals)

`data-text="<expr>"` updates the element's text content reactively on every render cycle. Expressions are evaluated against the current signal store:

```kotlin
// <span data-text="'count is ' + $count">
driver.findElement(By.id("inc")).click()
assertThat(driver.findElement(By.id("out")).text, equalTo("count is 2"))
```

## Gotchas

- **In-process only**: HTTP requests go directly to the handler — no network, no real browser.
- **SSE consumed synchronously**: SSE response streams are consumed and applied before the action returns — test assertions can follow directly.
- **Local signals excluded from transport**: Signal names starting with `_` are never sent to the backend. Use them for UI-only state.
- **`pageSource` reflects live DOM**: `driver.pageSource` returns the post-patch live document, not the original response body.
- **Morphing is merge-based**: `patch-elements` uses a morphing strategy — elements are updated in place rather than replaced, preserving focused inputs.
- **No real JS engine**: Expressions in `data-*` attributes are evaluated by a lightweight built-in evaluator, not V8. Complex JS (closures, async, DOM APIs) is not supported.
