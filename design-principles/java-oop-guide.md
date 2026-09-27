# Object-Oriented Programming (OOP) in Java

Object-Oriented Programming (OOP) organizes code around **objects** (data structures containing fields) and **methods** (actions performed on that data). Java implements this paradigm through four fundamental pillars, along with modern features like **sealed classes** that offer precise control over class hierarchies.

---

## 1. The Four Pillars of OOP in Java

### Encapsulation
Encapsulation is the practice of bundling fields and methods into a single unit (class) while restricting direct access to internal state. Access is controlled via modifiers (`private`, `protected`, `public`) and accessed or modified through public getter and setter methods.

```java
public class BankAccount {
    private double balance; // Hidden internal state

    public BankAccount(double initialBalance) {
        if (initialBalance >= 0) {
            this.balance = initialBalance;
        }
    }

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount > 0) {
            this.balance += amount;
        }
    }
}
```

### Abstraction
Abstraction exposes only essential features while hiding implementation complexity using `abstract` classes or `interface` types.

```java
public interface PaymentProcessor {
    void processPayment(double amount); // Defines WHAT, not HOW
}

public class StripeProcessor implements PaymentProcessor {
    @Override
    public void processPayment(double amount) {
        // Specific Stripe API implementation logic
    }
}
```

### Inheritance
Inheritance allows one class (subclass) to acquire attributes and methods from another (superclass) using the `extends` keyword, promoting code reusability.

```java
public class Vehicle {
    protected String brand = "Generic";
    
    public void start() {
        System.out.println("Vehicle starting...");
    }
}

public class Car extends Vehicle {
    private int doors = 4;
    // Inherits brand and start() automatically
}
```

### Polymorphism
Polymorphism allows objects to take on multiple forms depending on the context.
* **Compile-time (Overloading):** Multiple methods with the same name but different parameter signatures in the same class.
* **Runtime (Overriding):** A subclass provides a specific implementation of a method declared in its superclass or interface.

```java
public class Shape {
    public void draw() { System.out.println("Drawing shape"); }
}

public class Circle extends Shape {
    @Override
    public void draw() { System.out.println("Drawing circle"); } // Overriding
}
```

---

## 2. Modern Java: Sealed Classes & Interfaces

Introduced to provide precise control over inheritance, **Sealed Classes and Interfaces** (`sealed`) allow you to specify exactly which classes are permitted to extend or implement them. This prevents arbitrary, external subclassing while maintaining type safety.

### Rules for Sealed Hierarchies
1. **Permitted Subclasses:** Permitted classes must extend or implement the sealed type directly.
2. **Subclass Modifiers:** Every subclass of a sealed class MUST explicitly declare one of three modifiers:
   * `final`: Cannot be extended any further.
   * `sealed`: Can be extended, but only by its own permitted list.
   * `non-sealed`: Opens the hierarchy back up for unconstrained extension.
3. **Module/Package Bounds:** The sealed class and its permitted subclasses must belong to the same module (or same package if in an unnamed module).

### Code Example

```java
// Top-level sealed interface
public sealed interface PaymentMethod permits CreditCard, PayPal, CryptoPayment {}

// Final subclass: Hierarchy stops here
public final class CreditCard implements PaymentMethod {
    private String cardNumber;
    public CreditCard(String cardNumber) { this.cardNumber = cardNumber; }
}

// Sealed subclass: Has its own strictly defined permitted children
public sealed class PayPal implements PaymentMethod permits PayPalExpress {
    private String email;
    public PayPal(String email) { this.email = email; }
}

public final class PayPalExpress extends PayPal {
    public PayPalExpress(String email) { super(email); }
}

// Non-sealed subclass: Re-opens the hierarchy to any caller
public non-sealed class CryptoPayment implements PaymentMethod {
    private String walletAddress;
    public CryptoPayment(String walletAddress) { this.walletAddress = walletAddress; }
}
```

### Pattern Matching with Sealed Classes
Sealed hierarchies work seamlessly with Java's enhanced `switch` expressions and pattern matching. Because the compiler knows all possible subtypes of a sealed type, it enforces exhaustive checks **without needing a `default` clause**:

```java
public String processPayment(PaymentMethod method) {
    return switch (method) {
        case CreditCard cc -> "Processing credit card transaction";
        case PayPal pp     -> "Processing PayPal transaction";
        case CryptoPayment crypto -> "Processing blockchain transaction";
        // No 'default' block required! If a new permitted type is added, 
        // the compiler forces you to handle it here.
    };
}
```

---

## 3. Best Practices for OOP in Java

| Practice | Explanation |
| :--- | :--- |
| **Favor Composition Over Inheritance** | Use delegation (`has-a`) instead of rigid class inheritance (`is-a`) to avoid fragile base class issues and maintain flexibility. |
| **Program to Interfaces, Not Implementations** | Declare variable types as interface types (e.g., `List<String> list = new ArrayList<>()`) to decouple code from concrete implementations. |
| **Follow SOLID Principles** | Structure classes according to Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion principles. |
| **Prefer Immutability** | Use `final` fields, Java `record` types, and immutable objects to eliminate thread-safety bugs and unintended side effects. |
| **Restrict Fields & Mutators** | Keep fields `private` and only expose setters when state mutation is explicitly required. |
| **Use Sealed Types for Closed Domains** | Represent fixed domain models (e.g., state machines, API results, AST nodes) using `sealed` hierarchies for compiler-level exhaustiveness checking. |