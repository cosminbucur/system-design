Behavioral patterns are reusable solutions for how objects interact and distribute responsibility — encapsulating an algorithm, notifying dependents of a change, or passing a request along until something handles it, without hardcoding that interaction into a rigid, hard-to-change shape.

## 1. Strategy

Encapsulates interchangeable algorithms behind a common interface, selected at runtime. In modern Java, a lambda often replaces the whole class hierarchy.

```java
public interface DiscountStrategy {
    BigDecimal apply(BigDecimal price);
}

// old-style: separate classes per strategy
public class BlackFridayDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) { return price.multiply(new BigDecimal("0.7")); }
}

// modern: just pass a lambda, no class needed
DiscountStrategy noDiscount = price -> price;
DiscountStrategy tenPercentOff = price -> price.multiply(new BigDecimal("0.9"));

BigDecimal finalPrice = tenPercentOff.apply(originalPrice);
```

## 2. Observer

Notifies a set of dependents automatically when an object's state changes — the basis of Java's event listeners and Spring's `ApplicationEventPublisher`.

```java
@Component
public class AccountService {
    private final ApplicationEventPublisher publisher;

    public void closeAccount(Long id) {
        // ... close logic
        publisher.publishEvent(new AccountClosedEvent(id));
    }
}

@Component
public class AuditListener {
    @EventListener
    public void onAccountClosed(AccountClosedEvent event) {
        // react without the publisher knowing this listener exists
    }
}
```

## 3. Template Method

Defines the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the overall structure.

```java
public abstract class DataImporter {
    public final void importData(String source) { // template — final so subclasses can't reorder steps
        Data raw = readData(source);
        Data validated = validate(raw);
        save(validated);
    }
    protected abstract Data readData(String source);
    protected abstract Data validate(Data raw);
    protected void save(Data data) { repository.save(data); } // default step, overridable
}
```

## 4. Chain of Responsibility

Passes a request along a chain of handlers until one handles it — Spring's servlet `Filter` chain and Spring Security's filter chain are real-world examples.

```java
public abstract class ValidationHandler {
    protected ValidationHandler next;
    public ValidationHandler setNext(ValidationHandler next) { this.next = next; return next; }

    public void handle(Request request) {
        if (!isValid(request)) throw new ValidationException();
        if (next != null) next.handle(request);
    }
    protected abstract boolean isValid(Request request);
}
```

## 5. Command

Turns a request itself into an object — bundling the action, its receiver, and its parameters together so it can be queued, logged, undone, or passed around like any other value.

```java
public interface Command {
    void execute();
    void undo();
}

public class TransferFundsCommand implements Command {
    private final Account from, to;
    private final BigDecimal amount;

    public TransferFundsCommand(Account from, Account to, BigDecimal amount) {
        this.from = from; this.to = to; this.amount = amount;
    }

    public void execute() { from.withdraw(amount); to.deposit(amount); }
    public void undo() { from.deposit(amount); to.withdraw(amount); } // reverses execute() exactly
}

// A caller can queue, log, or undo commands without knowing what each one actually does
Deque<Command> history = new ArrayDeque<>();
Command transfer = new TransferFundsCommand(accountA, accountB, amount);
transfer.execute();
history.push(transfer);
// ... later
history.pop().undo();
```

This is what makes undo/redo stacks, job queues, and transactional macro-recording all possible — the request is a first-class object, not just an immediate method call that's already forgotten once it returns.

## 6. Iterator

Provides a way to access elements of a collection sequentially without exposing its underlying structure — Java's own `Iterable`/`Iterator` interfaces are this pattern built directly into the language.

```java
public class RingBuffer<T> implements Iterable<T> {
    private final T[] items;
    private int start, count;

    @Override
    public Iterator<T> iterator() {
        return new Iterator<>() {
            private int index = 0;
            public boolean hasNext() { return index < count; }
            public T next() { return items[(start + index++) % items.length]; }
        };
    }
}

// Caller iterates without knowing anything about the ring-buffer's wraparound indexing
for (String item : ringBuffer) {
    System.out.println(item);
}
```

Because this pattern is baked into the language via `for-each`, most Java developers use it constantly without ever thinking of it by its Gang-of-Four name.

## 7. Mediator

Centralizes how a set of objects communicate, so they refer only to the mediator instead of directly to each other — reducing a tangle of many-to-many references down to a hub-and-spoke shape.

```java
public interface ChatRoomMediator {
    void sendMessage(String message, User sender);
}

public class ChatRoom implements ChatRoomMediator {
    private final List<User> participants = new ArrayList<>();
    public void register(User user) { participants.add(user); }

    public void sendMessage(String message, User sender) {
        for (User user : participants) {
            if (user != sender) user.receive(message); // sender never talks to other users directly
        }
    }
}
```

Without a mediator, every `User` would need a reference to every other `User` — with it, each `User` only knows about the `ChatRoom`, and participants can be added or removed without touching any other participant's code.

## 8. Memento

Captures an object's internal state so it can be restored later, without exposing that state's internal representation to the code doing the saving/restoring — the standard building block behind undo functionality and snapshots.

```java
public class EditorState { // the memento — an opaque, immutable snapshot
    private final String content;
    EditorState(String content) { this.content = content; } // package-private constructor limits who can create one
    String getContent() { return content; }
}

public class TextEditor {
    private String content = "";
    public void type(String text) { content += text; }
    public EditorState save() { return new EditorState(content); }
    public void restore(EditorState state) { this.content = state.getContent(); }
}

// A separate caretaker holds the history without ever inspecting what's inside a snapshot
Deque<EditorState> history = new ArrayDeque<>();
history.push(editor.save());
editor.type("more text");
editor.restore(history.pop()); // back to the previous content, exactly
```

## 9. State

Lets an object change its behavior by switching its internal state object, rather than branching on a status flag throughout every method — each state becomes its own class implementing shared behavior differently.

```java
public interface OrderState {
    OrderState next(Order order);
    String describe();
}

public class PendingState implements OrderState {
    public OrderState next(Order order) { return new ShippedState(); }
    public String describe() { return "Pending"; }
}

public class ShippedState implements OrderState {
    public OrderState next(Order order) { return new DeliveredState(); }
    public String describe() { return "Shipped"; }
}

public class Order {
    private OrderState state = new PendingState();
    public void advance() { state = state.next(this); }
    public String getStatus() { return state.describe(); }
}
```

This replaces a sprawling `if (status == PENDING) { ... } else if (status == SHIPPED) { ... }` scattered across many methods with one class per state, each owning exactly its own transition logic — adding a new state means adding a class, not editing every existing conditional (the same benefit already seen with the Open/Closed Principle).

## 10. Visitor

Lets you add new operations to a family of classes without modifying those classes — the operation "visits" each type and the correct behavior is chosen via double dispatch (the element accepts the visitor, then calls back the visitor method matching its own type).

```java
public interface ShapeVisitor {
    void visit(Circle circle);
    void visit(Square square);
}

public interface Shape {
    void accept(ShapeVisitor visitor);
}

public class Circle implements Shape {
    public double radius;
    public void accept(ShapeVisitor visitor) { visitor.visit(this); } // calls the Circle-specific overload
}

public class Square implements Shape {
    public double side;
    public void accept(ShapeVisitor visitor) { visitor.visit(this); } // calls the Square-specific overload
}

public class AreaCalculator implements ShapeVisitor {
    public double totalArea = 0;
    public void visit(Circle circle) { totalArea += Math.PI * circle.radius * circle.radius; }
    public void visit(Square square) { totalArea += square.side * square.side; }
}
```

Visitor is the direct tradeoff-inverse of adding a method to each `Shape` class: it's easy to add a new *operation* (a new Visitor implementation) without touching `Circle`/`Square`, but adding a new *shape* means updating every existing Visitor — worth reaching for specifically when new operations are added more often than new types.

## 11. Interpreter

Defines a representation for a simple grammar and an interpreter to evaluate sentences in it — genuinely useful for small, custom expression languages (a rules engine, a filter query syntax), but rare in ordinary application code, where reaching for an existing parser library or embedded scripting engine is almost always the better call.

```java
public interface Expression {
    int evaluate();
}

public class NumberExpression implements Expression {
    private final int value;
    public NumberExpression(int value) { this.value = value; }
    public int evaluate() { return value; }
}

public class AddExpression implements Expression {
    private final Expression left, right;
    public AddExpression(Expression left, Expression right) { this.left = left; this.right = right; }
    public int evaluate() { return left.evaluate() + right.evaluate(); }
}

// Represents "3 + 4" as a tree of expression objects, then evaluates it
Expression expr = new AddExpression(new NumberExpression(3), new NumberExpression(4));
int result = expr.evaluate(); // 7
```

## 12. Pattern Selection Cheat Sheet

| Problem | Reach for |
| --- | --- |
| Swap an algorithm at runtime | Strategy (often just a lambda/functional interface) |
| Notify multiple listeners on a state change | Observer / event publishing |
| Same algorithm skeleton, different steps per subclass | Template Method |
| Sequence of handlers, any of which might process the request | Chain of Responsibility |
| A request needs to be queued, logged, or undone | Command |
| Traverse a collection without exposing its internal structure | Iterator |
| Many objects need to communicate without referencing each other directly | Mediator |
| Need to save/restore an object's state without exposing its internals | Memento |
| Behavior changes based on internal state, without a sprawling conditional | State |
| Add new operations to a class family without modifying those classes | Visitor |
| Evaluating a small custom expression grammar | Interpreter (rare — prefer an existing parser/scripting library first) |

## 13. Best Practices

| Practice | Recommendation |
| --- | --- |
| Don't force a pattern | Behavioral patterns solve specific recurring interaction problems — applying one where the problem doesn't exist adds indirection for no benefit. |
| Use lambdas for single-method strategies | A functional interface + lambda often replaces an entire Strategy class hierarchy in modern Java. |
| Recognize patterns already in the JDK/Spring | `Comparator` (Strategy), `for-each`/`Iterator` (Iterator), `ApplicationEventPublisher` (Observer), servlet `Filter` (Chain of Responsibility) — you're likely already using these. |
| Reach for State when a status flag is branched on in many methods | Replacing scattered conditionals with one class per state means adding a new state is adding a class, not editing every existing branch. |
| Reach for Visitor only when new operations are far more frequent than new types | It inverts the usual extension cost — cheap to add operations, expensive to add a new type to the family. |
| Avoid hand-rolling Interpreter for anything beyond a tiny grammar | A real parser generator or embedded scripting engine scales far better than a hand-written expression tree once the grammar grows. |
