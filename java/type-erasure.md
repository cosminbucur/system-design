# Type Erasure in Java

**Type erasure** is the core mechanism Java uses to implement generics. Introduced in Java 5 to support generics while maintaining backward compatibility with older bytecode, type erasure ensures that no new classes are created for parameterized types; all generic type information is removed at compile-time.

---

## How Type Erasure Works

When you compile generic Java code, the compiler performs the following steps:

- **Replaces Type Parameters:** It replaces all type parameters in generic classes and methods with their bounded type, or with `Object` if no bound is specified. For example, `List<T>` becomes `List<Object>`, and `List<T extends Number>` becomes `List<Number>`.
- **Inserts Type Casts:** Because the generic types are replaced by `Object` (or bounds), the compiler automatically inserts explicit type casts where retrieved values are used, ensuring type safety without runtime overhead from the generic definitions themselves.
- **Generates Bridge Methods:** To preserve polymorphism in extended generic types, the compiler sometimes generates synthetic bridge methods.

### Example Transformation

**Before Compilation:**

```java
public class Box<T> {
    private T item;

    public void set(T item) { this.item = item; }
    public T get() { return item; }
}
```

**After Erasure (Bytecode Level):**

```java
public class Box {
    private Object item;

    public void set(Object item) { this.item = item; }
    public Object get() { return item; }
}
```

---

## Key Implications and Limitations

Because type information does not exist at runtime, type erasure introduces several important restrictions:

- **Cannot use primitives as type parameters:** You cannot write `List<int>`. You must use wrapper classes like `List<Integer>`, because primitives do not inherit from `Object`.
- **`instanceof` checks are restricted:** You cannot check against a parameterized type at runtime. For instance, `if (list instanceof List<String>)` is a compile-time error. You can only check raw types (`if (list instanceof List)`).
- **Cannot create generic arrays:** Instantiating arrays of generic types (e.g., `T[] array = new T[10];`) is illegal because arrays carry reified type information at runtime, which conflicts with erasure.
- **Overloading conflicts:** You cannot create overloaded methods whose parameters differ only by their generic type (e.g., `print(List<String> list)` and `print(List<Integer> list)` both erase to `print(List list)`, causing an erasure clash).

* Identical Bytecode: Parameterized types like `List<String>` and `List<Integer>` compile to the same raw List class, meaning instanceof checks and array creation of generic types are prohibited.
* Method Overloading Limitations: Methods cannot be overloaded solely based on different generic type parameters (e.g., process(`List<String>`) and process(`List<Integer>`) collide) because their signatures become identical after erasure.
* Workarounds: To retain type information at runtime, developers must pass `Class<T>` objects as parameters or utilize Java's Reflection API to inspect generic metadata.
