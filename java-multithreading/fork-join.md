# Java Fork/Join Pattern

The **Fork/Join pattern** (introduced in Java 7 via `java.util.concurrent`) is a framework designed to accelerate **parallel, CPU-bound tasks** using a classic divide-and-conquer strategy.

Instead of assigning one massive job to a single thread or evenly dividing work upfront, it recursively breaks down a large task into smaller subtasks (**Fork**), runs them across multiple CPU cores, and combines the sub-results back into a single final answer (**Join**).

---

## 🏢 Real-Life Analogy: The Moving Company

Imagine you run a moving company and need to pack **1,000 cardboard boxes**.

- **Sequential Approach (No Concurrency):** You pack all 1,000 boxes yourself, one by one. It takes 20 hours.

- **Standard ThreadPool Executor:** You hire 4 workers and give a strict assignment of 250 boxes directly to each worker. If Worker 1 gets stuck packing fragile glassware (slow task) while Worker 2 finishes their 250 simple clothes boxes in 2 hours, Worker 2 sits idle while Worker 1 is overwhelmed.

- **Fork/Join Pattern:**
  1. **Fork:** The supervisor sees 1,000 boxes and splits them down into manageable 50-box batches.
  2. **Work-Stealing:** Each worker manages their own task queue. If Worker 2 finishes early, they **steal** unstarted tasks from the back of Worker 1's queue.
  3. **Join:** Individual batch tallies are added together hierarchically to yield the final result.

---

## 🛠️ Core Concepts & Architecture

| Concept                     | Description                                                                                                         |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **`fork()`**                | Asynchronously submits a subtask to be executed in the `ForkJoinPool`.                                              |
| **`join()`**                | Waits for the completion of a subtask and retrieves its result.                                                     |
| **Threshold (Base Case)**   | A defined limit determining when a subtask is small enough to run sequentially.                                     |
| **Work-Stealing Algorithm** | Idle worker threads actively steal pending tasks from the back of busy threads' queues, maximizing CPU utilization. |

---

## 💻 Concrete Real-World Application: Recursive Disk Space Analyzer

A primary real-world application for Fork/Join is **Recursive File System Processing**—such as computing total disk usage across nested directory structures (similar to tools like _WinDirStat_ or _TreeSize_).

Because directory trees are inherently unbalanced (some subfolders contain 2 files, while others hold 100,000 nested files), standard thread pools result in idle workers. The **work-stealing algorithm** solves this by rebalancing subfolder tasks dynamically.

```java
import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class DirectorySizeCalculator extends RecursiveTask<Long> {
    private final File directory;

    public DirectorySizeCalculator(File directory) {
        this.directory = directory;
    }

    @Override
    protected Long compute() {
        long totalSize = 0;
        File[] files = directory.listFiles();

        if (files == null) {
            return 0L; // Directory empty or access denied
        }

        List<DirectorySizeCalculator> subTasks = new ArrayList<>();

        for (File file : files) {
            if (file.isDirectory()) {
                // FORK: Create a subtask for every nested directory
                DirectorySizeCalculator subTask = new DirectorySizeCalculator(file);
                subTask.fork();
                subTasks.add(subTask);
            } else {
                // BASE CASE: Directly accumulate file sizes in current folder
                totalSize += file.length();
            }
        }

        // JOIN: Wait for all subdirectories to finish and sum their sizes
        for (DirectorySizeCalculator subTask : subTasks) {
            totalSize += subTask.join();
        }

        return totalSize;
    }

    public static void main(String[] args) {
        File rootDir = new File("/Users/username/Documents");

        // ForkJoinPool.commonPool() defaults to CPU core count
        try (ForkJoinPool pool = ForkJoinPool.commonPool()) {
            long startTime = System.currentTimeMillis();

            long totalBytes = pool.invoke(new DirectorySizeCalculator(rootDir));

            long endTime = System.currentTimeMillis();

            System.out.printf("Total Size: %.2f MB%n", totalBytes / (1024.0 * 1024.0));
            System.out.printf("Execution Time: %d ms%n", (endTime - startTime));
        }
    }
}
```

### How Work-Stealing Handles Unbalanced Directories

```text
[Thread 1: /Users]
  ├─ Forks /Users/Alice task
  ├─ Forks /Users/Bob task
  └─ Processing local files...

[Thread 2 (Idle Core)] ---> STEALS /Users/Bob from Thread 1's queue
  ├─ Scans /Users/Bob
  ├─ Forks /Users/Bob/Videos (100 GB)
  └─ Forks /Users/Bob/Notes  (2 KB)

[Thread 3 (Finishes /Users/Bob/Notes in 1ms)]
  └---> STEALS half of /Users/Bob/Videos subfolder tasks from Thread 2
```

---

## 🌐 Other Production Use Cases

1. **`Arrays.parallelSort()`**: Switches internally to a Fork/Join merge-sort implementation once an array exceeds $8,192$ elements.
2. **Parallel Streams**: Operations like `list.parallelStream().filter(...).collect(...)` execute tasks across the common `ForkJoinPool`.
3. **Image & Matrix Processing**: Dividing $4K$ image pixel grids into quadrants to apply filters (e.g., Gaussian blur) concurrently before stitching them back together.

---

## 💡 Best Practices and When NOT to Use

### Best Practices

- **Optimize Fork/Compute**: Instead of calling `leftTask.fork()` and `rightTask.fork()`, fork one task (`leftTask.fork()`) and directly call `.compute()` on the second task in the current thread to reduce context switching.
- **Define Proper Base Cases**: If tasks are split down to individual single operations, task management overhead will outweigh parallel computing gains.

### When NOT to Use

- **Blocking I/O Operations**: Avoid network calls, REST APIs, or database queries inside Fork/Join tasks. Blocked worker threads reduce available parallelism across the pool. Use standard thread pools or **Java Virtual Threads** instead.
