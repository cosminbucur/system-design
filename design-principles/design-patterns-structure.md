Structural patterns are reusable solutions for composing classes and objects into larger structures while keeping them loosely coupled — how to make incompatible interfaces work together, add behavior without subclass explosion, or represent a whole made of parts, without those pieces becoming tightly bound to each other's concrete implementations.

## 1. Adapter

Converts one interface into another the client expects — useful when integrating a third-party or legacy API that doesn't match your interfaces.

```java
public interface NotificationSender {
    void send(String recipient, String message);
}

public class LegacySmsGatewayAdapter implements NotificationSender {
    private final LegacySmsGateway legacyGateway; // third-party class you can't change

    @Override
    public void send(String recipient, String message) {
        legacyGateway.dispatchMessage(recipient, message, LegacySmsGateway.PRIORITY_NORMAL);
    }
}
```

## 2. Bridge

Separates an abstraction from its implementation so the two can vary and evolve independently — where Adapter retrofits compatibility onto something already built, Bridge is designed in upfront so a high-level abstraction never depends on a specific low-level implementation in the first place.

```java
// Implementation hierarchy — how a notification is actually delivered
public interface DeliveryChannel {
    void deliver(String message);
}
public class EmailChannel implements DeliveryChannel {
    public void deliver(String message) { /* SMTP send */ }
}
public class SmsChannel implements DeliveryChannel {
    public void deliver(String message) { /* SMS gateway send */ }
}

// Abstraction hierarchy — what kind of notification it is, delegating delivery to a DeliveryChannel
public abstract class Notification {
    protected final DeliveryChannel channel; // the "bridge" — composed, not inherited
    protected Notification(DeliveryChannel channel) { this.channel = channel; }
    public abstract void send();
}
public class UrgentNotification extends Notification {
    public UrgentNotification(DeliveryChannel channel) { super(channel); }
    public void send() { channel.deliver("[URGENT] " + buildMessage()); }
    private String buildMessage() { return "..."; }
}
```

Without Bridge, adding a new notification urgency *and* a new delivery channel independently would require a class for every combination (`UrgentEmailNotification`, `UrgentSmsNotification`, `RoutineEmailNotification`, ...) — Bridge composes the two hierarchies instead of multiplying them.

## 3. Composite

Treats individual objects and groups of objects through the same interface, so client code can work with a single item or a whole tree of items without knowing which it has.

```java
public interface FileSystemNode {
    long size();
}

public class File implements FileSystemNode {
    private final long sizeBytes;
    public File(long sizeBytes) { this.sizeBytes = sizeBytes; }
    public long size() { return sizeBytes; }
}

public class Directory implements FileSystemNode {
    private final List<FileSystemNode> children = new ArrayList<>();
    public void add(FileSystemNode node) { children.add(node); }
    public long size() {
        return children.stream().mapToLong(FileSystemNode::size).sum(); // recurses into sub-directories transparently
    }
}
```

Calling `.size()` on a single `File` or on a deeply nested `Directory` looks identical to the caller — the recursive structure (a directory containing directories) is exactly what makes this pattern a natural fit for any tree-shaped domain (UI component trees, org charts, nested categories).

## 4. Decorator

Adds behavior to an object dynamically, without modifying its class or affecting other instances. Java I/O (`BufferedReader(new FileReader(...))`) is the canonical built-in example.

```java
public interface Coffee {
    BigDecimal cost();
}

public class MilkDecorator implements Coffee {
    private final Coffee delegate;
    public MilkDecorator(Coffee delegate) { this.delegate = delegate; }

    @Override
    public BigDecimal cost() { return delegate.cost().add(new BigDecimal("0.50")); }
}

Coffee order = new MilkDecorator(new SimpleCoffee()); // stack decorators to compose behavior
```

## 5. Facade

Provides a simplified, unified interface over a complex subsystem — hides internal complexity from callers.

```java
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;

    public void placeOrder(Order order) {
        inventory.reserve(order);
        payment.charge(order);
        shipping.schedule(order);
    }
}
```

## 6. Flyweight

Shares a single instance of expensive, immutable state across many logical objects instead of duplicating it per instance — used when a huge number of objects would otherwise each carry a redundant copy of the same data.

```java
// Immutable, shareable "intrinsic" state — heavy data reused across many characters
public class CharacterStyle {
    private final String fontFamily;
    private final int size;
    private final String color;
    // ... constructor, equals/hashCode
}

public class CharacterStyleFactory {
    private final Map<String, CharacterStyle> cache = new ConcurrentHashMap<>();

    public CharacterStyle getStyle(String fontFamily, int size, String color) {
        String key = fontFamily + size + color;
        return cache.computeIfAbsent(key, k -> new CharacterStyle(fontFamily, size, color)); // reuse if it already exists
    }
}

// Each rendered character only stores its position (the "extrinsic" state) plus a shared style reference
public class RenderedCharacter {
    private final char value;
    private final int x, y;
    private final CharacterStyle style; // shared, not duplicated, across every character using this style
}
```

Java's own `String.intern()` and the boxed-integer cache (`Integer.valueOf(-128..127)`) are built-in examples of exactly this idea — reuse an immutable instance instead of allocating a new one for identical data.

## 7. Proxy

Provides a stand-in for another object that controls access to it — adding lazy loading, access control, caching, or remote-call plumbing, transparently to the caller.

```java
public interface Image {
    void display();
}

public class RealImage implements Image {
    private final String filename;
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk(filename); // expensive — only want this to happen if actually needed
    }
    public void display() { /* render already-loaded pixel data */ }
    private void loadFromDisk(String filename) { /* ... */ }
}

public class LazyImageProxy implements Image {
    private final String filename;
    private RealImage real; // not created until display() is actually called

    public LazyImageProxy(String filename) { this.filename = filename; }

    public void display() {
        if (real == null) real = new RealImage(filename); // load on first real use, not on construction
        real.display();
    }
}
```

Spring's own AOP proxies (what actually backs `@Transactional`, `@Cacheable`, `@Async`) are a real-world Proxy pattern — the caller invokes what looks like the real bean, but a generated proxy intercepts the call first to add cross-cutting behavior.

## 8. Pattern Selection Cheat Sheet

| Problem | Reach for |
| --- | --- |
| Two incompatible interfaces need to work together | Adapter |
| An abstraction and its implementation need to vary independently, without a class explosion | Bridge |
| Individual items and groups of items should be treated identically | Composite |
| Want to add behavior without subclassing every combination | Decorator |
| Simplify a complex subsystem for callers | Facade |
| Many objects share large amounts of identical, immutable data | Flyweight |
| Need to control/defer/intercept access to an object (lazy load, cache, remote call, access check) | Proxy |

## 9. Best Practices

| Practice | Recommendation |
| --- | --- |
| Don't force a pattern | Structural patterns solve specific recurring composition problems — applying one where the problem doesn't exist adds indirection for no benefit. |
| Prefer composition over inheritance | Decorator/Bridge/Facade all favor composing small objects over deep class hierarchies — easier to test and extend. |
| Recognize patterns already in the JDK/Spring | `InputStream` wrapping (Decorator), AOP proxies backing `@Transactional`/`@Cacheable` (Proxy), `String.intern()` (Flyweight) — you're likely already using these. |
| Reach for Bridge specifically when two independent dimensions would otherwise multiply into a class per combination | If there's only one varying dimension, a simpler pattern (Strategy, plain inheritance) is usually enough. |
| Name things by pattern when it helps communication | Calling a class `PaymentGatewayAdapter` or `OrderFacade` signals intent to teammates faster than a generic name. |
