`@Transactional` is pure AOP via proxies.

`@Transactional` does **zero work itself** as an annotation. It's a marker. The work is done by a proxy Spring creates around your bean — a proxy that intercepts method calls and wraps them in transaction management code. This is the first thing to understand cold.

```java
// What you think you get:
OrderService bean = new OrderService();

// What Spring actually gives you:
OrderService bean = new $Proxy42();  // a generated proxy wrapping OrderService
// Every call to bean.placeOrder() hits the PROXY first, not your class
```
### Step 1— Proxy intercepts the call
Your call never reaches the real object directly.
```java
// Your service class 
@Service 
public class OrderService { 
	@Transactional 
	public void placeOrder(Order order) { 
		// your actual method 
		orderRepo.save(order); 
		paymentService.charge(order); 
	} 
} 
// What Spring puts in the ApplicationContext instead: 
// OrderService$$EnhancerBySpringCGLIB$$abc123
// A generated subclass that OVERRIDES placeOrder() 
// When you call: orderService.placeOrder(order);
// You're calling the PROXY's placeOrder(), not yours
```
>[!swap]
>Spring swaps the real bean with a proxy in the ApplicationContext. Every injected reference to OrderService is actually a reference to the proxy. Your code never knows.

### Step 2 — TransactionInterceptor takes over
The proxy delegates to `TransactionInterceptor` (AOP around-advice)
```java
// What the generated proxy does (simplified): 
public class OrderService$$CGLIB extends OrderService {
	// Spring's AOP interceptor — the brain of @Transactional
	private TransactionInterceptor txInterceptor;
	@Override 
	public void placeOrder(Order order) {
		// 1. Hand off to TransactionInterceptor txInterceptor.invoke(this, placeOrderMethod, order);
		// TransactionInterceptor will call super.placeOrder(order)
		// at the right moment, wrapped in transaction logic
	}
}

// TransactionInterceptor is an AOP MethodInterceptor: 
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor { 
	public Object invoke(MethodInvocation invocation) { 
		// All transaction logic lives here 
		return invokeWithinTransaction(...); 
	} 
}
```
>[!imp]
>**TransactionAspectSupport.invokeWithinTransaction()** is the single most important method in Spring's transaction infrastructure. Everything flows through it.

### Step 3— Transaction begins
`PlatformTransactionManager` gets a connection and binds it to the thread
```java
// Inside invokeWithinTransaction() — simplified: 
// 1. Ask transaction manager to start a transaction 
TransactionStatus status = transactionManager.getTransaction(txDefinition); 
// txDefinition carries: propagation, isolation, timeout, readOnly 

// Inside JpaTransactionManager.getTransaction(): 
// 2. Get connection from HikariCP pool 
Connection conn = dataSource.getConnection();

// 3. Disable autocommit — this IS what "begin transaction" means in 
JDBC conn.setAutoCommit(false); 

// 4. Bind connection to CURRENT THREAD via ThreadLocal 
TransactionSynchronizationManager .bindResource(dataSource, connectionHolder);
// ↑ This is the KEY mechanism. The connection lives in a ThreadLocal. 
// Every repo/DAO on this thread gets the SAME connection.
// That's how they all participate in the SAME transaction.
```
>[!ThreadLocal]
>**ThreadLocal is the binding glue.** The connection is stored per-thread. Every JPA/JDBC call on this thread automatically picks up the same connection — that's how OrderRepo and PaymentService share the same transaction without you wiring them together.

### Step 4— Method executes
Your actual method runs — all repos share the same ThreadLocal connection
```java
// Your method finally executes: public void placeOrder(Order order) { orderRepo.save(order); // ↑ orderRepo asks TransactionSynchronizationManager: // "Is there a connection on my thread?" // YES → reuses it. Same transaction. paymentService.charge(order); // ↑ paymentService.charge() is also @Transactional // PROPAGATION.REQUIRED (default) → "already in a tx? join it." // Uses the SAME ThreadLocal connection. Still same transaction. inventoryService.reserve(order); // Same. All three on the same connection = same transaction. // If ANY throws RuntimeException, ALL roll back. } // The EntityManager also uses the ThreadLocal connection: // EntityManagerFactoryUtils.getTransactionalEntityManager(emf) // returns the same EM bound to this thread
```
>[!note]
>This is why **@Transactional at the service level** propagates to all your repositories automatically — they all read from the same ThreadLocal to find the active connection/EntityManager.

### Step 5A— Success path
Method returns normally → commit → release connection
```java
// Back in invokeWithinTransaction() after method returns: try { Object result = invocation.proceed(); // your method ran OK // Commit the transaction transactionManager.commit(status); // → conn.commit() (actual SQL commit sent to DB) // → conn.setAutoCommit(true) // → TransactionSynchronizationManager.unbindResource() // → conn returned to HikariCP pool return result; } catch (RuntimeException | Error ex) { // ROLLBACK path — see next step completeTransactionAfterThrowing(status, ex); throw ex; }
```
>[!Commit]
>Commit flushes the Hibernate first-level cache (pending INSERTs/UPDATEs), sends them to the DB in one batch, commits, and returns the connection to the pool. The whole unit of work is atomic.

### Step 5B— Exception path
`RuntimeException` thrown → rollback (checked exception → commit by default).
```java
// RuntimeException or Error → ROLLBACK } catch (RuntimeException | Error ex) { transactionManager.rollback(status); // → conn.rollback() (DB undoes everything since BEGIN) // → connection released to pool throw ex; // exception propagates to caller } // ⚠️ Checked exception → COMMIT by default (historical EJB legacy) } catch (CheckedException ex) { transactionManager.commit(status); // ← surprises many devs! throw ex; } // Override rollback rules explicitly: @Transactional(rollbackFor = Exception.class) // rollback on ANY exception @Transactional(noRollbackFor = StaleDataException.class) // don't rollback on this // Or mark rollback manually inside the method: TransactionAspectSupport.currentTransactionStatus() .setRollbackOnly(); // forces rollback even without exception
```
>[!Key interview point:]
>checked exceptions do NOT trigger rollback by default. This catches many developers off guard. Always use `rollbackFor = Exception.class` if you throw checked exceptions from @Transactional methods.

## Why `@Transactional` Doesn't Work on Private Methods

```java
@Service
public class OrderService {
    @Transactional  // ← THIS HAS NO EFFECT
    private Order createOrderInternal(CreateOrderDto dto) {
        // No transaction here!
        return orderRepository.save(new Order(dto));
    }
}
```

**Two reasons:**
```less
Reason 1 — CGLIB cannot override private methods: 
	CGLIB creates a SUBCLASS to intercept methods. 
	Java rule: subclasses CANNOT override private methods.
	Therefore: CGLIB cannot intercept private @Transactional methods. 
	Therefore: the transaction interceptor never runs. 

Reason 2 — Even with JDK proxy, interface methods must be public: 
	JDK proxy requires the method to be declared in the interface. 
	Private methods can't be in interfaces. 
	Therefore: same result — no transaction. 

The Spring warning: 
	Spring 5.3+ logs a warning when it detects @Transactional on a private method: 
	"Transactional annotation on private method: OrderService.createOrderInternal"
```
