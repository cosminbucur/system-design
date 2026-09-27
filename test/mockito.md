# Mockito Core Concepts & Practical Examples

**Mockito** is the standard mock testing framework for Java. It allows you to isolate the unit under test by substituting real dependencies with configurable mock objects.

---

## Key Concepts Overview

### 1. Mocking vs. Spying
* **Mock (`@Mock` / `mock()`):** A dummy implementation where all methods return default values (`null`, `0`, `false`, or empty collections) unless explicitly stubbed.
* **Spy (`@Spy` / `spy()`):** Wraps a real object instance. Calling a method executes actual implementation code unless you explicitly stub that specific method.

### 2. Stubbing (`when ... thenReturn`)
Defining custom behavior for mocked methods during execution (e.g., returning values, throwing exceptions, or returning dynamic responses).

### 3. Verification (`verify`)
Confirming that expected methods were called on a mock during test execution, including parameters passed and call frequency.

### 4. Argument Matchers & Captors
* **Matchers (`any()`, `eq()`, `anyString()`):** Allow flexible method matching without requiring exact instance identity or equality.
* **Argument Captor (`@Captor` / `ArgumentCaptor`):** Captures argument instances passed into mocked methods so you can perform assertions on their internal state.

### 5. Dependency Injection (`@InjectMocks`)
Automatically instantiates the target test class and injects fields annotated with `@Mock` or `@Spy`.

---

## Practical Example: Complete Setup with JUnit 5

Suppose we are testing a `UserService` that depends on `UserRepository` and `EmailService`.

### Target Implementation

```java
public class UserService {
    private final UserRepository repository;
    private final EmailService emailService;

    public UserService(UserRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }

    public User registerUser(String name, String email) {
        if (repository.existsByEmail(email)) {
            throw new IllegalArgumentException("Email already taken");
        }

        User user = new User(name, email);
        User savedUser = repository.save(user);
        emailService.sendWelcomeEmail(email);

        return savedUser;
    }
}
```

---

### Mockito Unit Test Suite

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class) // Initializes Mockito annotations
class UserServiceTest {

    @Mock
    private UserRepository repository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService; // Injects repository and emailService

    @Captor
    private ArgumentCaptor<User> userCaptor;

    // 1. Stubbing and Verification Test
    @Test
    void registerUser_ShouldSaveAndReturnUser_WhenEmailIsNotTaken() {
        // Arrange (Stubbing)
        String email = "john@example.com";
        User expectedUser = new User("John", email);

        when(repository.existsByEmail(email)).thenReturn(false);
        when(repository.save(any(User.class))).thenReturn(expectedUser);

        // Act
        User actualUser = userService.registerUser("John", email);

        // Assert
        assertNotNull(actualUser);
        assertEquals("John", actualUser.getName());

        // Verification: Check interaction frequency
        verify(emailService, times(1)).sendWelcomeEmail(email);
    }

    // 2. Exception Handling Verification
    @Test
    void registerUser_ShouldThrowException_WhenEmailAlreadyExists() {
        // Arrange
        String email = "existing@example.com";
        when(repository.existsByEmail(email)).thenReturn(true);

        // Act & Assert
        assertThrows(IllegalArgumentException.class, () -> {
            userService.registerUser("Jane", email);
        });

        // Verification: Ensure side-effects didn't execute
        verify(repository, never()).save(any());
        verify(emailService, never()).sendWelcomeEmail(anyString());
    }

    // 3. Argument Captor Test
    @Test
    void registerUser_ShouldPassCorrectUserToRepository() {
        // Arrange
        when(repository.existsByEmail(anyString())).thenReturn(false);

        // Act
        userService.registerUser("Alice", "alice@example.com");

        // Verify & Capture arguments
        verify(repository).save(userCaptor.capture());
        User capturedUser = userCaptor.getValue();

        assertEquals("Alice", capturedUser.getName());
        assertEquals("alice@example.com", capturedUser.getEmail());
    }
}
```

---

## Quick Reference API Cheat-Sheet

| Operation | Syntax | Description |
| :--- | :--- | :--- |
| **Mock Creation** | `@Mock` / `mock(Class.class)` | Creates full mock instance. |
| **Spy Creation** | `@Spy` / `spy(new Class())` | Creates partial mock wrapping real instance. |
| **Inject Mocks** | `@InjectMocks` | Injects mocks into tested target object. |
| **Stub Method** | `when(mock.method()).thenReturn(value)` | Predefines invocation return value. |
| **Throw Exception**| `when(mock.method()).thenThrow(Ex.class)` | Configures method to throw exception. |
| **Verify Call** | `verify(mock, times(n)).method()` | Asserts method call count. |
| **Never Called** | `verify(mock, never()).method()` | Asserts method was not invoked. |
| **Void Methods** | `doNothing().when(mock).voidMethod()` | Stubbing behavior on `void` return methods. |