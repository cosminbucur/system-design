"Big data" describes datasets too large or too fast-moving to process on a single machine — the defining shift is moving computation to where the data lives across many machines, rather than pulling all the data to one machine's compute. Hadoop was the technology that made this practical and mainstream, and even where its original components have since been superseded, the ideas it introduced (data locality, horizontal scale via commodity hardware, separating storage from compute) still underpin the tools that replaced it.

## 1. Why a Single Machine Stops Working

A dataset that fits on one disk and one CPU is just a database problem — indexing, partitioning, and the usual database internals already covered elsewhere apply. Big data tooling exists specifically for the point past that: petabyte-scale storage no single disk can hold, and computation (a full scan, a join, an aggregation) that would take days on one CPU but can complete in minutes if split across thousands of machines working in parallel. The core engineering problem this raises isn't "how do I write faster code" — it's "how do I coordinate thousands of machines that will individually fail on a regular basis, without losing data or getting a wrong answer."

## 2. HDFS — Hadoop Distributed File System

HDFS solves the storage half: a single logical filesystem that spans many physical machines, splitting each file into fixed-size blocks (128MB or 256MB by default) and replicating each block across multiple nodes (3x by default).

```
File: sales_2024.csv (600MB)
  → Block 1 (256MB) → replicated to Node A, Node D, Node F
  → Block 2 (256MB) → replicated to Node B, Node C, Node E
  → Block 3 (88MB)  → replicated to Node A, Node C, Node F
```

- **NameNode**: the single metadata coordinator — knows which blocks make up which file and which nodes hold which blocks. It doesn't store any actual file data itself.
- **DataNodes**: store the actual blocks and serve read/write requests directly to clients.
- **Replication** exists for two reasons at once: fault tolerance (a node dying doesn't lose data — another replica already has it) and read throughput (multiple nodes can serve the same block's reads in parallel).

This replication-for-fault-tolerance idea is the same principle behind database replication covered elsewhere — HDFS just applies it at the filesystem block level instead of the row/transaction level.

## 3. MapReduce — The Original Compute Model

MapReduce is the programming model Hadoop popularized for processing HDFS-stored data in parallel: every job is expressed as a `map` phase (transform/filter records independently, in parallel, on the node that already holds that data) followed by a `reduce` phase (aggregate the map outputs by key).

```java
// Classic word-count MapReduce job — the canonical example of the model
public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
    public void map(LongWritable key, Text value, Context context) {
        for (String word : value.toString().split("\\s+")) {
            context.write(new Text(word), new IntWritable(1)); // emit (word, 1) for every occurrence
        }
    }
}

public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    public void reduce(Text key, Iterable<IntWritable> values, Context context) {
        int sum = 0;
        for (IntWritable val : values) sum += val.get();
        context.write(key, new IntWritable(sum)); // emit (word, totalCount)
    }
}
```

The critical design idea is **data locality**: the framework schedules each `map` task to run on (or near) the node that already physically holds that block, rather than shipping terabytes of data across the network to a compute node. Moving a small amount of code to where a huge amount of data already sits is cheaper than the reverse — this principle outlived MapReduce itself and is still why Spark and other successors are also HDFS/data-locality-aware.

MapReduce's real limitation, and the reason it's largely been superseded: every job's intermediate results are written to disk between the map and reduce phases (and between chained jobs), so a multi-step pipeline pays repeated disk I/O at every stage — fine for a single large batch job, painful for anything iterative (like machine learning training, which re-reads the same dataset many times).

## 4. YARN — Separating Resource Management from Compute

YARN (Yet Another Resource Negotiator) split cluster resource management out of MapReduce into its own layer, so the same cluster could run MapReduce jobs, Spark jobs, or other frameworks side by side, all competing for the same pool of CPU/memory through one scheduler.

| Component | Role |
| --- | --- |
| ResourceManager | Cluster-wide arbiter — decides which application gets which resources |
| NodeManager | Runs on each machine — reports available resources, launches containers |
| ApplicationMaster | One per running job — negotiates resources from the ResourceManager and manages that job's own tasks |

This separation (YARN for "where do I get CPU/memory," a processing engine for "what computation actually runs") is what let the ecosystem evolve past MapReduce without discarding HDFS or the cluster-management layer underneath it.

## 5. Spark — The In-Memory Successor

Spark keeps HDFS (or cloud storage like S3) as the storage layer but replaces MapReduce's disk-bound execution model with in-memory computation across a chain of transformations, avoiding the repeated disk writes that made multi-step MapReduce pipelines slow.

```java
// Spark — same word count, but transformations chain in memory until an action forces execution
JavaRDD<String> lines = sparkContext.textFile("hdfs://cluster/sales_2024.csv");
JavaPairRDD<String, Integer> wordCounts = lines
    .flatMap(line -> Arrays.asList(line.split("\\s+")).iterator())
    .mapToPair(word -> new Tuple2<>(word, 1))
    .reduceByKey(Integer::sum);

wordCounts.saveAsTextFile("hdfs://cluster/output");
```

Spark's core abstraction, the RDD (Resilient Distributed Dataset), tracks the *lineage* of transformations that produced it rather than eagerly computing each step — this is what lets Spark recompute just the lost partition from a failed node (by replaying its lineage) instead of needing full replication of every intermediate result the way MapReduce implicitly relied on writing everything to replicated HDFS between stages. Spark also unified batch processing, the streaming model already covered in data streaming, SQL queries (Spark SQL), and machine learning (MLlib) under one engine — a major reason it displaced MapReduce as the default compute engine on top of Hadoop-style clusters.

## 6. Hive — SQL on Top of Distributed Storage

Hive lets analysts query HDFS-resident data using SQL, translating queries into MapReduce or Spark jobs underneath, so a data analyst doesn't need to write Java/Scala to get an aggregate answer out of a petabyte-scale dataset.

```sql
-- Looks like ordinary SQL, but runs as a distributed job across the cluster underneath
SELECT region, SUM(total) AS revenue
FROM sales
WHERE year = 2024
GROUP BY region
ORDER BY revenue DESC;
```

Hive's "schema-on-read" model — the data on disk is just files (often Parquet or ORC, columnar formats optimized for this kind of scan-and-aggregate query), and the schema is applied only when a query reads it — is the opposite of a relational database's "schema-on-write," and is what allows the same raw files to be queried under multiple different schemas or reprocessed as requirements change, without an upfront migration.

## 7. The Broader Ecosystem, Briefly

| Tool | Role |
| --- | --- |
| HBase | A wide-column NoSQL database built on top of HDFS — random real-time read/write access to huge datasets, versus HDFS's own batch-oriented, append-mostly access pattern |
| ZooKeeper | Distributed coordination service (leader election, distributed locks, configuration) many of these systems (HBase, Kafka historically) depend on internally |
| Kafka | Not part of Hadoop originally, but the standard way data streams into a big data pipeline for either Spark Streaming or batch ingestion into HDFS/S3 — already covered in data streaming |

## 8. Where This Sits Today

Most new systems don't run on-premises HDFS clusters anymore — cloud object storage (S3, GCS, Azure Blob) has largely replaced HDFS as the storage layer, and managed Spark (Databricks, EMR, Dataproc) has replaced hand-run Hadoop clusters. What survived from Hadoop isn't the specific software — it's the architectural pattern: separate cheap, durable, replicated storage from an elastic compute layer that reads from it, and move computation to the data rather than the reverse. Recognizing that pattern matters more today than knowing HDFS's internals, since it's the same shape underneath a modern cloud data lake or lakehouse architecture.

## 9. Best Practices

| Practice | Recommendation |
| --- | --- |
| Think in terms of data locality, not just parallelism | Moving computation to where data already sits (Hadoop's core idea) still explains why cloud data lake architectures are laid out the way they are. |
| Prefer Spark over raw MapReduce for new work | In-memory, lineage-based recomputation avoids MapReduce's repeated-disk-I/O cost for multi-step or iterative pipelines. |
| Use columnar formats (Parquet/ORC) for analytical big-data workloads | Schema-on-read query engines scan far less data per query than row-oriented formats when aggregating over a few columns of a wide table. |
| Don't reach for Hadoop-scale tooling below actual big-data scale | If the dataset fits comfortably on one well-indexed database, distributed cluster tooling adds operational cost with no benefit. |
| Separate storage from compute deliberately | Whether HDFS+YARN or S3+managed Spark, decoupling the two lets each scale independently and lets multiple engines share the same data. |
| Recognize replication's dual purpose | Block/data replication buys both fault tolerance and parallel read throughput — the same tradeoff seen in database replication, applied at filesystem scale. |
