# Spy vs Mock in Mockito: When to Use and Examples

In Mockito, **Mocks** and **Spies** are both test doubles used to isolate code, but they behave fundamentally differently by default. Understanding when to use each is crucial for writing robust and maintainable unit tests.

---

## Core Differences at a Glance

| Feature | Mock (`@Mock` / `mock()`) | Spy (`@Spy` / `spy()`) |
| :--- | :--- | :--- |
| **What it wraps** | A synthetic, empty shell class. | A **real object** instance. |
| **Default Behavior** | Returns **default values** (`null`, `0`, `false`) for all method calls unless explicitly stubbed. | Executes the **real method implementation** unless explicitly stubbed (also known as a *partial mock*). |
| **Primary Use Case** | Complete isolation; replacing external or complex dependencies. | Testing classes where you want to keep real logic for most methods and override only a few. |

---

## When to Use Which?

### 🟢 Use a **Mock** When:
* You want **complete control** over a dependency.
* The dependency talks to external systems (databases, REST APIs, message queues) and you want to avoid side effects, slowness, or non-determinism.
* You don't care about the internal state of the dependency; you only care that specific methods are called with specific parameters.

### 🟡 Use a **Spy** When:
* You are dealing with **legacy code** where rewriting everything into clean components isn't immediately feasible.
* You want a class to execute its **real business logic**, but you need to intercept or mock a specific internal method (e.g., bypassing a database call or a heavy internal computation inside that same class).
* You are testing a class that has complex state management, and you need the underlying object to track changes accurately (like a real `List` or a utility helper).

---

## Code Examples

### 1. Example of a Mock (`mock()`)
Imagine a `UserService` that depends on a `UserRepository`. We want to test the service without hitting a real database.

```java
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

class UserServiceTest {

    @Test
    void testFindUser() {
        // Create a mock repository
        UserRepository mockRepo = mock(UserRepository.class);
        
        // Stubbing: Define what happens when findById is called
        User dummyUser = new User(1L, "Cosmin");
        when(mockRepo.findById(1L)).thenReturn(dummyUser);

        UserService userService = new UserService(mockRepo);
        
        // Execute
        User result = userService.getUserDetails(1L);

        // Assert
        assertEquals("Cosmin", result.getName());
        // The real database code inside UserRepository was never executed.
        verify(mockRepo, times(1)).findById(1L);
    }
}
```

### 2. Example of a Spy (`spy()`)
Imagine a `Calculator` class where you want to use the **real** implementation of most methods, but you want to mock one specific helper method (e.g., `isAuthorized()`, which might normally check an external security context).

```java
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    @Test
    void testSpyPartialMock() {
        // Create a spy wrapping a real Calculator instance
        Calculator calculator = new Calculator();
        Calculator spyCalculator = spy(calculator);

        // Stub ONLY the internal security check method, let everything else run normally
        doReturn(true).when(spyCalculator).isAuthorized();

        // Execution:
        // 1. isAuthorized() is stubbed and returns true.
        // 2. add(10, 5) uses the REAL implementation of Calculator because it wasn't stubbed.
        int result = spyCalculator.calculateSumWithAuthorization(10, 5);

        assertEquals(15, result);
        
        // Verify the real add method actually ran
        verify(spyCalculator).add(10, 5);
    }
}
```

---

## ⚠️ Important Gotcha with Spies: `doReturn()` vs `when()`

When stubbing a spy, using the standard `when(spy.method()).thenReturn(value)` syntax can cause unexpected behavior because **Mockito evaluates the inner method call first** before applying the stubbing. This means the real method will execute and might throw an exception (like a `NullPointerException`).

To safely stub a spy, always use the **`doReturn(...).when(spy).method()`** syntax:

```java
// ❌ Dangerous (may execute the real method and crash)
when(spyCalculator.isAuthorized()).thenReturn(true);

// ✅ Safe (bypasses the real method execution completely)
doReturn(true).when(spyCalculator).isAuthorized();