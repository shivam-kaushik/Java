# Section 1: Java Core

---

## 1.1 OOP — Encapsulation, Inheritance, Polymorphism, Abstraction

OOP organizes code around objects that combine state and behavior. In financial systems,
encapsulation protects sensitive fields (balance, account number) from direct mutation;
inheritance lets you model account hierarchies without duplicating logic; polymorphism lets
a single method call (`verify()`) behave differently for each account type; abstraction hides
payment-processing complexity behind a stable interface. At RBC you used this hierarchy every
day — an `InvestorAccount` IS-A `BankAccount`, but adds portfolio-specific verification.

**One-sentence interview answer:** "OOP encapsulates state behind controlled APIs, uses
inheritance to share behavior across a hierarchy, polymorphism to dispatch behavior at runtime,
and abstraction to hide implementation details behind contracts."

**Most likely follow-ups:**
1. What is the difference between an abstract class and an interface?
2. Can you have a constructor in an abstract class?
3. What is the Liskov Substitution Principle?

```java
// ── PaymentProcessor.java ─────────────────────────────────────────────────
// Abstract class: shared state + enforced contract.
// Use abstract class (not interface) when you need both a default
// implementation AND shared mutable state (processorId, auditLog).
public abstract class PaymentProcessor {

    private final String processorId;          // encapsulated — no public setter
    private final List<String> auditLog = new ArrayList<>();

    protected PaymentProcessor(String processorId) {
        this.processorId = processorId;
    }

    // Abstract: every subclass MUST define how it processes
    public abstract boolean process(double amount, String currency);

    // Template method: shared pre/post logic, variable core step
    public final boolean executePayment(double amount, String currency) {
        auditLog.add("START " + processorId + " amount=" + amount);
        boolean result = process(amount, currency);          // polymorphic dispatch
        auditLog.add("END success=" + result);
        return result;
    }

    public String getProcessorId() { return processorId; }
    public List<String> getAuditLog() { return Collections.unmodifiableList(auditLog); }
}

// ── Verifiable.java ───────────────────────────────────────────────────────
// Interface: pure contract, no state.
// Use an interface when you want multiple unrelated classes to share a contract.
public interface Verifiable {
    boolean verify(String token);
    default String verificationMethod() { return "DEFAULT"; }  // Java 8 default method
}

// ── BankAccount.java ──────────────────────────────────────────────────────
public class BankAccount {

    private final String accountNumber;     // encapsulated, immutable after construction
    private double balance;                 // encapsulated — mutated only through deposit/withdraw

    public BankAccount(String accountNumber, double initialBalance) {
        if (initialBalance < 0) throw new IllegalArgumentException("Initial balance cannot be negative");
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    // Controlled mutation — validation lives here, not in callers
    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Deposit amount must be positive");
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        this.balance -= amount;
    }

    public double getBalance() { return balance; }
    public String getAccountNumber() { return accountNumber; }

    @Override
    public String toString() {
        return String.format("BankAccount[%s, balance=%.2f]", accountNumber, balance);
    }
}

// ── InvestorAccount.java ──────────────────────────────────────────────────
// Inherits BankAccount, implements Verifiable, extends PaymentProcessor.
// This mirrors the RBC investor verification microservice where each investor
// account had its own JWT-based verification flow on top of core bank logic.
public class InvestorAccount extends BankAccount implements Verifiable {

    private final String investorId;
    private final String portfolioType;   // e.g. "EQUITY", "FIXED_INCOME", "BALANCED"
    private double portfolioValue;

    public InvestorAccount(String accountNumber, double balance,
                           String investorId, String portfolioType) {
        super(accountNumber, balance);    // call parent constructor — required
        this.investorId = investorId;
        this.portfolioType = portfolioType;
    }

    // Polymorphism: overrides toString() — a call on BankAccount ref will use THIS version
    @Override
    public String toString() {
        return String.format("InvestorAccount[%s, investor=%s, portfolio=%s, value=%.2f]",
                getAccountNumber(), investorId, portfolioType, portfolioValue);
    }

    // Implements Verifiable contract
    @Override
    public boolean verify(String jwtToken) {
        // In production (RBC): call Interac API with JWT, check claims, validate expiry
        if (jwtToken == null || jwtToken.isBlank()) return false;
        System.out.printf("Verifying investor %s with token %s...%n", investorId,
                jwtToken.substring(0, Math.min(8, jwtToken.length())) + "***");
        return jwtToken.startsWith("eyJ");   // simplified — real impl decodes + validates
    }

    @Override
    public String verificationMethod() { return "JWT_INTERAC"; }

    public void updatePortfolioValue(double value) {
        if (value < 0) throw new IllegalArgumentException("Portfolio value cannot be negative");
        this.portfolioValue = value;
    }

    public String getInvestorId() { return investorId; }
    public double getPortfolioValue() { return portfolioValue; }
}

// ── InteracPaymentProcessor.java ─────────────────────────────────────────
// Concrete PaymentProcessor for Interac — the actual RBC payment flow
public class InteracPaymentProcessor extends PaymentProcessor {

    private final InvestorAccount account;

    public InteracPaymentProcessor(InvestorAccount account) {
        super("INTERAC-" + account.getInvestorId());
        this.account = account;
    }

    @Override
    public boolean process(double amount, String currency) {
        // Real logic: call Interac API, debit account, record in PostgreSQL
        if (!"CAD".equals(currency)) {
            System.out.println("Interac only supports CAD");
            return false;
        }
        try {
            account.withdraw(amount);
            System.out.printf("Processed %.2f %s for investor %s%n",
                    amount, currency, account.getInvestorId());
            return true;
        } catch (IllegalStateException e) {
            System.out.println("Payment failed: " + e.getMessage());
            return false;
        }
    }
}

// ── OOPDemo.java ─────────────────────────────────────────────────────────
public class OOPDemo {
    public static void main(String[] args) {
        InvestorAccount investor = new InvestorAccount(
                "ACC-001", 50000.0, "INV-RBC-42", "EQUITY");
        investor.updatePortfolioValue(125000.0);

        // Polymorphism: variable typed as BankAccount, object is InvestorAccount
        BankAccount account = investor;
        System.out.println(account);   // calls InvestorAccount.toString() — runtime dispatch

        // Verifiable contract
        System.out.println(investor.verify("eyJhbGciOiJSUzI1NiJ9.payload.sig"));  // true
        System.out.println(investor.verificationMethod());                          // JWT_INTERAC

        // Template method pattern in PaymentProcessor
        InteracPaymentProcessor processor = new InteracPaymentProcessor(investor);
        processor.executePayment(1000.0, "CAD");
        System.out.println(processor.getAuditLog());
    }
}
```

---

## 1.2 Collections Framework

The Java Collections Framework provides data structures with well-defined performance
guarantees. In financial services, choosing the wrong collection can cause silent performance
degradation — using `LinkedList` for indexed access or `HashMap` where insertion order matters.
The rule of thumb: pick the collection whose performance contract matches your *most frequent
operation*, not the one you're most familiar with.

**One-sentence interview answer:** "I choose collections based on access pattern: `HashMap` for
O(1) lookup, `LinkedHashMap` when insertion order matters, `TreeMap` for sorted keys,
`ArrayList` for indexed access, and `PriorityQueue` for priority-ordered processing."

**Most likely follow-ups:**
1. What happens when two keys hash to the same bucket in a `HashMap`?
2. How does `ConcurrentHashMap` differ from `Collections.synchronizedMap()`?
3. What is the time complexity of `TreeMap.get()`?

```java
import java.util.*;
import java.math.BigDecimal;

// ── CollectionsDemo.java ──────────────────────────────────────────────────

public class CollectionsDemo {

    // ── HashMap vs LinkedHashMap vs TreeMap ───────────────────────────────
    //
    //  HashMap        O(1) get/put  — no order guarantee     → use for fast lookup
    //  LinkedHashMap  O(1) get/put  — insertion order kept   → use for LRU cache, audit trail
    //  TreeMap        O(log n)      — keys sorted naturally   → use when sorted iteration needed
    //
    // At RBC: HashMap for the token cache (fast lookup by token hash),
    // LinkedHashMap for audit logs (insertion = chronological order),
    // TreeMap for account statements (sorted by account number).

    public static void mapComparison() {
        // HashMap — unordered, fastest
        Map<String, Double> balanceCache = new HashMap<>();
        balanceCache.put("INV-003", 75000.0);
        balanceCache.put("INV-001", 120000.0);
        balanceCache.put("INV-002", 43000.0);
        System.out.println("HashMap (no guaranteed order): " + balanceCache.keySet());

        // LinkedHashMap — insertion order preserved
        Map<String, String> authAuditLog = new LinkedHashMap<>();
        authAuditLog.put("2025-08-01T09:00", "INV-001 LOGIN SUCCESS");
        authAuditLog.put("2025-08-01T09:05", "INV-002 LOGIN SUCCESS");
        authAuditLog.put("2025-08-01T09:10", "INV-001 LOGOUT");
        System.out.println("LinkedHashMap (insertion order): " + authAuditLog.values());

        // TreeMap — sorted by key (natural order or Comparator)
        TreeMap<String, Double> sortedByAccount = new TreeMap<>(balanceCache);
        System.out.println("TreeMap (sorted): " + sortedByAccount);
        // First/last account — O(log n)
        System.out.println("Highest account number: " + sortedByAccount.lastKey());
    }

    // ── ArrayList vs LinkedList ───────────────────────────────────────────
    //
    //  Operation          ArrayList    LinkedList
    //  get(index)         O(1)         O(n)         ← ArrayList wins for random access
    //  add(end)           O(1) amort.  O(1)
    //  add(middle)        O(n)         O(1)*        ← LinkedList wins for mid-insert
    //  remove(index)      O(n)         O(n)*        (* still O(n) to find the node)
    //  memory             contiguous   node+pointers  ← ArrayList better cache locality
    //
    // Verdict: Use ArrayList almost always. LinkedList only if you have a known iterator
    // position and need frequent mid-list insertions (rare in practice).

    public static void listComparison() {
        List<String> transactionIds = new ArrayList<>(10_000);  // pre-size to avoid resize copies
        transactionIds.add("TXN-001");
        transactionIds.add("TXN-002");
        System.out.println("ArrayList get(0): " + transactionIds.get(0));  // O(1)

        Deque<String> processingQueue = new LinkedList<>();  // Used as a Queue, not List
        processingQueue.offer("TXN-003");
        processingQueue.offer("TXN-004");
        System.out.println("LinkedList as Queue poll: " + processingQueue.poll());
    }

    // ── HashSet vs TreeSet ────────────────────────────────────────────────
    //
    //  HashSet:  O(1) add/contains — unordered     → dedup investor IDs
    //  TreeSet:  O(log n)          — sorted order   → sorted unique account numbers
    //
    public static void setComparison() {
        Set<String> processedTokens = new HashSet<>();   // fast dedup of auth tokens
        processedTokens.add("token-abc");
        processedTokens.add("token-xyz");
        processedTokens.add("token-abc");   // duplicate silently ignored
        System.out.println("HashSet size (deduped): " + processedTokens.size());  // 2

        TreeSet<String> sortedAccountIds = new TreeSet<>();
        sortedAccountIds.add("ACC-300");
        sortedAccountIds.add("ACC-100");
        sortedAccountIds.add("ACC-200");
        System.out.println("TreeSet (sorted): " + sortedAccountIds);  // [ACC-100, ACC-200, ACC-300]
        System.out.println("Range view ACC-100 to ACC-200: " + sortedAccountIds.subSet("ACC-100", true, "ACC-200", true));
    }

    // ── PriorityQueue ─────────────────────────────────────────────────────
    // Min-heap by default. Used at RBC for priority-ordering authentication
    // requests: high-net-worth investors get priority processing.
    record AuthRequest(String investorId, int priority, String requestType) {}

    public static void priorityQueueExample() {
        // Max-heap by priority (highest priority number = first out)
        PriorityQueue<AuthRequest> authQueue = new PriorityQueue<>(
                Comparator.comparingInt(AuthRequest::priority).reversed()
        );

        authQueue.offer(new AuthRequest("INV-001", 5, "INTERAC"));
        authQueue.offer(new AuthRequest("INV-002", 1, "OAUTH"));
        authQueue.offer(new AuthRequest("INV-003", 10, "INTERAC"));  // VIP

        while (!authQueue.isEmpty()) {
            AuthRequest req = authQueue.poll();
            System.out.printf("Processing [priority=%d] %s → %s%n",
                    req.priority(), req.investorId(), req.requestType());
        }
        // Output: priority 10, then 5, then 1
    }

    // ── Transaction Frequency Map ─────────────────────────────────────────
    // Real use case: at RBC, count how many times each investor authenticated
    // in a rolling window to detect anomalous activity.
    public static Map<String, Integer> buildTransactionFrequencyMap(List<String> investorIds) {
        Map<String, Integer> frequencyMap = new HashMap<>();

        for (String id : investorIds) {
            // getOrDefault avoids NullPointerException on first occurrence
            frequencyMap.put(id, frequencyMap.getOrDefault(id, 0) + 1);
            // Java 8+ alternative: frequencyMap.merge(id, 1, Integer::sum);
        }

        // Sort by frequency descending to find most-active investors
        frequencyMap.entrySet().stream()
                .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
                .forEach(e -> System.out.printf("  %s → %d transactions%n", e.getKey(), e.getValue()));

        return frequencyMap;
    }

    // ── Sort Investor Accounts by Balance using TreeMap ───────────────────
    public static void sortByBalance() {
        Map<String, Double> accounts = new HashMap<>();
        accounts.put("INV-001", 120000.0);
        accounts.put("INV-002", 43000.0);
        accounts.put("INV-003", 75000.0);

        // TreeMap with custom comparator — sort by balance (value), not key
        // NOTE: TreeMap comparator is on keys; for value-sorting, use a List
        List<Map.Entry<String, Double>> sorted = new ArrayList<>(accounts.entrySet());
        sorted.sort(Map.Entry.comparingByValue(Comparator.reverseOrder()));

        System.out.println("Investors sorted by balance (descending):");
        sorted.forEach(e -> System.out.printf("  %s: $%.2f%n", e.getKey(), e.getValue()));
    }

    public static void main(String[] args) {
        System.out.println("=== Map Comparison ===");
        mapComparison();

        System.out.println("\n=== List Comparison ===");
        listComparison();

        System.out.println("\n=== Set Comparison ===");
        setComparison();

        System.out.println("\n=== PriorityQueue ===");
        priorityQueueExample();

        System.out.println("\n=== Transaction Frequency ===");
        buildTransactionFrequencyMap(List.of(
                "INV-001", "INV-002", "INV-001", "INV-003", "INV-001", "INV-002"));

        System.out.println("\n=== Sort by Balance ===");
        sortByBalance();
    }
}
```

---

## 1.3 Multithreading + Concurrency

Java's threading model lets multiple tasks run concurrently, critical for high-throughput
systems like the RBC authentication service (10,000+ daily authentications). The key insight
is that *shared mutable state* is the source of all concurrency bugs — every synchronization
primitive (`synchronized`, `volatile`, locks) exists to either serialize access to shared
state or to eliminate it entirely. `ExecutorService` is preferred over raw `Thread` creation
because it manages the thread lifecycle and prevents thread-leak bugs.

**One-sentence interview answer:** "I use `ExecutorService` to manage thread pools, `synchronized`
or `ReentrantLock` to serialize access to shared state, `volatile` for visibility guarantees
on single flags, and `CountDownLatch`/`CyclicBarrier` to coordinate multi-phase parallel work."

**Most likely follow-ups:**
1. What is the difference between `synchronized` and `ReentrantLock`?
2. How does `volatile` differ from `synchronized`?
3. How would you design a thread-safe singleton?

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.*;
import java.util.logging.Logger;

// ── AuthenticationRequest.java ────────────────────────────────────────────
record AuthenticationRequest(String investorId, String token, String protocol) {}

// ── AuthenticationResult.java ────────────────────────────────────────────
record AuthenticationResult(String investorId, boolean success, long processingTimeMs) {}

// ── ParallelAuthProcessor.java ────────────────────────────────────────────
// Models the RBC Interac authentication service: 10,000+ daily auth requests
// processed in parallel, with metrics tracking and graceful shutdown.
public class ParallelAuthProcessor {

    private static final Logger log = Logger.getLogger(ParallelAuthProcessor.class.getName());

    // volatile: single boolean flag read by many threads — visibility guarantee
    // without the overhead of full synchronization. Safe here because we never
    // do a read-modify-write on this flag (it's only ever set to true once).
    private volatile boolean isShuttingDown = false;

    // AtomicInteger: thread-safe counter without synchronized block.
    // Use Atomic classes for single-variable counters/flags to avoid lock overhead.
    private final AtomicInteger successCount = new AtomicInteger(0);
    private final AtomicInteger failureCount = new AtomicInteger(0);

    // ThreadPoolExecutor gives fine-grained control over pool behavior.
    // corePoolSize=10: always-alive threads for baseline RBC traffic
    // maxPoolSize=50:  burst capacity for market-open authentication spike
    // keepAliveTime=60s: extra threads die after 60s of inactivity
    // LinkedBlockingQueue(1000): bounded queue — reject if backpressure exceeds 1000
    private final ThreadPoolExecutor executor = new ThreadPoolExecutor(
            10, 50, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            new ThreadFactory() {
                private final AtomicInteger threadNum = new AtomicInteger(0);
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, "auth-worker-" + threadNum.incrementAndGet());
                    t.setDaemon(true);  // don't block JVM shutdown
                    return t;
                }
            },
            // RejectedExecutionHandler: caller runs the task itself rather than dropping it.
            // In production use a Dead Letter Queue or return 429 to the client instead.
            new ThreadPoolExecutor.CallerRunsPolicy()
    );

    // ── Process a batch in parallel ───────────────────────────────────────
    // CountDownLatch: wait for ALL requests in a batch to complete before
    // returning results. This was used in the RBC batch verification flow
    // where a daily reconciliation needed all 10k verifications to finish.
    public List<AuthenticationResult> processBatch(List<AuthenticationRequest> requests)
            throws InterruptedException {

        CountDownLatch latch = new CountDownLatch(requests.size());
        List<AuthenticationResult> results = Collections.synchronizedList(new ArrayList<>());

        for (AuthenticationRequest req : requests) {
            if (isShuttingDown) break;

            executor.submit(() -> {
                try {
                    AuthenticationResult result = authenticate(req);
                    results.add(result);
                    if (result.success()) successCount.incrementAndGet();
                    else failureCount.incrementAndGet();
                } finally {
                    latch.countDown();  // ALWAYS countdown, even on exception
                }
            });
        }

        // Wait max 30 seconds for all to finish — prevents hanging indefinitely
        boolean completed = latch.await(30, TimeUnit.SECONDS);
        if (!completed) log.warning("Batch did not complete within 30s timeout");

        return results;
    }

    // ── Two-phase processing with CyclicBarrier ───────────────────────────
    // CyclicBarrier: all threads synchronize at a barrier, then continue.
    // Different from CountDownLatch: reusable, and all threads are participants.
    // Use case: first verify tokens in parallel, then AFTER all verified,
    // record to PostgreSQL in parallel. The barrier is the gate between phases.
    public void twoPhaseProcessing(List<AuthenticationRequest> requests) throws Exception {

        int numWorkers = Math.min(requests.size(), 10);
        List<AuthenticationResult> phase1Results = Collections.synchronizedList(new ArrayList<>());

        CyclicBarrier barrier = new CyclicBarrier(numWorkers, () -> {
            // This runs once when all numWorkers threads reach the barrier
            log.info("Phase 1 (verify) complete. " + phase1Results.size() +
                     " results ready. Starting Phase 2 (persist)...");
        });

        List<Future<?>> futures = new ArrayList<>();
        int batchSize = (int) Math.ceil((double) requests.size() / numWorkers);

        for (int i = 0; i < numWorkers; i++) {
            final int start = i * batchSize;
            final int end = Math.min(start + batchSize, requests.size());
            if (start >= requests.size()) break;

            futures.add(executor.submit(() -> {
                // Phase 1: verify tokens
                for (int idx = start; idx < end; idx++) {
                    phase1Results.add(authenticate(requests.get(idx)));
                }
                try {
                    barrier.await(10, TimeUnit.SECONDS);  // wait for all workers to finish phase 1
                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Barrier interrupted", e);
                }
                // Phase 2: persist results (runs after barrier)
                persistResults(phase1Results.subList(start, Math.min(end, phase1Results.size())));
            }));
        }

        for (Future<?> f : futures) f.get();  // propagate exceptions from workers
    }

    // ── Deadlock demonstration + prevention ──────────────────────────────
    // Deadlock: Thread A holds lock1, waits for lock2.
    //           Thread B holds lock2, waits for lock1. Neither can proceed.
    //
    // Prevention: ALWAYS acquire multiple locks in a consistent global order.
    // Here: always lock the account with the lower ID first.
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    // DEADLOCK-PRONE — do not use in production
    public void deadlockProneTransfer(String fromId, String toId, double amount) {
        // If thread 1 runs this with (A→B) and thread 2 runs this with (B→A),
        // thread 1 holds lockA waiting for lockB,
        // thread 2 holds lockB waiting for lockA → deadlock.
        Object first = fromId.compareTo(toId) < 0 ? lockA : lockB;   // WRONG: not enforced globally
        Object second = first == lockA ? lockB : lockA;
        synchronized (first) {
            synchronized (second) {
                // transfer logic
            }
        }
    }

    // DEADLOCK-SAFE: enforce global lock ordering by comparing account IDs
    public void safeTransfer(String fromAccountId, String toAccountId, double amount) {
        // Determine lock order by natural string comparison — globally consistent
        String firstLock  = fromAccountId.compareTo(toAccountId) < 0 ? fromAccountId : toAccountId;
        String secondLock = firstLock.equals(fromAccountId) ? toAccountId : fromAccountId;

        synchronized (firstLock.intern()) {    // intern() so same string == same monitor
            synchronized (secondLock.intern()) {
                log.info("Transferring " + amount + " from " + fromAccountId + " to " + toAccountId);
                // actual transfer logic
            }
        }
    }

    // ── Simulated authentication ──────────────────────────────────────────
    private AuthenticationResult authenticate(AuthenticationRequest req) {
        long start = System.currentTimeMillis();
        try {
            Thread.sleep(10);  // simulate Interac API latency
            boolean success = req.token() != null && req.token().startsWith("eyJ");
            return new AuthenticationResult(req.investorId(), success,
                    System.currentTimeMillis() - start);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();  // restore interrupt flag — critical!
            return new AuthenticationResult(req.investorId(), false, 0);
        }
    }

    private void persistResults(List<AuthenticationResult> results) {
        // In production: batch insert into PostgreSQL using JdbcTemplate.batchUpdate()
        log.info("Persisting " + results.size() + " results to PostgreSQL");
    }

    public void shutdown() {
        isShuttingDown = true;
        executor.shutdown();
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow();  // force stop after timeout
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
        log.info("Shutdown complete. Success: " + successCount + " Failures: " + failureCount);
    }

    public static void main(String[] args) throws Exception {
        ParallelAuthProcessor processor = new ParallelAuthProcessor();

        List<AuthenticationRequest> requests = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            String token = i % 10 == 0 ? "invalid-token" : "eyJhbGciOiJSUzI1NiJ9.test";
            requests.add(new AuthenticationRequest("INV-" + i, token, "INTERAC"));
        }

        List<AuthenticationResult> results = processor.processBatch(requests);
        System.out.println("Processed: " + results.size());
        processor.shutdown();
    }
}
```

---

## 1.4 Java 8+ Features

Java 8 introduced functional programming constructs that dramatically reduce boilerplate.
Streams enable declarative data processing — you describe *what* you want (filter investors
with balance > 100k, group by portfolio type) rather than *how* to iterate. At Ontario
Government, Streams replaced 50-line loop blocks with 5-line pipelines when processing
caseworker data. `Optional` eliminates null checks that litter financial code.

**One-sentence interview answer:** "Java 8 Streams let me write declarative pipelines
(filter → map → collect) with lazy evaluation; `Optional` makes null-safety explicit;
lambdas and method references eliminate anonymous class boilerplate."

**Most likely follow-ups:**
1. What is the difference between `map()` and `flatMap()`?
2. Is a Stream reusable after a terminal operation?
3. What is the difference between `findFirst()` and `findAny()` in a parallel stream?

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;
import java.math.BigDecimal;
import java.time.LocalDate;

// ── InvestorTransaction.java ──────────────────────────────────────────────
record InvestorTransaction(
        String transactionId,
        String investorId,
        String type,          // "BUY", "SELL", "DIVIDEND", "FEE"
        double amount,
        String currency,
        LocalDate date,
        String portfolioType  // "EQUITY", "FIXED_INCOME", "BALANCED"
) {}

// ── Java8FeaturesDemo.java ────────────────────────────────────────────────
public class Java8FeaturesDemo {

    // ── Lambdas + Functional Interfaces ───────────────────────────────────
    // Functional interface: exactly one abstract method. Can be a lambda target.
    // Built-in: Predicate<T>, Function<T,R>, Consumer<T>, Supplier<T>, BiFunction<T,U,R>
    @FunctionalInterface
    interface TransactionFilter {
        boolean test(InvestorTransaction txn);  // custom functional interface
    }

    public static void lambdaExamples(List<InvestorTransaction> txns) {
        // Lambda vs anonymous class (pre-Java-8)
        // Old way:
        Comparator<InvestorTransaction> oldWay = new Comparator<>() {
            @Override public int compare(InvestorTransaction a, InvestorTransaction b) {
                return Double.compare(a.amount(), b.amount());
            }
        };

        // Lambda (Java 8):
        Comparator<InvestorTransaction> byAmount = (a, b) -> Double.compare(a.amount(), b.amount());

        // Method reference — even cleaner when lambda just delegates to existing method
        Comparator<InvestorTransaction> byAmountRef = Comparator.comparingDouble(InvestorTransaction::amount);

        // Predicate composition — combines filters without nested ifs
        Predicate<InvestorTransaction> isBuy      = t -> "BUY".equals(t.type());
        Predicate<InvestorTransaction> isHighValue = t -> t.amount() > 10_000;
        Predicate<InvestorTransaction> isCAD      = t -> "CAD".equals(t.currency());

        // and() / or() / negate() for readable composition
        Predicate<InvestorTransaction> highValueBuyInCAD = isBuy.and(isHighValue).and(isCAD);
        long count = txns.stream().filter(highValueBuyInCAD).count();
        System.out.println("High-value CAD buys: " + count);

        // Custom functional interface
        TransactionFilter equityFilter = txn -> "EQUITY".equals(txn.portfolioType());
        txns.stream().filter(equityFilter::test).forEach(t ->
                System.out.println("  Equity txn: " + t.transactionId()));
    }

    // ── Streams: filter, map, reduce, flatMap, collect ─────────────────
    public static void streamPipelines(List<InvestorTransaction> txns) {

        // filter + map + collect: extract IDs of all BUY transactions over $5000
        List<String> largeBuyIds = txns.stream()
                .filter(t -> "BUY".equals(t.type()))
                .filter(t -> t.amount() > 5_000)
                .map(InvestorTransaction::transactionId)
                .sorted()
                .collect(Collectors.toList());
        System.out.println("Large buy IDs: " + largeBuyIds);

        // reduce: sum total investment amount for a specific investor
        double totalInvested = txns.stream()
                .filter(t -> "INV-001".equals(t.investorId()))
                .filter(t -> "BUY".equals(t.type()))
                .mapToDouble(InvestorTransaction::amount)
                .reduce(0.0, Double::sum);  // identity + accumulator
        System.out.printf("INV-001 total invested: $%.2f%n", totalInvested);

        // mapToDouble + sum — more efficient than reduce for primitives
        double totalFees = txns.stream()
                .filter(t -> "FEE".equals(t.type()))
                .mapToDouble(InvestorTransaction::amount)
                .sum();

        // flatMap: flatten nested structures.
        // Example: each investor has multiple portfolios, each portfolio has transactions
        // flatMap collapses List<List<Transaction>> → Stream<Transaction>
        List<List<InvestorTransaction>> portfolioBatches = List.of(
                txns.subList(0, txns.size() / 2),
                txns.subList(txns.size() / 2, txns.size())
        );
        List<InvestorTransaction> allTxns = portfolioBatches.stream()
                .flatMap(Collection::stream)   // flatten batches into single stream
                .collect(Collectors.toList());
        System.out.println("Flattened txn count: " + allTxns.size());

        // groupingBy: group transactions by portfolio type
        Map<String, List<InvestorTransaction>> byPortfolio = txns.stream()
                .collect(Collectors.groupingBy(InvestorTransaction::portfolioType));
        byPortfolio.forEach((type, list) ->
                System.out.printf("  %s: %d transactions%n", type, list.size()));

        // groupingBy + downstream collector: sum by portfolio type
        Map<String, Double> totalByPortfolio = txns.stream()
                .filter(t -> "BUY".equals(t.type()))
                .collect(Collectors.groupingBy(
                        InvestorTransaction::portfolioType,
                        Collectors.summingDouble(InvestorTransaction::amount)
                ));
        System.out.println("Buy totals by portfolio: " + totalByPortfolio);

        // toMap: build investorId → total balance map
        Map<String, Double> investorTotals = txns.stream()
                .collect(Collectors.toMap(
                        InvestorTransaction::investorId,
                        InvestorTransaction::amount,
                        Double::sum   // merge function for duplicate keys
                ));

        // Statistics — useful for dashboards
        DoubleSummaryStatistics stats = txns.stream()
                .mapToDouble(InvestorTransaction::amount)
                .summaryStatistics();
        System.out.printf("Txn stats — count:%d min:%.2f max:%.2f avg:%.2f%n",
                stats.getCount(), stats.getMin(), stats.getMax(), stats.getAverage());
    }

    // ── Optional ──────────────────────────────────────────────────────────
    // In financial code, a lookup that returns null and is then dereferenced
    // causes a NullPointerException in production. Optional makes the "might
    // not be present" contract explicit and forces callers to handle it.
    public static Optional<InvestorTransaction> findLargestTransaction(
            List<InvestorTransaction> txns, String investorId) {

        return txns.stream()
                .filter(t -> investorId.equals(t.investorId()))
                .max(Comparator.comparingDouble(InvestorTransaction::amount));
    }

    public static void optionalExamples(List<InvestorTransaction> txns) {

        // BAD — old null check style:
        // InvestorTransaction t = findLargest(txns, "INV-001");
        // if (t != null) { ... }   ← easy to forget, causes NPE in prod

        // GOOD — Optional forces the caller to handle absence
        findLargestTransaction(txns, "INV-001")
                .map(t -> String.format("Largest: %s $%.2f", t.transactionId(), t.amount()))
                .ifPresentOrElse(
                        System.out::println,
                        () -> System.out.println("No transactions found for INV-001")
                );

        // orElse vs orElseGet:
        // orElse(default):    the default is ALWAYS evaluated (even if value is present)
        // orElseGet(supplier): the supplier is ONLY called if value is absent — prefer this
        //                      when the default is expensive to compute
        double amount = findLargestTransaction(txns, "INV-MISSING")
                .map(InvestorTransaction::amount)
                .orElseGet(() -> 0.0);  // supplier: only evaluated if absent

        // orElseThrow: for cases where absence is a programming error
        InvestorTransaction firstTxn = txns.stream().findFirst()
                .orElseThrow(() -> new IllegalStateException("Transaction list cannot be empty"));

        // Chaining Optional — replaces nested null checks
        Optional<String> currencyOfLargest = findLargestTransaction(txns, "INV-001")
                .map(InvestorTransaction::currency)
                .filter(c -> !"USD".equals(c));
    }

    // ── Filter + Sort + Group investor portfolios ─────────────────────────
    public static Map<String, DoubleSummaryStatistics> analyzePortfolios(
            List<InvestorTransaction> txns) {

        return txns.stream()
                .filter(t -> t.date().isAfter(LocalDate.now().minusMonths(3)))  // last quarter
                .filter(t -> List.of("BUY", "SELL").contains(t.type()))
                .sorted(Comparator.comparing(InvestorTransaction::date)
                        .thenComparing(InvestorTransaction::amount).reversed())
                .collect(Collectors.groupingBy(
                        InvestorTransaction::portfolioType,
                        Collectors.summarizingDouble(InvestorTransaction::amount)
                ));
    }

    public static void main(String[] args) {
        List<InvestorTransaction> txns = List.of(
                new InvestorTransaction("TXN-001", "INV-001", "BUY",   15000.0, "CAD",
                        LocalDate.now().minusDays(10), "EQUITY"),
                new InvestorTransaction("TXN-002", "INV-001", "FEE",     250.0, "CAD",
                        LocalDate.now().minusDays(9),  "EQUITY"),
                new InvestorTransaction("TXN-003", "INV-002", "BUY",    8000.0, "CAD",
                        LocalDate.now().minusDays(5),  "FIXED_INCOME"),
                new InvestorTransaction("TXN-004", "INV-002", "SELL",   3000.0, "USD",
                        LocalDate.now().minusDays(3),  "FIXED_INCOME"),
                new InvestorTransaction("TXN-005", "INV-003", "BUY",   22000.0, "CAD",
                        LocalDate.now().minusDays(1),  "BALANCED")
        );

        System.out.println("=== Lambdas ===");
        lambdaExamples(txns);

        System.out.println("\n=== Streams ===");
        streamPipelines(txns);

        System.out.println("\n=== Optional ===");
        optionalExamples(txns);

        System.out.println("\n=== Portfolio Analysis ===");
        analyzePortfolios(txns).forEach((type, stats) ->
                System.out.printf("  %s: count=%d total=%.2f avg=%.2f%n",
                        type, stats.getCount(), stats.getSum(), stats.getAverage()));
    }
}
```

---

## 1.5 Design Patterns

Design patterns are reusable solutions to recurring OOP design problems. In interviews, knowing
*when* to apply a pattern matters more than memorizing the structure. At Thales you used Factory
and Strategy for REST vs gRPC classification; at RBC you used Builder for complex JWT claims
objects and Observer for real-time portfolio event propagation.

**One-sentence interview answer:** "I use Factory to decouple object creation from usage,
Strategy to swap algorithms at runtime, Builder for complex object construction, Singleton for
shared resources with controlled lifecycle, and Observer for event-driven decoupling."

**Most likely follow-ups:**
1. How do you make a Singleton thread-safe without synchronizing every call?
2. What is the difference between Factory Method and Abstract Factory?
3. How does Spring use these patterns internally?

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;

// ═══════════════════════════════════════════════════════
// FACTORY PATTERN — REST vs gRPC Authentication Handler
// (Thales: Python framework, now in Java for RBC)
// ═══════════════════════════════════════════════════════

interface AuthenticationHandler {
    AuthenticationResult handle(AuthenticationRequest request);
    String protocol();
}

class RestAuthenticationHandler implements AuthenticationHandler {
    @Override
    public AuthenticationResult handle(AuthenticationRequest request) {
        System.out.println("Processing REST auth for " + request.investorId());
        // In production: validate JWT Bearer token, call /verify endpoint
        boolean success = request.token() != null && request.token().startsWith("eyJ");
        return new AuthenticationResult(request.investorId(), success, 15L);
    }
    @Override public String protocol() { return "REST"; }
}

class GrpcAuthenticationHandler implements AuthenticationHandler {
    @Override
    public AuthenticationResult handle(AuthenticationRequest request) {
        System.out.println("Processing gRPC auth for " + request.investorId());
        // In production: validate gRPC metadata token, call AuthService/Verify RPC
        boolean success = request.token() != null && request.token().length() > 20;
        return new AuthenticationResult(request.investorId(), success, 8L);
    }
    @Override public String protocol() { return "gRPC"; }
}

class InteracAuthenticationHandler implements AuthenticationHandler {
    @Override
    public AuthenticationResult handle(AuthenticationRequest request) {
        System.out.println("Processing Interac auth for " + request.investorId());
        return new AuthenticationResult(request.investorId(), true, 45L);
    }
    @Override public String protocol() { return "INTERAC"; }
}

// Factory: centralizes creation logic; callers never use 'new' directly
// This is exactly what Thales needed — classify incoming requests by protocol
// and dispatch to the correct handler without a massive if-else chain.
class AuthHandlerFactory {
    // Registry pattern inside Factory — easy to add new protocols without changing Factory
    private static final Map<String, Supplier<AuthenticationHandler>> registry = new HashMap<>();

    static {
        registry.put("REST",    RestAuthenticationHandler::new);
        registry.put("GRPC",    GrpcAuthenticationHandler::new);
        registry.put("INTERAC", InteracAuthenticationHandler::new);
    }

    // Register new protocols at runtime (open/closed principle)
    public static void register(String protocol, Supplier<AuthenticationHandler> supplier) {
        registry.put(protocol.toUpperCase(), supplier);
    }

    public static AuthenticationHandler create(String protocol) {
        Supplier<AuthenticationHandler> supplier = registry.get(protocol.toUpperCase());
        if (supplier == null) {
            throw new IllegalArgumentException("Unknown protocol: " + protocol +
                    ". Supported: " + registry.keySet());
        }
        return supplier.get();
    }
}

// ═══════════════════════════════════════════════════════
// SINGLETON PATTERN — Thread-safe (Bill Pugh / enum approach)
// ═══════════════════════════════════════════════════════

// Option 1: Bill Pugh (lazy, thread-safe, no synchronized overhead on every call)
class TokenBlacklistCache {
    // Inner class is loaded only when getInstance() is first called.
    // JVM class loading is inherently thread-safe — no synchronized needed.
    private static class Holder {
        private static final TokenBlacklistCache INSTANCE = new TokenBlacklistCache();
    }

    private final Set<String> blacklist = ConcurrentHashMap.newKeySet();

    private TokenBlacklistCache() {
        System.out.println("TokenBlacklistCache initialized (once)");
    }

    public static TokenBlacklistCache getInstance() { return Holder.INSTANCE; }

    public void blacklist(String token) { blacklist.add(token); }
    public boolean isBlacklisted(String token) { return blacklist.contains(token); }
    public int size() { return blacklist.size(); }
}

// Option 2: Enum Singleton — JVM guarantees exactly one instance, serialization-safe
enum AuditLogger {
    INSTANCE;

    public void log(String message) {
        System.out.println("[AUDIT] " + System.currentTimeMillis() + " | " + message);
    }
}

// ═══════════════════════════════════════════════════════
// STRATEGY PATTERN — Payment Processing Strategies
// ═══════════════════════════════════════════════════════

// Strategy interface — swappable algorithm
interface FeeCalculationStrategy {
    double calculateFee(double tradeAmount, String portfolioType);
    String strategyName();
}

class EquityFeeStrategy implements FeeCalculationStrategy {
    @Override
    public double calculateFee(double amount, String portfolioType) {
        return amount * 0.0025;  // 0.25% for equity trades
    }
    @Override public String strategyName() { return "EQUITY_FEE"; }
}

class FixedIncomeFeeStrategy implements FeeCalculationStrategy {
    @Override
    public double calculateFee(double amount, String portfolioType) {
        return Math.min(amount * 0.001, 500.0);  // 0.10% capped at $500
    }
    @Override public String strategyName() { return "FIXED_INCOME_FEE"; }
}

class InstitutionalFeeStrategy implements FeeCalculationStrategy {
    private static final double FLAT_RATE = 9.99;
    @Override
    public double calculateFee(double amount, String portfolioType) {
        return FLAT_RATE;  // flat fee for institutional accounts
    }
    @Override public String strategyName() { return "INSTITUTIONAL_FLAT"; }
}

// Context class — uses a strategy without knowing which one
class TradeExecutor {
    private FeeCalculationStrategy feeStrategy;

    public TradeExecutor(FeeCalculationStrategy feeStrategy) {
        this.feeStrategy = feeStrategy;
    }

    // Strategy can be swapped at runtime — key advantage over if-else chains
    public void setStrategy(FeeCalculationStrategy strategy) {
        this.feeStrategy = strategy;
    }

    public double executeTrade(String investorId, double amount, String portfolioType) {
        double fee = feeStrategy.calculateFee(amount, portfolioType);
        double total = amount + fee;
        System.out.printf("[%s] Trade: $%.2f + fee $%.2f (strategy: %s) = $%.2f%n",
                investorId, amount, fee, feeStrategy.strategyName(), total);
        AuditLogger.INSTANCE.log("Trade executed for " + investorId + " amount=" + amount);
        return total;
    }
}

// ═══════════════════════════════════════════════════════
// BUILDER PATTERN — Building Complex Investor Objects
// ═══════════════════════════════════════════════════════

// Use Builder when:
// 1. Object has many optional parameters (avoid telescoping constructors)
// 2. Construction is multi-step
// 3. Object should be immutable once built

class InvestorProfile {
    // All fields final — immutable after construction
    private final String investorId;
    private final String name;
    private final String portfolioType;
    private final double riskTolerance;       // optional
    private final double investmentLimit;     // optional
    private final boolean accreditedInvestor; // optional
    private final List<String> preferredAssets; // optional

    private InvestorProfile(Builder builder) {
        this.investorId       = builder.investorId;
        this.name             = builder.name;
        this.portfolioType    = builder.portfolioType;
        this.riskTolerance    = builder.riskTolerance;
        this.investmentLimit  = builder.investmentLimit;
        this.accreditedInvestor = builder.accreditedInvestor;
        this.preferredAssets  = Collections.unmodifiableList(builder.preferredAssets);
    }

    public static Builder builder(String investorId, String name) {
        return new Builder(investorId, name);
    }

    public static class Builder {
        private final String investorId;
        private final String name;
        private String portfolioType   = "BALANCED";
        private double riskTolerance   = 0.5;
        private double investmentLimit = 1_000_000.0;
        private boolean accreditedInvestor = false;
        private List<String> preferredAssets = new ArrayList<>();

        private Builder(String investorId, String name) {
            Objects.requireNonNull(investorId, "investorId required");
            Objects.requireNonNull(name, "name required");
            this.investorId = investorId;
            this.name = name;
        }

        public Builder portfolioType(String type)        { this.portfolioType = type; return this; }
        public Builder riskTolerance(double r)           { this.riskTolerance = r; return this; }
        public Builder investmentLimit(double limit)     { this.investmentLimit = limit; return this; }
        public Builder accredited(boolean accredited)    { this.accreditedInvestor = accredited; return this; }
        public Builder preferredAsset(String asset)      { this.preferredAssets.add(asset); return this; }

        public InvestorProfile build() {
            if (riskTolerance < 0 || riskTolerance > 1)
                throw new IllegalStateException("Risk tolerance must be 0.0–1.0");
            return new InvestorProfile(this);
        }
    }

    @Override public String toString() {
        return String.format("InvestorProfile[%s, %s, portfolio=%s, risk=%.1f, accredited=%b]",
                investorId, name, portfolioType, riskTolerance, accreditedInvestor);
    }
}

// ═══════════════════════════════════════════════════════
// OBSERVER PATTERN — Real-Time Portfolio Update Notifications
// ═══════════════════════════════════════════════════════

// Observer interface
interface PortfolioEventListener {
    void onPortfolioEvent(PortfolioEvent event);
}

record PortfolioEvent(String investorId, String eventType, double newValue, double change) {}

// Subject (Observable)
class PortfolioEventBus {
    private final Map<String, List<PortfolioEventListener>> listenersByEventType = new HashMap<>();

    public void subscribe(String eventType, PortfolioEventListener listener) {
        listenersByEventType.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }

    public void publish(PortfolioEvent event) {
        // Notify listeners registered for this specific event type
        List<PortfolioEventListener> specific = listenersByEventType.getOrDefault(
                event.eventType(), Collections.emptyList());
        // Also notify wildcard listeners
        List<PortfolioEventListener> wildcard = listenersByEventType.getOrDefault(
                "*", Collections.emptyList());

        for (PortfolioEventListener l : specific) l.onPortfolioEvent(event);
        for (PortfolioEventListener l : wildcard) l.onPortfolioEvent(event);
    }
}

class AlertingService implements PortfolioEventListener {
    @Override
    public void onPortfolioEvent(PortfolioEvent event) {
        if (Math.abs(event.change()) > 10_000) {
            System.out.printf("[ALERT] Large portfolio movement: %s changed by $%.2f%n",
                    event.investorId(), event.change());
        }
    }
}

class AnalyticsDashboard implements PortfolioEventListener {
    @Override
    public void onPortfolioEvent(PortfolioEvent event) {
        System.out.printf("[DASHBOARD] %s portfolio now $%.2f (Δ%.2f)%n",
                event.investorId(), event.newValue(), event.change());
    }
}

// ── DesignPatternsDemo.java ───────────────────────────────────────────────
public class DesignPatternsDemo {
    public static void main(String[] args) {
        System.out.println("=== Factory ===");
        AuthenticationHandler restHandler   = AuthHandlerFactory.create("REST");
        AuthenticationHandler grpcHandler   = AuthHandlerFactory.create("GRPC");
        AuthenticationHandler interacHandler = AuthHandlerFactory.create("INTERAC");

        AuthenticationRequest req = new AuthenticationRequest("INV-001", "eyJhbGciOiJSUzI1NiJ9", "REST");
        System.out.println(restHandler.handle(req));
        System.out.println(grpcHandler.handle(req));

        System.out.println("\n=== Singleton ===");
        TokenBlacklistCache cache1 = TokenBlacklistCache.getInstance();
        TokenBlacklistCache cache2 = TokenBlacklistCache.getInstance();
        cache1.blacklist("old-token-123");
        System.out.println("Same instance: " + (cache1 == cache2));         // true
        System.out.println("Blacklisted: " + cache2.isBlacklisted("old-token-123")); // true

        System.out.println("\n=== Strategy ===");
        TradeExecutor executor = new TradeExecutor(new EquityFeeStrategy());
        executor.executeTrade("INV-001", 50_000.0, "EQUITY");
        executor.setStrategy(new InstitutionalFeeStrategy());
        executor.executeTrade("INV-001", 50_000.0, "EQUITY");

        System.out.println("\n=== Builder ===");
        InvestorProfile profile = InvestorProfile.builder("INV-RBC-42", "Jane Smith")
                .portfolioType("EQUITY")
                .riskTolerance(0.8)
                .investmentLimit(5_000_000.0)
                .accredited(true)
                .preferredAsset("TSX:RY")
                .preferredAsset("TSX:TD")
                .build();
        System.out.println(profile);

        System.out.println("\n=== Observer ===");
        PortfolioEventBus bus = new PortfolioEventBus();
        bus.subscribe("REBALANCE", new AlertingService());
        bus.subscribe("*", new AnalyticsDashboard());  // wildcard: receives all events

        bus.publish(new PortfolioEvent("INV-001", "REBALANCE", 125_000.0, 15_000.0));
        bus.publish(new PortfolioEvent("INV-002", "DIVIDEND",   43_500.0,    250.0));
    }
}
```

---

## 1.6 Exception Handling

Robust exception handling is a first-class concern in financial systems. Unchecked exceptions
indicate programming errors (null, illegal argument); checked exceptions signal recoverable
conditions (network timeout, verification failure) that callers must handle. Custom exception
hierarchies let you propagate meaningful context up the call stack without losing the original
cause. `try-with-resources` is mandatory for anything implementing `AutoCloseable` — at RBC,
forgetting it on a JDBC `Connection` caused connection pool exhaustion under load.

**One-sentence interview answer:** "Checked exceptions model recoverable failures callers
must handle; unchecked exceptions are programming errors; I use custom hierarchies to carry
context and `try-with-resources` to guarantee resource cleanup."

**Most likely follow-ups:**
1. When should you catch `Exception` vs a specific type?
2. What happens if you throw inside a `finally` block?
3. How do you handle exceptions in a `CompletableFuture`?

```java
import java.util.logging.Logger;

// ── Custom exception hierarchy ────────────────────────────────────────────

// Base: all RBC investor errors extend this — callers can catch the base
// to handle any investor error, or catch specific subclasses for fine-grained handling
public class InvestorServiceException extends RuntimeException {
    private final String investorId;
    private final String errorCode;

    public InvestorServiceException(String message, String investorId, String errorCode) {
        super(message);
        this.investorId = investorId;
        this.errorCode  = errorCode;
    }

    public InvestorServiceException(String message, String investorId,
                                    String errorCode, Throwable cause) {
        super(message, cause);   // preserve original cause — critical for debugging
        this.investorId = investorId;
        this.errorCode  = errorCode;
    }

    public String getInvestorId() { return investorId; }
    public String getErrorCode()  { return errorCode; }
}

// Specific subclasses for different failure modes
public class InvestorVerificationException extends InvestorServiceException {
    private final int attemptNumber;

    public InvestorVerificationException(String investorId, int attempt, String reason) {
        super("Verification failed for investor " + investorId +
              " on attempt " + attempt + ": " + reason, investorId, "VERIFY_FAILED");
        this.attemptNumber = attempt;
    }

    public int getAttemptNumber() { return attemptNumber; }
}

public class InvestorNotFoundException extends InvestorServiceException {
    public InvestorNotFoundException(String investorId) {
        super("Investor not found: " + investorId, investorId, "NOT_FOUND");
    }
}

public class InsufficientFundsException extends InvestorServiceException {
    private final double requested;
    private final double available;

    public InsufficientFundsException(String investorId, double requested, double available) {
        super(String.format("Insufficient funds for %s: requested $%.2f, available $%.2f",
                investorId, requested, available), investorId, "INSUFFICIENT_FUNDS");
        this.requested = requested;
        this.available = available;
    }

    public double getShortfall() { return requested - available; }
}

// Checked exception — callers MUST handle or declare throws
// Use checked when the failure is recoverable and expected (e.g. external API timeout)
public class InteracApiException extends Exception {
    private final int httpStatus;

    public InteracApiException(String message, int httpStatus, Throwable cause) {
        super(message, cause);
        this.httpStatus = httpStatus;
    }

    public int getHttpStatus() { return httpStatus; }
}

// ── Exception handling in service code ────────────────────────────────────
public class InvestorVerificationService {
    private static final Logger log = Logger.getLogger(InvestorVerificationService.class.getName());
    private static final int MAX_ATTEMPTS = 3;

    // try-with-resources: DatabaseConnection implements AutoCloseable.
    // Connection is guaranteed to be closed even if an exception is thrown.
    // Without this, a thrown exception before connection.close() would leak
    // the connection — causing pool exhaustion under load (happened at RBC).
    public boolean verifyWithAudit(String investorId, String token)
            throws InteracApiException {

        try (DatabaseConnection conn = DatabaseConnection.acquire();       // auto-closed
             AuditWriter audit = new AuditWriter(investorId)) {           // also auto-closed

            audit.begin("VERIFY_START");

            for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
                try {
                    boolean result = callInteracApi(token);
                    conn.recordVerification(investorId, result);
                    audit.end("VERIFY_SUCCESS");
                    return result;

                } catch (InteracApiException e) {
                    log.warning("Interac API attempt " + attempt + " failed: " + e.getMessage());
                    if (attempt == MAX_ATTEMPTS || e.getHttpStatus() == 401) {
                        // 401 = bad credentials — retrying won't help, rethrow immediately
                        audit.end("VERIFY_FAILED_HTTP_" + e.getHttpStatus());
                        throw e;  // rethrow checked exception to caller
                    }
                    // else: transient failure, retry
                    try { Thread.sleep(100L * attempt); } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();  // restore interrupt status
                        throw new InteracApiException("Interrupted during retry", 503, ie);
                    }
                } catch (Exception e) {
                    // Unexpected exception — wrap in our domain exception to preserve context
                    throw new InvestorServiceException(
                            "Unexpected error during verification", investorId, "UNEXPECTED", e);
                }
            }
            return false;

        } catch (InvestorServiceException | InteracApiException e) {
            throw e;  // rethrow domain exceptions as-is
        } catch (Exception e) {
            // DatabaseConnection.acquire() or AuditWriter() failed
            log.severe("Resource acquisition failed: " + e.getMessage());
            throw new InvestorServiceException(
                    "Verification infrastructure failure", investorId, "INFRA_ERROR", e);
        }
    }

    private boolean callInteracApi(String token) throws InteracApiException {
        if (token == null) throw new InteracApiException("Token is null", 400, null);
        if (token.equals("expired")) throw new InteracApiException("Token expired", 401, null);
        return token.startsWith("eyJ");
    }
}

// Minimal stubs for compilation
class DatabaseConnection implements AutoCloseable {
    static DatabaseConnection acquire() { return new DatabaseConnection(); }
    void recordVerification(String id, boolean result) {}
    @Override public void close() { System.out.println("DB connection returned to pool"); }
}

class AuditWriter implements AutoCloseable {
    AuditWriter(String investorId) {}
    void begin(String event) { System.out.println("[AUDIT] " + event); }
    void end(String event)   { System.out.println("[AUDIT] " + event); }
    @Override public void close() { System.out.println("Audit writer flushed"); }
}
```

---

## 1.7 Java Memory + JVM

The JVM divides memory into the heap (object instances), stack (method frames, local variables),
metaspace (class metadata), and code cache (JIT-compiled code). In high-throughput Spring Boot
services, GC pauses directly impact P99 latency — the RBC auth service hitting 10k daily
authentications needed GC tuned to avoid stop-the-world pauses during market-open spikes.
G1GC is the production default; ZGC/Shenandoah target sub-millisecond pauses for latency-
sensitive workloads.

**One-sentence interview answer:** "The JVM heap is where objects live — GC reclaims unreachable
objects; G1GC balances throughput and pause time; ZGC targets sub-1ms pauses at cost of
slightly more CPU; tuning means right-sizing heap regions and controlling pause targets."

**Most likely follow-ups:**
1. What causes an `OutOfMemoryError: Java heap space` vs `PermGen/Metaspace`?
2. What is a memory leak in Java — how can it happen with GC?
3. How do you diagnose a memory leak in production?

```
┌──────────────────────────────────────────────────────────────────┐
│                         JVM MEMORY LAYOUT                        │
├──────────────────────────────────────────────────────────────────┤
│  HEAP (GC-managed)                                               │
│  ├─ Young Generation                                             │
│  │   ├─ Eden Space      ← new objects allocated here            │
│  │   └─ Survivor S0/S1  ← objects that survived minor GC        │
│  └─ Old Generation (Tenured) ← long-lived objects promoted here │
│                                                                  │
│  METASPACE (non-heap)  ← class metadata (replaced PermGen)      │
│  CODE CACHE (non-heap) ← JIT-compiled native code               │
│  STACK (per-thread)    ← frames, local vars, method refs        │
└──────────────────────────────────────────────────────────────────┘

GC TYPES — choosing the right one:
  G1GC    → Default Java 9+. Divides heap into regions.
            Good balance of throughput + pause time.
            JVM flag: -XX:+UseG1GC -XX:MaxGCPauseMillis=200
            Use: most production Spring Boot services

  ZGC     → Sub-millisecond pauses (concurrent marking + compaction).
            Slight CPU overhead. Java 15+ production-ready.
            JVM flag: -XX:+UseZGC
            Use: RBC real-time auth service, latency-sensitive APIs

  Shenandoah → Similar to ZGC (concurrent, low-pause). Red Hat.
            JVM flag: -XX:+UseShenandoahGC

  SerialGC    → Single-threaded. Only for small single-core JVMs.
  ParallelGC  → Max throughput, high pause. Good for batch jobs (Spark).

COMMON MEMORY LEAKS IN SPRING BOOT:
  1. Static collections accumulating entries:
     static Map<String, Cache> cache = new HashMap<>();
     // Never cleared → grows indefinitely in long-running service

  2. Unclosed resources (HttpClient, InputStream, JDBC Connection):
     // Forgot try-with-resources → objects referenced by GC roots
     // connection pool → connection → never GC'd

  3. ThreadLocal not removed:
     ThreadLocal<InvestorContext> ctx = new ThreadLocal<>();
     // Set in filter, never removed → reference held by thread pool thread
     // thread pool thread is a GC root → InvestorContext never collected
     // Fix: ctx.remove() in finally block

  4. Event listeners never deregistered (Observer pattern):
     eventBus.subscribe("REBALANCE", listener);
     // listener holds reference to large object
     // eventBus has GC root ref → listener → large object → never collected
     // Fix: always provide an unsubscribe mechanism

  5. Hibernate/JPA 1st-level cache in long-running sessions:
     // @Transactional on a method that loops millions of rows
     // EntityManager caches every loaded entity → heap explosion
     // Fix: entityManager.clear() periodically, or use StatelessSession

JVM TUNING FOR HIGH-THROUGHPUT AUTH SERVICE (RBC-style):

  -Xms2g -Xmx2g            # Set min == max to avoid heap resizing pauses
  -XX:+UseZGC               # Sub-ms pauses for auth API latency requirements
  -XX:MaxGCPauseMillis=10   # Target pause (ZGC best-effort)
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/rbc-auth/heapdump.hprof
  -Xlog:gc*:file=/var/log/rbc-auth/gc.log:time,uptime:filecount=5,filesize=20m

DIAGNOSING MEMORY LEAKS IN PRODUCTION:
  1. Enable GC logging (flags above)
  2. Monitor heap usage over time in CloudWatch / Prometheus
     → Sawtooth pattern = healthy (allocate, collect, repeat)
     → Upward trend = leak (allocation > collection)
  3. Take heap dump: jmap -dump:live,format=b,file=heap.hprof <pid>
     or: kill -3 <pid> (thread dump), or JVM flags above on OOM
  4. Analyze in Eclipse Memory Analyzer (MAT):
     → "Dominator tree" shows biggest retained objects
     → "Leak suspects" report identifies likely leak holders
  5. Common fix pattern: find the GC root holding the leaked object
     → static field? ThreadLocal? event listener? JPA session?
```

---
