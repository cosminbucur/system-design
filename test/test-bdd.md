Behavior-Driven Development (BDD) writes tests as executable specifications of behavior, expressed in plain language a non-technical stakeholder can read and agree with, rather than starting from test code. The distinguishing idea isn't a specific tool — it's that the scenario itself, written in a structured natural-language format, is both the specification and the automated test, so there's no separate "requirements doc" that can silently drift out of sync with what the code actually does.

## 1. Gherkin: Given/When/Then

BDD scenarios are typically written in Gherkin, a structured plain-language syntax built around three clauses: the starting state (`Given`), the action taken (`When`), and the expected outcome (`Then`).

```gherkin
Feature: Order placement

  Scenario: Placing an order with sufficient stock succeeds
    Given a customer with an active account
    And the SKU "SKU-1" has 5 units in stock
    When the customer places an order for 2 units of "SKU-1"
    Then the order should be confirmed
    And the stock for "SKU-1" should be reduced to 3 units

  Scenario: Placing an order with insufficient stock fails
    Given a customer with an active account
    And the SKU "SKU-1" has 0 units in stock
    When the customer places an order for 2 units of "SKU-1"
    Then the order should be rejected with reason "OUT_OF_STOCK"
```

This file is readable by a product owner, a QA engineer, and a developer without any of them needing to read Java — and it's simultaneously the literal input the test runner executes.

## 2. Cucumber: Wiring Gherkin to Java

Cucumber is the standard JVM tool that parses `.feature` files and maps each Gherkin line to a Java method (a "step definition") that actually exercises the system.

```java
public class OrderStepDefinitions {

    private Customer customer;
    private OrderResult result;

    @Given("a customer with an active account")
    public void aCustomerWithAnActiveAccount() {
        customer = testFixtures.createActiveCustomer();
    }

    @Given("the SKU {string} has {int} units in stock")
    public void theSkuHasUnitsInStock(String sku, int quantity) {
        inventoryService.setStock(sku, quantity);
    }

    @When("the customer places an order for {int} units of {string}")
    public void theCustomerPlacesAnOrder(int quantity, String sku) {
        result = orderService.placeOrder(customer, sku, quantity);
    }

    @Then("the order should be confirmed")
    public void theOrderShouldBeConfirmed() {
        assertThat(result.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }

    @Then("the stock for {string} should be reduced to {int} units")
    public void theStockShouldBeReducedTo(String sku, int expectedStock) {
        assertThat(inventoryService.getStock(sku)).isEqualTo(expectedStock);
    }
}
```

`{string}` and `{int}` are Cucumber expression placeholders — the same step definition method is reused across scenarios that pass different concrete values, the same parameterization idea as a JUnit `@ParameterizedTest`.

## 3. Where BDD Tests Fit in the Test Pyramid

BDD isn't a separate layer of the pyramid so much as a different *authoring style* that can sit at different layers — the same `Given/When/Then` structure can drive a fast in-process test calling services directly (closer to an integration test) or drive a full browser session via Playwright underneath the step definitions (closer to an E2E test). What determines its cost and speed is what the step definitions actually do, not the fact that it's written in Gherkin.

| Style | What step definitions do | Speed |
| --- | --- | --- |
| BDD over service calls | Call application services/repositories directly, no HTTP or browser involved | Fast — comparable to an integration test |
| BDD over the API | Step definitions make real HTTP calls to a running instance | Moderate |
| BDD over the browser | Step definitions drive Playwright/Selenium under the hood | Slow — comparable to a full E2E test |

## 4. BDD's Real Value: Shared Language, Not Just Automation

The main benefit isn't "yet another way to write tests" — it's that scenarios become a shared artifact between product, QA, and engineering, written in the same ubiquitous language the business already uses (the same principle behind naming things well in clean code, and behind DDD's ubiquitous language). A scenario like "placing an order with insufficient stock fails" is something a product owner can review and confirm matches their intent *before* it's ever automated, catching a misunderstanding of requirements far earlier and far more cheaply than discovering it after the feature is built.

## 5. Common Pitfalls

- **Turning Gherkin into a scripting language**: steps like `Given I click the button with id "submit"` are just UI automation wearing a Gherkin costume — they've lost the plain-language, business-readable quality that's the entire point of BDD. Keep steps expressed in terms of business behavior, not UI mechanics.
- **One step definition per scenario, duplicated**: step definitions should be small, reusable, and composable across many scenarios — a step definition written to only ever be used once defeats the reuse that makes Gherkin scenarios fast to write.
- **Letting feature files become the only source of test coverage**: BDD scenarios are naturally sparse (they read best when there aren't too many) — don't let "we have BDD scenarios" substitute for the detailed edge-case coverage that belongs in ordinary unit tests.

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Write scenarios in business language, not UI mechanics | A step like "clicks the button with id X" has lost the whole point of a plain-language, business-readable scenario. |
| Involve non-engineers in writing/reviewing scenarios | The shared-language benefit only materializes if product/QA actually read and shape the scenarios, not just engineers. |
| Keep step definitions small and reusable | Steps duplicated per scenario lose the composability that makes Gherkin fast to extend with new scenarios. |
| Choose the right layer for step definitions | Service-level steps for fast feedback; browser-driven steps only for the specific user journeys that genuinely need full E2E coverage. |
| Don't replace unit test coverage with BDD scenarios | BDD scenarios are meant to be few and readable — detailed edge-case coverage still belongs in unit tests. |
| Keep `.feature` files under version control alongside the code | They're both specification and test — they should evolve, review, and merge exactly like the code they describe. |
