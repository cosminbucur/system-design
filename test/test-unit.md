Unit tests verify one class in isolation, with its collaborators replaced by fakes — the fastest, cheapest, and most numerous layer of the test pyramid. In the Java ecosystem this centers on JUnit 5 (the test runner/framework), Mockito (mocking collaborators), and AssertJ (fluent assertions). The goal isn't 100% coverage — it's confidence that each unit behaves correctly, at a cost cheap enough to run constantly.

## 1. JUnit 5 Basics

```java
class AccountServiceTest {

    private AccountService accountService;

    @BeforeEach
    void setUp() {
        accountService = new AccountService(); // fresh instance per test — avoid shared mutable state
    }

    @Test
    void shouldThrowWhenWithdrawingMoreThanBalance() {
        Account account = new Account("Cosmin", new BigDecimal("100.00"));

        assertThrows(InsufficientFundsException.class,
            () -> accountService.withdraw(account, new BigDecimal("150.00")));
    }

    @ParameterizedTest
    @ValueSource(strings = {"", " ", "\t"})
    void shouldRejectBlankOwnerName(String blankName) {
        assertThrows(IllegalArgumentException.class, () -> new Account(blankName, BigDecimal.ZERO));
    }

    @Nested
    class WhenAccountIsFrozen {
        @Test
        void shouldRejectAnyWithdrawal() { /* ... */ }
    }
}
```

Naming convention: `shouldXWhenY()` or `methodName_condition_expectedResult()` — pick one and be consistent; the test name should describe the behavior, not implementation.

Key annotations: `@Test`, `@BeforeEach`/`@AfterEach` (per-test setup/teardown), `@BeforeAll`/`@AfterAll` (once per class, must be `static`), `@ParameterizedTest` + `@ValueSource`/`@CsvSource`/`@MethodSource` (run the same test with multiple inputs), `@Nested` (group related tests, share context via outer class).

## 2. Assertions

```java
assertEquals(expected, actual);
assertTrue(condition);
assertNotNull(value);

// Assert multiple things together — all run even if one fails, all failures reported at once
assertAll(
    () -> assertEquals("Cosmin", account.getOwnerName()),
    () -> assertEquals(BigDecimal.ZERO, account.getBalance())
);

// AssertJ (more readable, fluent, widely preferred over plain JUnit assertions)
assertThat(account.getBalance()).isEqualByComparingTo("100.00");
assertThat(accounts).hasSize(3).extracting(Account::getOwnerName).contains("Cosmin");
```

Prefer AssertJ's fluent assertions over raw JUnit ones — better failure messages and far more expressive for collections/objects.

## 3. Mockito: Isolating the Unit Under Test

Mocking replaces a real collaborator with a fake that returns programmed responses — so a unit test for `OrderService` doesn't actually hit a database or call a payment gateway.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private InventoryClient inventoryClient;

    @Mock
    private PaymentService paymentService;

    @InjectMocks // creates OrderService, injecting the two mocks above via constructor
    private OrderService orderService;

    @Test
    void shouldFailOrderWhenStockUnavailable() {
        when(inventoryClient.getStock("SKU-1")).thenReturn(StockLevel.of(0));

        assertThrows(OutOfStockException.class, () -> orderService.placeOrder(order));

        verify(paymentService, never()).charge(any()); // never charge if stock check fails
    }

    @Test
    void shouldChargeExactAmountOnSuccess() {
        when(inventoryClient.getStock("SKU-1")).thenReturn(StockLevel.of(5));

        orderService.placeOrder(order);

        verify(paymentService).charge(eq(new BigDecimal("49.99")));
    }
}
```

`@Mock` creates a fake; `@InjectMocks` wires mocks into the class under test. `when(...).thenReturn(...)` stubs behavior; `verify(...)` asserts an interaction actually happened — use `verify` for side effects, `assert` for return values.

Common Mockito pitfalls:

- Stubbing a method that's never called leaves `UnnecessaryStubbingException` under strict stubs (Mockito's default) — a signal your test setup doesn't match what the code actually does.
- Over-mocking: if a test needs 6 mocks to set up, the class under test likely has too many responsibilities — a design smell, not just a testing inconvenience.
- Don't verify implementation details (e.g., "was this private helper called") — test observable behavior only.

## 4. Testing Exceptions and Edge Cases

```java
// Verify the exception AND its details, not just the type
InsufficientFundsException ex = assertThrows(InsufficientFundsException.class,
    () -> accountService.withdraw(account, new BigDecimal("150.00")));

assertThat(ex.getMessage()).contains("Current balance is 100.00");

// AssertJ style, if you prefer fluent chains
assertThatThrownBy(() -> accountService.withdraw(account, new BigDecimal("150.00")))
    .isInstanceOf(InsufficientFundsException.class)
    .hasMessageContaining("100.00");
```

Always test the boundary cases explicitly: empty collections, null-but-allowed fields, exact threshold values (e.g., withdrawing exactly the current balance), and concurrent/duplicate submissions if relevant.

## 5. Test Doubles: Know the Difference

| Type | Behavior |
| --- | --- |
| Dummy | Passed in but never actually used — just satisfies a parameter list. |
| Stub | Returns canned answers to calls (`when(...).thenReturn(...)`) — no behavior verification. |
| Mock | Like a stub, but you also verify specific interactions happened (`verify(...)`). |
| Fake | A working, simplified implementation (e.g., an in-memory `Map`-backed repository) — behaves correctly, just not production-grade. |

Mockito's `@Mock` technically creates objects that can act as stubs or mocks depending on whether you `verify()` them — the terminology distinction is about how you use it, not the tool.

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| One logical assertion focus per test | A test should fail for one clear reason; group related assertions with `assertAll` rather than splitting into many trivial tests. |
| Name tests to describe behavior | A failing test name should tell you what broke without opening the test body. |
| Keep tests independent and order-agnostic | Never rely on test execution order or shared mutable static state between tests. |
| Prefer AssertJ over raw JUnit assertions | Better failure messages, far more expressive for collections/objects. |
| Watch for over-mocking as a design signal | A test needing 6 mocks usually means the class under test has too many responsibilities. |
| Test observable behavior, not implementation details | Verifying "was this private helper called" couples the test to internals instead of behavior. |
| Always test boundary cases explicitly | Empty collections, null-but-allowed fields, exact threshold values, and duplicate/concurrent submissions. |
