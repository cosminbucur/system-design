Clean code is code optimized for the next person reading it — often you, six months later, with no memory of why you wrote it this way. It's not about following rules mechanically; it's about reducing the mental effort required to understand and safely change the code. Every pattern below exists to serve that one goal, not as a style preference for its own sake.

## 1. Naming — The Cheapest Form of Documentation

A good name eliminates the need for a comment explaining what something is. Vague or misleading names force every reader to go dig through the implementation to understand intent.

```java
// BAD: what is "d"? what does "process" actually do? what units?
int d;
void process(List<Object> l) { ... }

// GOOD: self-explanatory, no comment needed
int daysSinceLastLogin;
void archiveExpiredSessions(List<Session> sessions) { ... }
```

Guidelines:

- Use intention-revealing names — `elapsedTimeInDays` over `d`, `isActive()` over `flag()`.
- Avoid disinformation — don't name something `accountList` if it's actually a `Set`, and don't abbreviate in ways that could mean something else (`hp` could be "hit points" or "horsepower").
- Use pronounceable, searchable names — `genymdhms` fails both; `generationTimestamp` fails neither.
- Class names should be nouns (`Invoice`, `PaymentProcessor`); method names should be verbs (`calculateTotal()`, `send()`).
- Match the vocabulary the business actually uses — this is the same principle as ubiquitous language in DDD; a mismatch between code names and domain language is itself a form of unclear naming.

## 2. Functions — Small, and Doing One Thing

A function should do one thing, do it well, and do only that thing. If you can extract a meaningful chunk of a function into a well-named helper, that's usually a sign it was doing more than one thing.

```java
// BAD: one method doing validation, calculation, AND persistence
public void processOrder(Order order) {
    if (order.getItems().isEmpty()) throw new IllegalArgumentException("Empty order");
    if (order.getCustomer() == null) throw new IllegalArgumentException("No customer");

    BigDecimal total = BigDecimal.ZERO;
    for (OrderLine line : order.getLines()) {
        total = total.add(line.getPrice().multiply(BigDecimal.valueOf(line.getQuantity())));
    }
    order.setTotal(total);

    orderRepository.save(order);
    emailService.sendConfirmation(order);
}

// GOOD: each method has one clear responsibility, and processOrder reads like a summary
public void processOrder(Order order) {
    validate(order);
    order.setTotal(calculateTotal(order));
    orderRepository.save(order);
    emailService.sendConfirmation(order);
}

private void validate(Order order) {
    if (order.getItems().isEmpty()) throw new IllegalArgumentException("Empty order");
    if (order.getCustomer() == null) throw new IllegalArgumentException("No customer");
}

private BigDecimal calculateTotal(Order order) {
    return order.getLines().stream()
        .map(line -> line.getPrice().multiply(BigDecimal.valueOf(line.getQuantity())))
        .reduce(BigDecimal.ZERO, BigDecimal::add);
}
```

A useful heuristic: if you have to use "and" to describe what a function does ("validates and calculates and saves"), it's doing more than one thing. Also prefer fewer parameters — beyond 2-3, group related parameters into an object (this is often where a value object naturally emerges, e.g., bundling `street`/`city`/`zip` into an `Address`).

## 3. Comments — A Last Resort, Not a First Instinct

A comment is often a sign the code itself failed to communicate clearly — the fix is usually to rename or restructure, not to explain the confusing code as-is.

```java
// BAD: comment explains WHAT, which the code should already say
// check if user is eligible
if (user.getAge() >= 18 && user.getAccountStatus() == ACTIVE && !user.isSuspended()) { ... }

// GOOD: extracted into a well-named method — no comment needed, and it's reusable
if (isEligibleForCheckout(user)) { ... }

private boolean isEligibleForCheckout(User user) {
    return user.getAge() >= 18 && user.getAccountStatus() == ACTIVE && !user.isSuspended();
}
```

When a comment IS worth writing: explaining _why_, not _what_ — a non-obvious business rule, a workaround for a specific bug/library quirk, or a warning about a subtle consequence of changing the code.

```java
// GOOD comment: explains a non-obvious WHY that the code alone can't convey
// Stripe requires amounts in the smallest currency unit (cents), not dollars —
// see https://stripe.com/docs/currencies#zero-decimal
long amountInCents = amount.multiply(BigDecimal.valueOf(100)).longValueExact();
```

Avoid commented-out code and changelog-style comments in the code itself — version control already has that history; a comment like `// added by John for ticket JIRA-123` rots the moment the ticket is closed and the context is gone.

## 4. Error Handling as a Distinct Concern

Mixing business logic with error handling logic makes both harder to read. Keep the "happy path" clear, and let exceptions carry meaningful context.

```java
// BAD: business logic buried inside error-handling noise
public Account findAccount(String id) {
    Account account = null;
    try {
        account = repository.findById(id);
        if (account == null) {
            log.error("not found");
            return null; // caller now has to null-check — easy to forget
        }
    } catch (Exception e) {
        log.error("error", e);
        return null;
    }
    return account;
}

// GOOD: clear happy path, explicit failure signaled through the type system
public Optional<Account> findAccount(String id) {
    return repository.findById(id);
}
```

Prefer returning `Optional<T>` (or throwing a specific exception) over returning `null` — `null` forces every caller to remember a check that the compiler can't enforce, while `Optional` makes "this might be absent" visible in the method signature itself.

## 5. SOLID Principles — Briefly

These are widely known by name but easy to apply too rigidly; each solves a real problem when the codebase has actually grown to need it.

| Principle             | Idea                                                        | Java example signal                                                                                                             |
| --------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Single Responsibility | A class should have one reason to change                    | A `ReportGenerator` that also handles email sending and file compression has three reasons to change, not one                   |
| Open/Closed           | Open for extension, closed for modification                 | Adding a new `DiscountStrategy` shouldn't require editing existing strategy classes                                             |
| Liskov Substitution   | A subtype must be usable anywhere its base type is expected | A `Square extends Rectangle` that overrides `setWidth` to also change height breaks callers expecting independent width/height  |
| Interface Segregation | Prefer several small, focused interfaces over one large one | A `Worker` interface forcing every implementer to implement `eat()` even for a `RobotWorker` is a signal to split the interface |
| Dependency Inversion  | Depend on abstractions, not concrete implementations        | A service depending on an `OrderRepository` interface, not a concrete `JpaOrderRepository`, stays testable and swappable        |

Don't apply all five preemptively to every class — a small, stable utility class doesn't need an interface "just in case." Apply them when a concrete pain point shows up (a class keeps changing for unrelated reasons, a test needs six mocks).

## 6. Code Smells Worth Recognizing

| Smell               | What it looks like                                                                 | Usual fix                                                        |
| ------------------- | ---------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Long method         | A method that scrolls past one screen                                              | Extract smaller, well-named private methods                      |
| Long parameter list | `createUser(String, String, String, int, boolean, boolean)`                        | Group related parameters into a value object                     |
| Primitive obsession | Passing raw `String`/`BigDecimal` for concepts with rules (money, email, currency) | Introduce a value object                                         |
| Feature envy        | A method that mostly calls getters on another object rather than its own fields    | Move the method onto the class whose data it's actually using    |
| Shotgun surgery     | One conceptual change requires touching many unrelated classes                     | Consolidate the responsibility into fewer, more cohesive classes |
| Duplicate code      | The same logic copy-pasted in multiple places                                      | Extract a shared method/class — but see the DRY caution first    |
| God class           | One class that does almost everything (`Manager`, `Utils`, `Helper`)               | Split by responsibility, guided by Single Responsibility         |

## 7. DRY — But Don't Force It Prematurely

Don't Repeat Yourself is a real principle, but two pieces of _similar-looking_ code aren't necessarily the same _concept_ — merging them prematurely can create a false abstraction that has to be pulled apart again once the two use cases diverge.

```java
// These look duplicated today...
BigDecimal orderDiscount = price.multiply(new BigDecimal("0.9"));
BigDecimal loyaltyDiscount = price.multiply(new BigDecimal("0.9"));

// ...but if "loyalty discount" and "order discount" are conceptually different business
// rules that just happen to both be 10% today, forcing them into one shared
// applyTenPercentDiscount() method couples two things that will likely need to
// evolve independently the moment the business changes one but not the other.
```

Rule of thumb: three similar-looking occurrences is a reasonable trigger to consider extracting a shared abstraction (the "rule of three") — two is often still a coincidence. Always ask whether the duplication represents the _same business concept_ before merging it, not just visually similar code.

## 8. The Boy Scout Rule

Leave the code a little cleaner than you found it — a small rename, extracting one confusing block, deleting dead code — as an incidental part of whatever change you're already making, not a separate large refactoring effort.

This works because it spreads improvement cost across normal feature work instead of requiring a dedicated "refactor everything" project (which is hard to prioritize and often never happens) — but it also means: don't let it balloon a small bug fix into an unrelated rewrite of a file. Keep the scope of "cleaning" proportional to the change you're already making.

## 9. Consistency Over Personal Preference

In a shared codebase, a consistent (if imperfect) style beats a "more correct" one applied inconsistently — a reader constantly re-adjusting to different conventions per file pays a real cognitive tax. Follow the existing codebase's conventions (naming style, formatting, package structure) even if you'd have chosen differently starting from scratch; raise the convention itself for discussion separately from the change you're making, rather than deviating unilaterally in one PR.

## 10. Best Practices

| Practice                                                                                  | Recommendation                                                                                  |
| ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Name things by intent, not implementation                                                 | A reader should understand what something represents without reading its body.                  |
| Keep functions small and single-purpose                                                   | If describing a function needs "and," it's doing more than one thing.                           |
| Prefer restructuring code over explaining it with a comment                               | Reserve comments for non-obvious _why_, not a description of _what_ the code already shows.     |
| Make absence explicit with `Optional`, not `null`                                         | Forces callers to handle the missing case, rather than relying on memory.                       |
| Apply SOLID/design patterns when the pain shows up, not preemptively                      | A pattern added "just in case" is speculative complexity.                                       |
| Verify duplication is conceptual, not just visual, before extracting a shared abstraction | A false abstraction merging two unrelated rules is often worse than the duplication it removed. |
| Leave code slightly better than you found it, scoped to your change                       | Incremental cleanup compounds over time without needing a dedicated rewrite effort.             |
| Match the existing codebase's conventions                                                 | Consistency reduces cognitive load more than any single "better" individual choice.             |
