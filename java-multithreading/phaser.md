# Understanding the Phaser Pattern in Java

The **Phaser** pattern is a synchronization construct in Java (introduced in `java.util.concurrent`) that acts as a more flexible and powerful generalization of the `CyclicBarrier` and `CountDownLatch`. 

It allows a set of threads to wait for each other to reach a common execution barrier, but with a key advantage: **the number of registered parties (threads) can dynamically change** over time (parties can register and arrive/deregister dynamically).

---

## How a Phaser Works

A Phaser coordinates threads across multiple **phases** (generations). 
1. **Registration:** Threads register with the phaser to become "parties".
2. **Arrival:** Threads perform their task for the current phase and call `arriveAndAwaitAdvance()` or `arriveAndDeregister()`.
3. **Phase Advancement:** Once all registered parties arrive, the phase number increments, any optional on-advance action (`onAdvance()`) is executed, and waiting threads are released to start the next phase.

---

## Real-Life Application

**Application:** A Multi-Stage Distributed Simulation (e.g., a Game Server or Manufacturing Pipeline).

Imagine a game simulation engine running a multi-step turn-based combat round or a physics simulation with dependent steps:
1. **Phase 1: Input Processing** – All player threads read inputs.
2. **Phase 2: Physics Calculation** – All objects move based on inputs.
3. **Phase 3: Collision Detection & Rendering** – Frame is resolved and drawn.

New players can join mid-game (dynamic registration), and disconnected players can leave (deregistration) without breaking the synchronization of the ongoing rounds.

---

## Code Example

Here is a complete Java example demonstrating a 3-phase task where multiple worker threads synchronize at each phase using a `Phaser`:

```java
import java.util.concurrent.Phaser;

public class PhaserExample {

    public static void main(String[] args) {
        // Create a phaser with 1 registered party (the main thread) initially, 
        // or register workers dynamically. Here we start with 3 worker threads.
        Phaser phaser = new Phaser(1) { // 1 means 'main' thread is registered initially
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println("--- Phase " + phase + " completed. Parties: " + registeredParties + " ---\n");
                // Return true to terminate the phaser after phase 2 (total 3 phases: 0, 1, 2)
                return phase == 2 || registeredParties == 0;
            }
        };

        // Spawn 3 worker threads
        for (int i = 1; i <= 3; i++) {
            new Thread(new Worker("Worker-" + i, phaser)).start();
        }

        // The main thread also participates or just waits for completion
        // Let's have the main thread wait until the phaser is terminated
        while (!phaser.isTerminated()) {
            phaser.arriveAndAwaitAdvance();
        }

        System.out.println("All phases completed. Simulation finished.");
    }

    static class Worker implements Runnable {
        private final String name;
        private final Phaser phaser;

        Worker(String name, Phaser phaser) {
            this.name = name;
            this.phaser = phaser;
            phaser.register(); // Dynamically register this thread with the phaser
        }

        @Override
        public void run() {
            try {
                // Phase 0: Initialization
                System.out.println(name + " is initializing...");
                Thread.sleep(500);
                phaser.arriveAndAwaitAdvance(); // Wait for other workers to finish phase 0

                // Phase 1: Processing
                System.out.println(name + " is processing data...");
                Thread.sleep(500);
                phaser.arriveAndAwaitAdvance(); // Wait for other workers to finish phase 1

                // Phase 2: Cleanup
                System.out.println(name + " is cleaning up...");
                Thread.sleep(500);
                
                // Arrive and deregister so the phaser knows this thread is done permanently
                phaser.arriveAndDeregister(); 
                System.out.println(name + " has finished and exited.");

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.err.println(name + " was interrupted.");
            }
        }
    }
}
```

---

## Key Methods at a Glance

| Method | Description |
| :--- | :--- |
| `register()` | Adds a new unarrived party to the phaser. |
| `arrive()` | Arrives at the phaser without waiting for others to catch up. |
| `arriveAndAwaitAdvance()` | Arrives and blocks until all other registered parties arrive for the current phase. |
| `arriveAndDeregister()` | Arrives and removes the party from the phaser (decrements the registered party count). |
| `onAdvance(int phase, int registeredParties)` | Overridable hook executed whenever the phase changes; returns `true` to terminate. |