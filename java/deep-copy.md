**Deep copy** in Java is a technique that creates a completely independent duplicate of an object, including **recursively copying all nested objects** and mutable references. Unlike a shallow copy, which only duplicates the top-level object and shares references to internal objects, a deep copy ensures that the original and cloned objects share **no mutable state**.

Common implementation approaches include:

- **Override `clone()`**: Manually call `super.clone()` and then recursively clone mutable fields (e.g., `cloned.addr = (Address) addr.clone()`).
- **Copy Constructors**: Create new instances of nested objects within the constructor (e.g., `this.addr = new Address(other.addr)`).
- **Serialization**: Serialize the object to a byte stream and deserialize it back, which automatically handles deep copying of all `Serializable` nested objects.
- **External Libraries**: Use tools like **Jackson**, **Gson**, or **Apache Commons Lang** to serialize/deserialize or use reflection-based cloning.

In a deep copy, modifying a nested object in the clone (e.g., `clonedObject.getNested().setValue(5)`) **does not affect** the original object, providing full data isolation.
