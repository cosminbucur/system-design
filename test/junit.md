# JUnit 5 Comprehensive Guide

JUnit 5 (also known as **JUnit Jupiter**) is the standard testing framework for modern Java development. It is composed of three main sub-projects:

1. **JUnit Platform**: The foundation for launching testing frameworks on the JVM.
2. **JUnit Jupiter**: The combination of the new programming model and extension model for writing tests and extensions in JUnit 5.
3. **JUnit Vintage**: Provides a `TestEngine` for running JUnit 3 and JUnit 4 based tests on the platform.

---

## 1. Core Concepts

### Key Annotations

| Annotation | Description |
| :--- | :--- |
| `@Test` | Denotes a method as a test case. |
| `@BeforeEach` | Executed before **each** `@Test` method in the current class. Used to set up test state/fixtures. |
| `@AfterEach` | Executed after **each** `@Test` method in the current class. Used to clean up test state. |
| `@BeforeAll` | Executed **once** before all test methods in the current class. Must be `static` by default. |
| `@AfterAll` | Executed **once** after all test methods in the current class. Must be `static` by default. |
| `@DisplayName` | Declares a custom display name for the test class or test method. |
| `@Disabled` | Used to disable a test class or test method. |
| `@Nested` | Denotes an inner class as an embedded, non-static test class for logical grouping. |
| `@ParameterizedTest` | Denotes that a test method should be executed multiple times with different arguments. |

### Assertion Types (`org.junit.jupiter.api.Assertions.*`)

* **Equality / Identity:** `assertEquals(expected, actual)`, `assertNotEquals(unexpected, actual)`, `assertSame(expected, actual)`
* **Conditions:** `assertTrue(booleanCondition)`, `assertFalse(booleanCondition)`
* **Nullability:** `assertNull(object)`, `assertNotNull(object)`
* **Exceptions:** `assertThrows(ExpectedException.class, () -> executableCode)`
* **Grouped Assertions:** `assertAll("groupName", () -> assertEquals(...), () -> assertTrue(...))`

---

## 2. Best Practices

1. **Follow the AAA Pattern (Arrange-Act-Assert)**
   * **Arrange:** Set up objects, mock dependencies, and initialize data.
   * **Act:** Invoke the method or code block under test.
   * **Assert:** Verify that the output or side effects match expectations.

2. **Ensure Test Isolation (Nondeterminism Prevention)**
   * Tests must be completely independent. Execution order should not matter.
   * Do not share mutable state between tests without clearing it in `@BeforeEach` or `@AfterEach`.

3. **Test One Concept Per Method**
   * Keep tests focused. Avoid long test methods that attempt to exercise an entire multi-step integration flow unless specifically building an integration test.

4. **Use Expressive and Descriptive Names**
   * Use `@DisplayName` to state business expectations clearly (e.g., `"Should throw exception when withdrawal exceeds current balance"` instead of `testWithdraw()`).

5. **Prefer `assertThrows` over `try-catch`**
   * Instead of using `try { ... fail(); } catch (Exception e) { ... }`, use `assertThrows()` to succinctly verify exception types and messages.

6. **Utilize Grouped Assertions (`assertAll`)**
   * When checking multiple fields on an object, `assertAll()` ensures that all assertions are evaluated even if one of them fails, providing full diagnostic information in a single run.

---

## 3. Practical Example

### Class Under Test (`BankAccount.java`)

```java
public class BankAccount {
    private double balance;

    public BankAccount(double initialBalance) {
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.balance = initialBalance;
    }

    public double deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Deposit amount must be greater than zero");
        }
        this.balance += amount;
        return this.balance;
    }

    public double withdraw(double amount) {
        if (amount > balance) {
            throw new IllegalStateException("Insufficient funds");
        }
        this.balance -= amount;
        return this.balance;
    }

    public double getBalance() {
        return balance;
    }
}
```

---

### Test Suite (`BankAccountTest.java`)

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("Bank Account Unit Tests")
class BankAccountTest {

    private BankAccount account;

    @BeforeEach
    void setUp() {
        // Arrange fixture before every test
        account = new BankAccount(100.0);
    }

    @Nested
    @DisplayName("Constructor Tests")
    class ConstructorTests {

        @Test
        @DisplayName("Should create account with valid initial balance")
        void createAccountWithValidBalance() {
            assertEquals(100.0, account.getBalance());
        }

        @Test
        @DisplayName("Should fail creation when initial balance is negative")
        void failOnNegativeInitialBalance() {
            IllegalArgumentException exception = assertThrows(
                IllegalArgumentException.class,
                () -> new BankAccount(-50.0)
            );
            assertEquals("Initial balance cannot be negative", exception.getMessage());
        }
    }

    @Nested
    @DisplayName("Deposit Operations")
    class DepositTests {

        @Test
        @DisplayName("Should successfully deposit a valid amount")
        void depositValidAmount() {
            // Act
            double newBalance = account.deposit(50.0);

            // Assert
            assertAll("Verify deposit outcomes",
                () -> assertEquals(150.0, newBalance, "Returned balance should reflect deposit"),
                () -> assertEquals(150.0, account.getBalance(), "Account state balance should be updated")
            );
        }

        @ParameterizedTest
        @ValueSource(doubles = {0.0, -10.0, -100.0})
        @DisplayName("Should throw exception when depositing zero or negative amounts")
        void depositInvalidAmount(double invalidAmount) {
            // Act & Assert
            assertThrows(
                IllegalArgumentException.class,
                () -> account.deposit(invalidAmount)
            );
        }
    }

    @Nested
    @DisplayName("Withdrawal Operations")
    class WithdrawalTests {

        @Test
        @DisplayName("Should successfully withdraw available funds")
        void withdrawValidAmount() {
            // Act
            double remainingBalance = account.withdraw(40.0);

            // Assert
            assertEquals(60.0, remainingBalance);
            assertEquals(60.0, account.getBalance());
        }

        @Test
        @DisplayName("Should throw exception when withdrawing more than current balance")
        void withdrawInsufficientFunds() {
            // Act & Assert
            IllegalStateException exception = assertThrows(
                IllegalStateException.class,
                () -> account.withdraw(150.0)
            );

            assertEquals("Insufficient funds", exception.getMessage());
        }
    }
}
```