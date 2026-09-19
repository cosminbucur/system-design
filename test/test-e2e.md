End-to-end tests exercise the full, running application the way an actual user does — clicking, typing, navigating a real browser against a real (or fully-started local) deployment — rather than testing any single layer in isolation. This sits at the top of the test pyramid: fewer of these tests than unit or integration tests, but they're the ones closest to what a real user actually experiences, and the only way to catch bugs that live purely in the frontend/browser integration (a broken button handler, a CSS issue hiding a submit button, a JS error that never reaches a backend log).

## 1. Playwright: Driving a Real Browser

Playwright drives Chromium, Firefox, and WebKit to exercise the application through its actual UI.

```java
// Playwright's Java binding — works the same way as its JS/Python/.NET counterparts
@Test
void shouldPlaceOrderSuccessfully() {
    try (Playwright playwright = Playwright.create()) {
        Browser browser = playwright.chromium().launch();
        Page page = browser.newPage();

        page.navigate("https://staging.example.com/checkout");
        page.fill("#sku-input", "SKU-1");
        page.fill("#quantity-input", "2");
        page.click("#place-order-button");

        assertThat(page.locator(".order-confirmation")).isVisible();
        assertThat(page.locator(".order-confirmation")).containsText("Order placed");

        browser.close();
    }
}
```

## 2. Playwright's Design Choices That Matter in Practice

- **Auto-waiting**: actions like `click()`/`fill()` automatically wait for the element to be visible/stable/actionable before acting — eliminates the flaky, hand-rolled `Thread.sleep()`/explicit-wait patterns that plagued older tools like Selenium.
- **Auto-generated traces/videos on failure**: a failed test can produce a full trace (DOM snapshots, network calls, console logs) for the exact moment of failure — dramatically easier to debug than a screenshot alone, especially for a failure that only reproduces in CI.
- **Cross-browser from one API**: the same test runs against Chromium, Firefox, and WebKit without rewriting it per browser engine.
- **Network interception**: you can stub/mock specific backend calls (`page.route(...)`) to test frontend behavior in isolation from backend state, or let real requests through when you want a true end-to-end check.

```java
// Network interception: force a specific backend response to test a frontend edge case
page.route("**/api/inventory/**", route ->
    route.fulfill(new Route.FulfillOptions()
        .setStatus(503)
        .setBody("{\"error\":\"inventory service unavailable\"}")));

page.navigate("https://staging.example.com/checkout");
assertThat(page.locator(".inventory-error-banner")).isVisible();
```

## 3. Where E2E Tests Fit in the Suite

Playwright tests are slow (seconds per test, browser startup overhead) and more brittle to UI changes than a unit test, so keep them to genuine user-journey coverage — login, checkout, critical navigation flows — rather than exhaustively covering every UI state. That exhaustive coverage belongs at the unit/component level (a frontend framework's own component testing tools), not in a full browser E2E suite.

Run E2E tests against a real, deployed environment (or a fully-started local stack) rather than trying to mock the backend entirely — the whole point of this layer is validating the real integration end to end, not re-testing what a backend test slice (`@WebMvcTest`) already covers.

## 4. Best Practices

| Practice | Recommendation |
| --- | --- |
| Reserve E2E browser tests for critical user journeys | They're slow and UI-change-brittle — exhaustive UI state coverage belongs at the unit/component level, not full browser E2E. |
| Rely on auto-waiting instead of manual sleeps | Playwright's built-in waiting eliminates the flaky `Thread.sleep()` patterns that plagued older tools. |
| Capture traces/videos on failure | A full trace of DOM, network, and console state at the moment of failure is far easier to debug than a bare screenshot. |
| Run against a real deployed environment or full local stack | The point of E2E is validating the real integration — mocking the backend away defeats the purpose. |
| Use network interception sparingly, for specific edge cases | Useful for testing an error state that's hard to trigger for real, but overusing it turns an E2E test into a mocked test with extra steps. |
| Run the same suite across browser engines when it matters | Cross-browser coverage from one API is cheap with Playwright — use it for user journeys where browser differences are a real risk. |
