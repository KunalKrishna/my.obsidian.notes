
## Chapter 1 — "The Wild West" — Java 1.0 (1996)

Java was born with multithreading baked right in from day one. That was actually revolutionary — most languages at the time had no built-in threading model. Java said "every program can do multiple things at once!" and gave developers the **Thread** class and the **synchronized** keyword.

The idea was simple:
- You want to do something concurrently? Extend **Thread** or implement **Runnable**.
- You want to protect shared data? Use **synchronized**.
- You want threads to talk to each other? Use **wait() / notify() / notifyAll()** on the Object monitor.

---

### What a typical program looked like in Java 1.0

**Scenario:** A bank wants to process 5 customer transactions simultaneously.

```java
// Approach 1 — Extend Thread directly
public class TransactionThread extends Thread {
    private String customerName;
    private double amount;

    public TransactionThread(String customerName, double amount) {
        this.customerName = customerName;
        this.amount = amount;
    }

    @Override
    public void run() {
        System.out.println("Processing transaction for: " + customerName);
        // simulate processing
        try { Thread.sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println("Done: " + customerName);
    }

    public static void main(String[] args) {
        // Create a new Thread for EVERY customer — the Wild West way
        new TransactionThread("Alice", 500.0).start();
        new TransactionThread("Bob",   300.0).start();
        new TransactionThread("Carol", 700.0).start();
        new TransactionThread("Dave",  200.0).start();
        new TransactionThread("Eve",   900.0).start();
    }
}
```

**Approach 2 — Implement Runnable (preferred even then — separation of task from thread)**

```java
public class TransactionTask implements Runnable {
    private String customerName;

    public TransactionTask(String customerName) {
        this.customerName = customerName;
    }

    @Override
    public void run() {
        System.out.println("Processing: " + customerName + 
                           " on " + Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        Thread t1 = new Thread(new TransactionTask("Alice"));
        Thread t2 = new Thread(new TransactionTask("Bob"));
        t1.start();
        t2.start();
        // Who waits for who? You had to manually join!
        try {
            t1.join(); // main thread waits for t1 to finish
            t2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("All transactions done.");
    }
}
```

**Protecting shared data — synchronized**

```java
public class BankAccount {
    private double balance = 1000.0;

    // synchronized = only one thread can enter at a time
    // uses the intrinsic lock (monitor) of the BankAccount object
    public synchronized void deposit(double amount) {
        balance += amount;
    }

    public synchronized void withdraw(double amount) {
        if (balance >= amount) {
            balance -= amount;
        }
    }

    public synchronized double getBalance() {
        return balance;
    }
}
```

**Thread coordination — wait/notify (producer-consumer)**

```java
public class SimpleQueue {
    private String item = null;

    public synchronized void produce(String value) throws InterruptedException {
        while (item != null) {
            wait(); // release lock and wait until consumer takes the item
        }
        item = value;
        System.out.println("Produced: " + value);
        notify(); // wake up the consumer
    }

    public synchronized String consume() throws InterruptedException {
        while (item == null) {
            wait(); // release lock and wait until producer puts an item
        }
        String value = item;
        item = null;
        System.out.println("Consumed: " + value);
        notify(); // wake up the producer
        return value;
    }
}
```

---

### The problems that showed up very quickly

```
1. "new Thread for every task" = extremely expensive
   - Creating a Thread maps to an OS thread
   - Each OS thread = ~1MB stack memory
   - 1000 customers = 1000 threads = ~1GB RAM consumed just for stacks
   - Context switching overhead between hundreds of threads crushed performance

2. No way to get a result back from a thread
   - Runnable.run() returns void — nothing!
   - You had to use shared state (fields, arrays) with synchronization hacks to get results out
   - Error-prone, messy, hard to read

3. No way to cancel or time-out a running task cleanly

4. Exception handling was a nightmare
   - run() can't throw checked exceptions
   - Exceptions inside run() silently killed the thread
   - You had no way to know the thread failed unless you manually tracked it

5. wait/notify was extremely fragile
   - Easy to call notify() before wait() — now the waiting thread waits forever
   - Lost signals (race between checking condition and calling wait)
   - Spurious wakeups (a thread wakes up even without notify — so you MUST use while not if)
   - You had to reason about monitor ownership constantly

6. Deadlocks were easy to write, hard to detect
   - Thread A holds lock on Object1, wants Object2
   - Thread B holds lock on Object2, wants Object1
   - Both wait forever — program hangs silently

7. No thread-pool concept — no resource control whatsoever
```

---

### Java 1.3 — A small band-aid: Timer and TimerTask (1999)

Java 1.3 introduced **java.util.Timer** and **java.util.TimerTask** for scheduling.

```java
import java.util.Timer;
import java.util.TimerTask;

public class ScheduledReport {
    public static void main(String[] args) {
        Timer timer = new Timer();

        TimerTask task = new TimerTask() {
            @Override
            public void run() {
                System.out.println("Generating daily report at: " + new java.util.Date());
            }
        };

        // run after 1 second delay, then every 24 hours
        timer.scheduleAtFixedRate(task, 1000, 24 * 60 * 60 * 1000);
    }
}
```

**But Timer had critical flaws:**
```
1. Timer uses a SINGLE background thread for ALL tasks
   - If one task takes too long, all other scheduled tasks are delayed

2. If a task throws an unchecked exception, the Timer thread DIES
   - All future tasks are silently cancelled — no warning, no restart

3. Timer is sensitive to system clock changes (wall-clock based)

4. No thread-pool — can't do concurrent task execution
```

---

## Chapter 2 — "The Great Leap Forward" — Java 5 (2004) — JSR-166

By 2004 it was clear: raw threads were too low-level and too dangerous for everyday use. **Doug Lea** (a concurrency expert at SUNY Oswego) had written a widely-used library called **util.concurrent**. The Java community said "this is so good it needs to be in the standard library" — and so **JSR-166** was born, adding the **java.util.concurrent** package in Java 5.

This was the single biggest leap in Java concurrency history.

The philosophy shifted from:
> "You manage threads manually"

to:

> "You submit tasks. The framework manages threads."

---

### The ExecutorService and Thread Pools arrive

```java
import java.util.concurrent.*;

public class BankTransactionV2 {
    public static void main(String[] args) throws InterruptedException {

        // Instead of creating 1 thread per task,
        // create a POOL of 3 threads that REUSE themselves
        ExecutorService pool = Executors.newFixedThreadPool(3);

        // Submit 10 tasks — pool queues them and processes with 3 threads
        for (int i = 1; i <= 10; i++) {
            final int taskId = i;
            pool.execute(() -> {
                System.out.println("Processing transaction " + taskId +
                        " on thread: " + Thread.currentThread().getName());
                try { Thread.sleep(500); } catch (InterruptedException e) {}
            });
        }

        // Graceful shutdown — let submitted tasks finish, accept no new ones
        pool.shutdown();
        pool.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("All transactions complete.");
    }
}
```

---

### Callable and Future — finally get results back from threads!

```java
import java.util.concurrent.*;

public class TaxCalculator {
    public static void main(String[] args) throws Exception {

        ExecutorService pool = Executors.newFixedThreadPool(4);

        // Callable = like Runnable but RETURNS a value and CAN throw checked exceptions
        Callable<Double> task1 = () -> {
            Thread.sleep(1000); // simulate computation
            return 500.0 * 0.18; // tax for customer 1
        };

        Callable<Double> task2 = () -> {
            Thread.sleep(800);
            return 300.0 * 0.18;
        };

        // Future = a "promise" — the result will be there eventually
        Future<Double> future1 = pool.submit(task1);
        Future<Double> future2 = pool.submit(task2);

        // Do other work here while tasks run in background...
        System.out.println("Tasks submitted, doing other work...");

        // Now retrieve results — get() BLOCKS until result is ready
        Double tax1 = future1.get();   // blocks if not done yet
        Double tax2 = future2.get(2, TimeUnit.SECONDS); // blocks max 2s, then TimeoutException

        System.out.println("Tax 1: " + tax1);
        System.out.println("Tax 2: " + tax2);

        pool.shutdown();
    }
}
```

---

### Explicit Locks — ReentrantLock and Condition (replacing synchronized + wait/notify)

```java
import java.util.concurrent.locks.*;

public class BankAccountV2 {
    private double balance = 1000.0;
    
    // Explicit lock — more powerful than synchronized
    private final Lock lock = new ReentrantLock();
    
    // Condition variables — more powerful than wait/notify
    private final Condition sufficientFunds = lock.newCondition();

    public void deposit(double amount) {
        lock.lock();     // must always pair with unlock in finally
        try {
            balance += amount;
            System.out.println("Deposited: " + amount + " | Balance: " + balance);
            sufficientFunds.signalAll(); // notify waiting withdrawals
        } finally {
            lock.unlock(); // ALWAYS in finally — never forget this!
        }
    }

    public void withdraw(double amount) throws InterruptedException {
        lock.lock();
        try {
            // while loop handles spurious wakeups (same as wait/notify)
            while (balance < amount) {
                System.out.println("Insufficient funds. Waiting...");
                sufficientFunds.await(); // releases lock and waits
            }
            balance -= amount;
            System.out.println("Withdrew: " + amount + " | Balance: " + balance);
        } finally {
            lock.unlock();
        }
    }
}
```

**Why ReentrantLock over synchronized?**
```
1. tryLock() — try to acquire lock without blocking, return false if unavailable
2. tryLock(timeout) — try for a time period, give up if can't acquire (avoids deadlock)
3. lockInterruptibly() — can be interrupted while waiting (synchronized blocks forever)
4. Multiple Conditions — one lock can have many condition variables
   (synchronized only has one wait-set per object)
5. Fairness option — new ReentrantLock(true) = longest-waiting thread gets lock first
6. lock.getHoldCount(), isHeldByCurrentThread() — diagnostic info
```

---

### Atomic variables — lock-free, blazing fast for simple counters

```java
import java.util.concurrent.atomic.*;

public class RequestCounter {
    // BAD — NOT thread-safe without synchronization
    // private int counter = 0;
    
    // GOOD — AtomicInteger uses CPU-level CAS (Compare-And-Swap) instruction
    //         No locks needed — much faster under high concurrency
    private AtomicInteger counter = new AtomicInteger(0);

    public void requestReceived() {
        counter.incrementAndGet(); // atomic: read + increment + write as ONE operation
    }

    public int getCount() {
        return counter.get();
    }

    // CAS example — compareAndSet(expected, newValue)
    // Only updates if current value == expected (no lock needed)
    public boolean resetIfReachedThreshold(int threshold) {
        return counter.compareAndSet(threshold, 0);
        // "If counter is currently 'threshold', set it to 0, return true
        //  Otherwise do nothing, return false"
    }
}
```

---

### Concurrent Collections — thread-safe without locking the whole collection

```java
import java.util.concurrent.*;

public class UserSessionStore {
    // Old way: Collections.synchronizedMap(new HashMap<>())
    //          — locks the ENTIRE map for every get/put = huge bottleneck
    
    // New way: ConcurrentHashMap
    //          — locks only a SEGMENT (stripe) of the map (Java 7) 
    //          — or uses CAS with no locking at all (Java 8+)
    //          — multiple threads can read/write simultaneously to different segments
    private ConcurrentHashMap<String, String> sessions = new ConcurrentHashMap<>();

    public void login(String userId, String sessionToken) {
        // putIfAbsent = atomic "put only if key doesn't exist"
        sessions.putIfAbsent(userId, sessionToken);
    }

    public String getSession(String userId) {
        return sessions.get(userId); // safe without lock
    }
}

public class TaskQueue {
    // BlockingQueue — thread-safe queue with blocking put/take
    // Perfect for producer-consumer pattern (replaces the fragile wait/notify dance)
    private BlockingQueue<String> queue = new LinkedBlockingQueue<>(100); // capacity 100

    // Producer thread
    public void produce(String task) throws InterruptedException {
        queue.put(task);    // BLOCKS if queue is full — no manual wait/notify needed!
        System.out.println("Produced: " + task);
    }

    // Consumer thread
    public String consume() throws InterruptedException {
        String task = queue.take(); // BLOCKS if queue is empty — no manual wait/notify needed!
        System.out.println("Consumed: " + task);
        return task;
    }
}
```

---

### Synchronizers — coordinating groups of threads elegantly

```java
import java.util.concurrent.*;

// CountDownLatch — wait for N things to finish before proceeding
public class ServiceStartup {
    public static void main(String[] args) throws InterruptedException {
        int serviceCount = 3;
        CountDownLatch latch = new CountDownLatch(serviceCount);

        // 3 services starting up in parallel
        for (String service : new String[]{"Database", "Cache", "MessageBroker"}) {
            new Thread(() -> {
                System.out.println(service + " starting...");
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
                System.out.println(service + " ready!");
                latch.countDown(); // decrement counter by 1
            }).start();
        }

        System.out.println("Waiting for all services...");
        latch.await(); // BLOCKS main thread until counter reaches 0
        System.out.println("All services ready! Starting application.");
    }
}

// CyclicBarrier — N threads all wait at a checkpoint before any proceeds
public class DataProcessing {
    public static void main(String[] args) {
        int workers = 4;
        CyclicBarrier barrier = new CyclicBarrier(workers,
            () -> System.out.println("All workers at checkpoint! Moving to next phase."));

        for (int i = 0; i < workers; i++) {
            final int workerId = i;
            new Thread(() -> {
                System.out.println("Worker " + workerId + " processing phase 1...");
                try {
                    Thread.sleep((long)(Math.random() * 2000));
                    System.out.println("Worker " + workerId + " reached barrier");
                    barrier.await(); // wait for ALL workers to reach here
                    System.out.println("Worker " + workerId + " starting phase 2");
                } catch (Exception e) {}
            }).start();
        }
    }
}

// Semaphore — limit concurrent access to N permits (like a bouncer)
public class DatabaseConnectionPool {
    // Only 5 connections allowed simultaneously
    private final Semaphore semaphore = new Semaphore(5);

    public void query(String sql) throws InterruptedException {
        semaphore.acquire(); // take a permit (blocks if 0 permits left)
        try {
            System.out.println(Thread.currentThread().getName() + " executing: " + sql);
            Thread.sleep(500); // simulate DB query
        } finally {
            semaphore.release(); // always return the permit
        }
    }
}
```

---

### The problems that remained after Java 5

```
1. ThreadPoolExecutor was powerful but complex to configure correctly
   - Easy to misconfigure (wrong queue size, wrong pool size, wrong rejection policy)
   - Too many threads = wasteful; too few = slow

2. Future.get() was still BLOCKING
   - Waiting for a future result blocks the calling thread
   - Chaining multiple async operations was messy ("callback hell" via manual code)
   - Example: "do A, then when A completes do B, then when B completes do C"
     required complex manual Future chaining

3. ForkJoin was missing (divide-and-conquer parallelism needed a model)
   - Breaking a big problem into recursive sub-tasks was not elegantly supported

4. Parallel computation on large datasets (lists/arrays) was verbose
```

---

## Chapter 3 — "Divide and Conquer" — Java 7 (2011)

Java 7 introduced **ForkJoinPool** — a specialized pool for **divide-and-conquer** tasks where a big task recursively splits into smaller tasks.

The secret sauce was **work stealing**: idle threads steal tasks from busy threads' queues — much better CPU utilization.

```java
import java.util.concurrent.*;

// Classic example: sum a large array by splitting it recursively
public class ParallelArraySum extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1000; // split if more than 1000 elements
    private final long[] array;
    private final int start, end;

    public ParallelArraySum(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;

        // Base case — small enough, compute directly
        if (length <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }

        // Recursive case — split in half
        int mid = start + length / 2;
        ParallelArraySum leftTask  = new ParallelArraySum(array, start, mid);
        ParallelArraySum rightTask = new ParallelArraySum(array, mid,   end);

        leftTask.fork();            // submit left task to pool asynchronously
        Long rightResult = rightTask.compute();  // compute right task directly
        Long leftResult  = leftTask.join();      // wait for left result

        return leftResult + rightResult;
    }

    public static void main(String[] args) throws Exception {
        long[] bigArray = new long[10_000_000];
        for (int i = 0; i < bigArray.length; i++) bigArray[i] = i + 1;

        ForkJoinPool pool = new ForkJoinPool(); // uses CPU core count by default
        Long total = pool.invoke(new ParallelArraySum(bigArray, 0, bigArray.length));
        System.out.println("Total sum: " + total);
    }
}
```

---

### The problems that remained after Java 7

```
1. ForkJoinPool was great for CPU-bound recursive tasks but overkill for simpler cases

2. Future.get() STILL blocks
   - You can't say "when this finishes, do that" without blocking
   - Composing multiple async operations was painful:
     Future<A> futureA = pool.submit(taskA);
     A a = futureA.get();          // BLOCKS thread here
     Future<B> futureB = pool.submit(() -> doB(a));
     B b = futureB.get();          // BLOCKS thread here again
     // The thread is just sitting there blocked — wasting resources
   
3. No natural way to combine Futures:
   "Run A and B in parallel, when BOTH complete do C" = messy with raw Futures
   "Run A and B in parallel, when EITHER completes do C" = even messier

4. Exception handling across async chain was ugly
```

---

## Chapter 4 — "Async Pipelines" — Java 8 (2014)

Java 8 brought two game-changers:
1. **CompletableFuture** — fully async, composable pipeline-style programming
2. **Parallel Streams** — data-parallel processing on collections using ForkJoinPool under the hood
3. **Lambdas** — made all concurrency code massively less verbose

---

### CompletableFuture — async pipelines without blocking

```java
import java.util.concurrent.*;

public class OrderProcessingV2 {
    static ExecutorService pool = Executors.newFixedThreadPool(4);

    static CompletableFuture<String> fetchUser(int userId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(500);
            return "User_" + userId;
        }, pool);
    }

    static CompletableFuture<String> fetchOrder(String user) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(300);
            return "Order_for_" + user;
        }, pool);
    }

    static CompletableFuture<Double> calculateTotal(String order) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(200);
            return 99.99;
        }, pool);
    }

    public static void main(String[] args) throws Exception {

        // --- CHAINING (thenCompose = flatMap for futures) ---
        // "fetch user, THEN fetch order, THEN calculate total"
        // NO BLOCKING in between — each step starts when previous completes
        CompletableFuture<Double> pipeline = fetchUser(42)
            .thenCompose(user    -> fetchOrder(user))
            .thenCompose(order   -> calculateTotal(order))
            .thenApply  (total   -> total * 1.18)       // add tax (sync transformation)
            .exceptionally(ex -> {                       // handle ANY error in the chain
                System.out.println("Something failed: " + ex.getMessage());
                return 0.0; // fallback value
            });

        System.out.println("Pipeline submitted, doing other work...");
        System.out.println("Final total: " + pipeline.get()); // only block at the very end

        // --- COMBINING (thenCombine = run two futures in parallel, combine results) ---
        CompletableFuture<String> userFuture  = fetchUser(1);
        CompletableFuture<String> orderFuture = fetchOrder("User_1");

        CompletableFuture<String> combined = userFuture.thenCombine(
            orderFuture,
            (user, order) -> user + " placed " + order
        );
        System.out.println(combined.get());

        // --- anyOf / allOf ---
        CompletableFuture<Object> firstToFinish = CompletableFuture.anyOf(
            fetchUser(1), fetchUser(2), fetchUser(3)  // whichever finishes first wins
        );

        CompletableFuture<Void> allDone = CompletableFuture.allOf(
            fetchUser(1), fetchUser(2), fetchUser(3)  // wait for ALL
        );
        allDone.thenRun(() -> System.out.println("All 3 users fetched!"));

        pool.shutdown();
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }
}
```

---

### Parallel Streams — data parallelism made trivially easy

```java
import java.util.List;
import java.util.stream.*;

public class ParallelProcessing {
    public static void main(String[] args) {
        List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
                                         .boxed()
                                         .collect(Collectors.toList());

        // Sequential — one thread
        long seqSum = numbers.stream()
                             .filter(n -> n % 2 == 0)
                             .mapToLong(Integer::longValue)
                             .sum();

        // Parallel — automatically splits work across ForkJoinPool.commonPool()
        // (uses CPU core count - 1 threads by default)
        long parSum = numbers.parallelStream()
                             .filter(n -> n % 2 == 0)
                             .mapToLong(Integer::longValue)
                             .sum();

        System.out.println("Sequential: " + seqSum);
        System.out.println("Parallel  : " + parSum);
    }
}
```

---

### Problems that remained after Java 8

```
1. Platform threads are still OS threads — 1:1 mapping
   - Each thread = ~1MB stack = expensive
   - A typical JVM application handles ~thousands of threads max
   - Modern apps (microservices, high-concurrency APIs) need to handle
     TENS OF THOUSANDS of concurrent requests
   
2. "Thread per request" model breaks under high load
   - Web server gets 50,000 simultaneous requests?
   - You'd need 50,000 threads = ~50GB RAM just for stacks
   - Not feasible

3. Reactive programming (Project Reactor, RxJava) emerged as the workaround
   - "Don't block threads — use callbacks and event loops"
   - Worked, but was COMPLEX, hard to debug, hard to read
   - Stack traces became incomprehensible
   - New developers found it very hard to learn

4. CompletableFuture chains, while better than raw futures,
   still required reactive/callback thinking which is unnatural
```

---

## Chapter 5 — "The Reactive Band-aid" — Java 9 (2017)

Java 9 formalized reactive programming with the **Flow API** (Publisher-Subscriber model) and added **VarHandle** for low-level atomic operations.

```java
import java.util.concurrent.*;

// Flow API — reactive publisher/subscriber
public class ReactiveExample {
    public static void main(String[] args) throws InterruptedException {

        // SubmissionPublisher — a built-in Publisher implementation
        SubmissionPublisher<String> publisher = new SubmissionPublisher<>();

        // Subscriber
        Flow.Subscriber<String> subscriber = new Flow.Subscriber<>() {
            Flow.Subscription subscription;

            @Override
            public void onSubscribe(Flow.Subscription sub) {
                this.subscription = sub;
                sub.request(Long.MAX_VALUE); // request all items (no backpressure here)
            }

            @Override
            public void onNext(String item) {
                System.out.println("Received: " + item);
            }

            @Override
            public void onError(Throwable t)  { t.printStackTrace(); }

            @Override
            public void onComplete()          { System.out.println("Done!"); }
        };

        publisher.subscribe(subscriber);
        publisher.submit("Message 1");
        publisher.submit("Message 2");
        publisher.submit("Message 3");
        publisher.close();

        Thread.sleep(1000); // wait for async delivery
    }
}
```

---

### The big problem reactive didn't solve cleanly

```
Reactive code is HARD:
  - Natural code reads top to bottom (sequential thinking)
  - Reactive code requires thinking in streams, operators, schedulers
  - Debugging is painful — stack traces span multiple threads with no context
  - Learning curve is steep
  - Simple operations become chains of operators

Example of the "callback maze" even with CompletableFuture:
  fetchUser(id)
    .thenCompose(user -> fetchPermissions(user.getId()))
    .thenCompose(perms -> fetchResource(perms.getLevel()))
    .thenApply(resource -> transform(resource))
    .thenAccept(result -> saveToDb(result))
    .exceptionally(err -> handleError(err))
    .thenRun(() -> sendResponse());

vs the NATURAL way you'd want to write it (sequential):
  User user       = fetchUser(id);
  Perms perms     = fetchPermissions(user.getId());
  Resource res    = fetchResource(perms.getLevel());
  Result result   = transform(res);
  saveToDb(result);
  sendResponse();

Question: "Can't we write sequential code but get async non-blocking behavior?"
Answer: Yes — Virtual Threads (Project Loom)!
```

---

## Chapter 6 — "The Revolution" — Java 19-21 (2022-2023) — Project Loom

**Virtual Threads** (Preview Java 19-20, GA in **Java 21**) fundamentally change the game.

The core idea:
- **Platform thread** = managed by OS = expensive (~1MB stack, context-switch cost)
- **Virtual thread** = managed by JVM = cheap (starts at ~few KB, grows as needed)
- You can have **MILLIONS of virtual threads** in a single JVM
- Write **simple, sequential, blocking code** — JVM handles the non-blocking I/O under the hood

```java
import java.util.concurrent.*;

public class VirtualThreadsDemo {
    public static void main(String[] args) throws Exception {

        // Old way — platform thread (OS thread) — expensive
        Thread platformThread = new Thread(() -> System.out.println("Platform thread"));
        platformThread.start();
        platformThread.join();

        // New way — Virtual Thread — dirt cheap!
        Thread virtualThread = Thread.ofVirtual()
                                     .name("vt-worker-1")
                                     .start(() -> System.out.println("Virtual thread"));
        virtualThread.join();

        // Create 1,000,000 virtual threads — totally fine!
        // (would crash JVM with platform threads)
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 1_000_000; i++) {
                final int id = i;
                executor.submit(() -> {
                    Thread.sleep(Duration.ofMillis(100)); // blocking is fine now!
                    return "Result " + id;
                });
            }
        } // executor.close() waits for all tasks — auto-closeable

        System.out.println("1,000,000 virtual threads completed!");
    }
}
```

**The key insight — write simple blocking code again!**

```java
// OLD WAY (reactive / CompletableFuture — complex, hard to read)
CompletableFuture<Response> handleRequest(Request req) {
    return fetchUser(req.getUserId())           // async
        .thenCompose(user -> fetchOrders(user)) // async chain
        .thenCompose(orders -> fetchPrices(orders)) // async chain
        .thenApply(prices -> buildResponse(prices));
}

// NEW WAY (Virtual Threads — simple sequential code)
Response handleRequest(Request req) {
    User   user   = fetchUser(req.getUserId()); // looks blocking, but JVM parks the virtual thread
    Orders orders = fetchOrders(user);           // another parking point
    Prices prices = fetchPrices(orders);         // another parking point
    return buildResponse(prices);                // done!
}
// The JVM automatically "parks" the virtual thread when it blocks on I/O
// and runs other virtual threads on the underlying platform thread
// = non-blocking behavior with blocking-looking code!
```

---

### Structured Concurrency (Java 21 Preview) — organizing virtual threads cleanly

```java
import java.util.concurrent.*;

public class StructuredConcurrencyDemo {
    record OrderSummary(User user, Order order) {}

    static User  fetchUser(int id)  throws InterruptedException { Thread.sleep(200); return new User(id);  }
    static Order fetchOrder(int id) throws InterruptedException { Thread.sleep(300); return new Order(id); }

    static OrderSummary processOrder(int userId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            
            // Fork two tasks — run concurrently in virtual threads
            StructuredTaskScope.Subtask<User>  userFork  = scope.fork(() -> fetchUser(userId));
            StructuredTaskScope.Subtask<Order> orderFork = scope.fork(() -> fetchOrder(userId));

            scope.join();           // wait for both
            scope.throwIfFailed();  // if either failed, propagate exception

            // Both succeeded — combine results
            return new OrderSummary(userFork.get(), orderFork.get());
        }
        // When scope closes:
        //   - if fetchUser fails  → fetchOrder is cancelled automatically
        //   - if fetchOrder fails → fetchUser is cancelled automatically
        //   - No orphaned threads! No resource leaks!
    }
}
```

---

## The Full Timeline Summary

```
Java 1.0  (1996) — Thread, Runnable, synchronized, wait/notify
                   → Too raw, no result handling, expensive per-task threads

Java 1.3  (1999) — Timer/TimerTask
                   → Single-threaded timer, fragile, limited

Java 5    (2004) — java.util.concurrent (JSR-166 by Doug Lea)
                   ExecutorService, ThreadPool, Callable/Future,
                   Concurrent Collections, Atomic variables,
                   ReentrantLock, Synchronizers (CountDownLatch etc.)
                   → Great leap — pools, results, explicit locks, safe collections

Java 7    (2011) — ForkJoinPool, RecursiveTask/Action
                   → Divide-and-conquer parallelism with work-stealing

Java 8    (2014) — CompletableFuture, Parallel Streams, Lambdas
                   → Async pipelines, data parallelism, cleaner syntax

Java 9    (2017) — Flow API (reactive streams), VarHandle
                   → Formalized reactive programming, low-level atomics

Java 21   (2023) — Virtual Threads (Project Loom GA),
                   Structured Concurrency (Preview)
                   → Millions of cheap threads, write sequential code again,
                     structured lifecycle for concurrent tasks
```

---

```
The core evolution in ONE sentence per era:

1996 → "You are on your own, manage everything manually"
2004 → "Submit tasks, we manage threads for you"
2011 → "Split your big problem, we'll steal work between threads"
2014 → "Chain your async steps like a pipeline"
2023 → "Just write normal sequential code — we'll make it non-blocking"
```

That is the complete story of how concurrency evolved in Java — from manually wrestling with OS threads in 1996, to writing simple sequential code in 2023 with the JVM doing all the heavy lifting behind the scenes.