# Apache Cassandra and Spring Boot Guide

Apache Cassandra is an open-source, distributed, wide-column NoSQL database designed to handle massive amounts of data across multiple commodity servers with **no single point of failure**. 

---

## Core Concepts of Cassandra

1. **Masterless Peer-to-Peer Architecture**: Unlike traditional relational databases or architectures with a master node, Cassandra uses a peer-to-peer ring topology where **every node is identical**. Any node can handle any read or write request, ensuring high availability and fault tolerance.
2. **Keyspace**: The top-level container for data organization (analogous to a database in relational systems). It defines settings like the replication strategy.
3. **Tables (formerly Column Families)**: Similar to tables in relational databases, containing rows and columns. However, rows can have dynamic sets of columns.
4. **Partition Key & Clustering Columns**: 
   * **Partition Key**: Determines which node in the cluster stores the data. Cassandra hashes the partition key to decide the token location.
   * **Clustering Columns**: Determines how data is physically sorted *inside* a specific partition on disk, allowing fast range queries.
5. **Tunable Consistency**: Cassandra lets you choose the level of consistency required for each read and write operation (e.g., `ONE`, `QUORUM`, `ALL`), trading off between consistency and availability depending on your application needs.

---

## How Cassandra Works (Under the Hood)

* **Write Path**: When a write request hits any node (acting as a *coordinator*), it is simultaneously written to an append-only on-disk **Commit Log** (for crash recovery) and an in-memory structure called a **Memtable**. Once both succeed, a success response is sent back. This makes writes extremely fast.
* **SSTables**: When Memtables reach a certain size, they are flushed to disk as immutable **SSTables** (Sorted String Tables). Background processes called *compactions* merge and clean up these SSTables over time.
* **Distributed Hashing & Replication**: Data is distributed evenly using consistent hashing across a token ring. A **Replication Factor (RF)** ensures that copies of data reside on multiple physical nodes for redundancy.

---

## Spring Boot + Cassandra Code Example

Here is how to set up and use Apache Cassandra in a Spring Boot application using **Spring Data Cassandra**.

### 1. Maven Dependencies (`pom.xml`)
```xml
<dependencies>
    <!-- Spring Boot Web Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Spring Boot Data Cassandra Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-cassandra</artifactId>
    </dependency>
    <!-- Lombok for Boilerplate Reduction -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### 2. Configuration (`application.properties`)
```properties
spring.cassandra.contact-points=localhost
spring.cassandra.port=9042
spring.cassandra.keyspace-name=bank_db
spring.cassandra.local-datacenter=datacenter1
spring.cassandra.schema-action=CREATE_IF_NOT_EXISTS
```

### 3. Domain Entity (`Customer.java`)
In Cassandra, annotations map your classes to keyspace tables, defining partition and clustering keys.
```java
package com.example.demo.model;

import org.springframework.data.cassandra.core.cql.PrimaryKeyType;
import org.springframework.data.cassandra.core.mapping.PrimaryKeyColumn;
import org.springframework.data.cassandra.core.mapping.Table;
import lombok.Data;
import java.util.UUID;

@Data
@Table("customers")
public class Customer {

    @PrimaryKeyColumn(name = "id", type = PrimaryKeyType.PARTITIONED)
    private UUID id;

    private String name;
    private String email;
    private String phone;
}
```

### 4. Repository (`CustomerRepository.java`)
```java
package com.example.demo.repository;

import com.example.demo.model.Customer;
import org.springframework.data.cassandra.repository.CassandraRepository;
import org.springframework.data.cassandra.repository.Query;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.UUID;

@Repository
public interface CustomerRepository extends CassandraRepository<Customer, UUID> {

    // Cassandra restricts non-primary key queries unless ALLOW FILTERING is specified
    @Query("SELECT * FROM customers WHERE name = ?0 ALLOW FILTERING")
    List<Customer> findByName(String name);
}
```

### 5. REST Controller (`CustomerController.java`)
```java
package com.example.demo.controller;

import com.example.demo.model.Customer;
import com.example.demo.repository.CustomerRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    @Autowired
    private CustomerRepository customerRepository;

    @PostMapping
    public Customer createCustomer(@RequestBody Customer customer) {
        customer.setId(UUID.randomUUID());
        return customerRepository.save(customer);
    }

    @GetMapping
    public List<Customer> getAllCustomers() {
        return customerRepository.findAll();
    }

    @GetMapping("/by-name/{name}")
    public List<Customer> getCustomerByName(@PathVariable String name) {
        return customerRepository.findByName(name);
    }

    @DeleteMapping("/{id}")
    public void deleteCustomer(@PathVariable UUID id) {
        customerRepository.deleteById(id);
    }
}
```