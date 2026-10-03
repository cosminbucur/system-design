# Core Concepts of Big Data: Hadoop, Spark, and Cassandra

Understanding big data involves processing, storing, and analyzing massive, complex datasets that traditional relational database management systems (RDBMS) cannot handle due to volume, velocity, and variety. 

---

## 1. Apache Hadoop (The Storage and Batch Processing Engine)
Hadoop is the foundational framework designed to store and process huge datasets across clusters of commodity hardware using a distributed architecture.

* **HDFS (Hadoop Distributed File System):** Splits large files into large blocks (e.g., $128\text{MB}$) and distributes them across multiple nodes in a cluster, replicating data to ensure fault tolerance.
* **MapReduce:** A programming model for processing large data sets with a parallel, distributed algorithm on a cluster. It consists of a **Map** step (filtering and sorting data) and a **Reduce** step (summarizing the results).
* **YARN (Yet Another Resource Negotiator):** The cluster resource management layer that schedules jobs and manages system resources across the cluster.

---

## 2. Apache Spark (The Fast In-Memory Compute Engine)
Spark was created to overcome the speed limitations of Hadoop's MapReduce by performing data processing in memory.

* **RDDs (Resilient Distributed Datasets):** The fundamental data structure of Spark—a fault-tolerant collection of elements that can be operated on in parallel across the cluster.
* **In-Memory Processing:** By caching data in RAM rather than writing intermediate results back to disk, Spark can run iterative algorithms and machine learning tasks up to $100\times$ faster than traditional Hadoop MapReduce.
* **Unified Ecosystem:** Spark provides libraries for diverse workloads, including:
  * **Spark SQL** for structured data processing.
  * **Spark Streaming** for real-time data streams.
  * **MLlib** for distributed machine learning.
  * **GraphX** for graph analytics.

---

## 3. Apache Cassandra (The NoSQL Distributed Database)
Cassandra is a massively scalable, distributed NoSQL database designed to handle high-velocity, structured, and semi-structured data across multiple servers with **no single point of failure**.

* **Masterless Architecture:** Every node in a Cassandra cluster plays the same role, eliminating bottlenecks and ensuring continuous availability even if multiple nodes fail.
* **Linear Scalability:** You can easily add capacity by adding new nodes to the cluster without shutting down the system or rewriting applications.
* **Tunable Consistency:** Allows developers to choose how consistent they need their read and write operations to be on a per-query basis (e.g., balancing speed vs. absolute synchronization).

---

## How They Work Together
In a modern big data architecture, these three technologies often complement each other:

1. **Cassandra** acts as the high-speed operational database handling real-time, user-facing reads and writes at scale.
2. **Hadoop (HDFS)** serves as the long-term, cost-effective data lake for massive historical storage and archiving.
3. **Spark** acts as the analytical engine—connecting to both data sources (reading historical batches from Hadoop or real-time streams from Cassandra), processing them rapidly in memory, and writing the resulting insights back out.