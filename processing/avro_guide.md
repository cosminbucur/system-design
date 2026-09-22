# Apache Avro: Quick Reference Guide

**Apache Avro** is a lightweight, language-agnostic binary serialization framework. It relies on **JSON-defined schemas** to enable fast, compact data serialization and structured schema evolution.

---

## Key Benefits of Avro

1. **Compact Payloads:** Data is stored in binary without field names/types in every record, greatly reducing payload size compared to JSON or XML.
2. **Schema Evolution:** Allows schemas to change over time (adding/deleting fields) without breaking older readers or writers.
3. **Language Agnostic:** Supports Java, Python, C++, C#, Go, and Node.js.

---

## 1. Defining an Avro Schema (`.avsc`)

Avro schemas are written in JSON format. Save this file as `user.avsc`:

```json
{
  "type": "record",
  "namespace": "com.example.avro",
  "name": "User",
  "doc": "Schema for user profiles",
  "fields": [
    { "name": "id", "type": "string" },
    { "name": "username", "type": "string" },
    { "name": "age", "type": ["null", "int"], "default": null },
    { "name": "email", "type": "string", "default": "" }
  ]
}
```

### Supported Data Types
* **Primitive Types:** `null`, `boolean`, `int`, `long`, `float`, `double`, `bytes`, `string`
* **Complex Types:**
  * `record`: Named fields (like a class or struct)
  * `enum`: Named set of values
  * `array`: List of items of a given type
  * `map`: Key-value pairs (keys are always strings)
  * `union`: Field that can hold multiple types (e.g., `["null", "int"]` for optional fields)

---

## 2. Using Avro in Java

### Maven Plugin Setup
Use the `avro-maven-plugin` to automatically generate Java classes from `.avsc` schema files during compilation:

```xml
<plugin>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro-maven-plugin</artifactId>
    <version>1.11.3</version>
    <executions>
        <execution>
            <phase>generate-sources</phase>
            <goals>
                <goal>schema</goal>
            </goals>
            <configuration>
                <sourceDirectory>${project.basedir}/src/main/resources/avro/</sourceDirectory>
                <outputDirectory>${project.build.directory}/generated-sources/avro</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Java Usage Example

```java
import com.example.avro.User;
import org.apache.avro.file.DataFileReader;
import org.apache.avro.file.DataFileWriter;
import org.apache.avro.io.*;
import org.apache.avro.specific.SpecificDatumReader;
import org.apache.avro.specific.SpecificDatumWriter;

import java.io.File;
import java.io.IOException;

public class AvroDemo {
    public static void main(String[] args) throws IOException {
        // 1. Build an Avro object using the generated Builder pattern
        User user = User.newBuilder()
                .setId("usr-1001")
                .setUsername("john_doe")
                .setAge(28)
                .setEmail("john@example.com")
                .build();

        File file = new File("users.avro");

        // 2. Serialize object to disk
        DatumWriter<User> userDatumWriter = new SpecificDatumWriter<>(User.class);
        try (DataFileWriter<User> dataFileWriter = new DataFileWriter<>(userDatumWriter)) {
            dataFileWriter.create(user.getSchema(), file);
            dataFileWriter.append(user);
        }

        // 3. Deserialize object from disk
        DatumReader<User> userDatumReader = new SpecificDatumReader<>(User.class);
        try (DataFileReader<User> dataFileReader = new DataFileReader<>(file, userDatumReader)) {
            while (dataFileReader.hasNext()) {
                User readUser = dataFileReader.next();
                System.out.println("Deserialized User: " + readUser.getUsername());
            }
        }
    }
}
```

---

## 3. Schema Evolution & Compatibility Rules

When modifying schemas over time, adhere to these compatibility strategies:

| Compatibility Mode | Rule | Reader Schema | Writer Schema |
| :--- | :--- | :--- | :--- |
| **BACKWARD** *(Default)* | Consumers using the **new** schema can read data written with the **old** schema. | New | Old |
| **FORWARD** | Consumers using the **old** schema can read data written with the **new** schema. | Old | New |
| **FULL** | Schema changes are both **BACKWARD** and **FORWARD** compatible. | New/Old | Old/New |

### Rules for Safe Schema Changes:
1. **Adding Fields:** Always provide a `default` value when adding a new field (enables **BACKWARD** compatibility).
2. **Deleting Fields:** Only delete fields that had a `default` value or were optional (enables **FORWARD** compatibility).
3. **Renaming Fields:** Avoid renaming directly; instead, use aliases:
   ```json
   { "name": "userEmail", "type": "string", "aliases": ["email"] }
   ```