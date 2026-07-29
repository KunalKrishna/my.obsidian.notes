# Comprehensive OOP in Java — Complete Study Notes

### Who · What · When · Where · Why · How

---

## Preface — Recommended Study Order

The 4 pillars of OOP are **not independent** — each one builds on the previous. Studying them out of order creates confusion. The recommended order is:

```
1. Encapsulation  →  2. Abstraction  →  3. Inheritance  →  4. Polymorphism
   (Protect data)      (Design contracts)  (Reuse structure)  (Runtime flexibility)
```

**Why this order?**

|Step|Pillar|What you learn|Why it must come here|
|---|---|---|---|
|1|Encapsulation|How to protect internal data|Foundation — everything else builds on well-structured classes|
|2|Abstraction|How to design clean public contracts|You need encapsulated classes before you can abstract them|
|3|Inheritance|How to reuse and extend structures|You need abstract contracts to know what to inherit|
|4|Polymorphism|How one reference adapts to many forms|Requires inheritance — impossible to understand without it|

**Domain used throughout these notes:** A **Banking System** (BankAccount, SavingsAccount, CurrentAccount, LoanAccount, etc.). This single domain will illustrate all 4 pillars. Where the banking domain would obscure a concept, a supplementary example is added clearly marked.

---

---

# PILLAR 1 — ENCAPSULATION

---

## WHO

**Encapsulation** is used by **every Java developer** at every level. It is the most foundational pillar — without it, you don't have OOP, you have a collection of disorganized variables.

## WHAT

Encapsulation means **bundling data (fields) and the methods that operate on that data inside a single class**, and **restricting direct external access** to the internal data using access modifiers (`private`, `protected`, `public`).

It is the principle of **data hiding + controlled access**.

## WHEN

Use encapsulation:

- Always — every class you write should encapsulate its fields
- Especially when a field has **validation rules** (balance can't be negative)
- When internal representation may change without affecting external users
- When you want to make a class **immutable** (all fields private, no setters)

## WHERE

- Every class definition in Java
- JavaBeans / POJOs (Plain Old Java Objects)
- Entity classes (mapped to database tables)
- Domain model classes

## WHY

- **Prevents invalid state:** No one can set `balance = -99999` directly
- **Hides implementation details:** Callers don't need to know how balance is stored
- **Enables validation:** Setters enforce business rules
- **Allows safe refactoring:** Change internals without breaking external code
- **Thread safety:** Controlled access makes synchronization easier

## HOW

### Access Modifiers (The Mechanism of Encapsulation)

```
private   →  Accessible only within the same class
(default) →  Accessible within the same package (no keyword)
protected →  Accessible within same package + subclasses
public    →  Accessible from anywhere
```

---

### Basic Encapsulation — BankAccount

```java
public class BankAccount {

    // ── ENCAPSULATED FIELDS ─────────────────────────────────────────────
    private String accountNumber;       // No external direct access
    private String accountHolderName;
    private double balance;             // Core — must never go negative (for savings)
    private String pin;                 // Sensitive — never expose directly

    // ── CONSTRUCTOR ─────────────────────────────────────────────────────
    public BankAccount(String accountNumber, String holderName,
                       double initialDeposit, String pin) {
        if (initialDeposit < 0)
            throw new IllegalArgumentException("Initial deposit cannot be negative.");
        this.accountNumber = accountNumber;
        this.accountHolderName = holderName;
        this.balance = initialDeposit;
        this.pin = pin;
    }

    // ── GETTERS (Read-only access) ───────────────────────────────────────
    public String getAccountNumber() { return accountNumber; }
    public String getAccountHolderName() { return accountHolderName; }
    public double getBalance() { return balance; }

    // ── NO getter for PIN — sensitive data never exposed ─────────────────

    // ── SETTER (With validation) ─────────────────────────────────────────
    public void setAccountHolderName(String name) {
        if (name == null || name.trim().isEmpty())
            throw new IllegalArgumentException("Name cannot be empty.");
        this.accountHolderName = name;
    }

    // ── BUSINESS METHODS (Controlled state changes) ──────────────────────
    public void deposit(double amount) {
        if (amount <= 0)
            throw new IllegalArgumentException("Deposit amount must be positive.");
        this.balance += amount;
        System.out.printf("Deposited ₹%.2f | New Balance: ₹%.2f%n", amount, balance);
    }

    public void withdraw(double amount) {
        if (amount <= 0)
            throw new IllegalArgumentException("Withdrawal amount must be positive.");
        if (amount > balance)
            throw new IllegalStateException("Insufficient funds.");
        this.balance -= amount;
        System.out.printf("Withdrawn ₹%.2f | New Balance: ₹%.2f%n", amount, balance);
    }

    public boolean validatePin(String inputPin) {
        return this.pin.equals(inputPin);   // PIN checked internally, never returned
    }

    @Override
    public String toString() {
        return String.format("[%s] %s — Balance: ₹%.2f",
            accountNumber, accountHolderName, balance);
    }
}
```

```java
// ── USAGE ────────────────────────────────────────────────────────────────
BankAccount acc = new BankAccount("ACC001", "Alice", 5000.00, "1234");

// acc.balance = -99999;            // ❌ COMPILER ERROR — field is private
// acc.pin = "0000";                // ❌ COMPILER ERROR — field is private

acc.deposit(2000);                  // ✅ → Deposited ₹2000.00 | New Balance: ₹7000.00
acc.withdraw(1000);                 // ✅ → Withdrawn ₹1000.00 | New Balance: ₹6000.00
acc.withdraw(9999);                 // ✅ → IllegalStateException: Insufficient funds

System.out.println(acc.getBalance());           // ✅ Read-only access
System.out.println(acc.validatePin("1234"));    // ✅ → true (PIN never leaves the object)
System.out.println(acc.validatePin("0000"));    // ✅ → false
```

---

### Immutable Class — Maximum Encapsulation

An immutable class has all fields `private final` and **no setters**. Once created, its state never changes. `String` in Java is immutable.

```java
public final class ImmutableAccount {           // final — no subclassing allowed

    private final String accountNumber;
    private final String holderName;
    private final double balance;

    public ImmutableAccount(String accountNumber, String holderName, double balance) {
        this.accountNumber = accountNumber;
        this.holderName = holderName;
        this.balance = balance;
    }

    public String getAccountNumber() { return accountNumber; }
    public String getHolderName()    { return holderName; }
    public double getBalance()       { return balance; }

    // No setters — state cannot change after construction

    // To "change" state, return a NEW object
    public ImmutableAccount withBalance(double newBalance) {
        return new ImmutableAccount(accountNumber, holderName, newBalance);
    }
}
```

```java
ImmutableAccount acc = new ImmutableAccount("ACC002", "Bob", 3000);
ImmutableAccount updated = acc.withBalance(5000);   // Original unchanged!

System.out.println(acc.getBalance());       // → 3000.0
System.out.println(updated.getBalance());   // → 5000.0
```

---

### Interview Questions — Encapsulation

**Q1. What is encapsulation and why is it important?**

> Encapsulation is the bundling of data and the methods that operate on it within a single class, with restricted access via access modifiers. It prevents invalid state, enforces business rules through controlled access points, and allows internal implementation to change without affecting callers.

**Q2. What is the difference between data hiding and encapsulation?**

> Data hiding is one aspect of encapsulation — making fields `private`. Encapsulation is the broader principle that includes both hiding data AND providing controlled access through public methods (getters, setters, business methods).

**Q3. Can we achieve encapsulation without getters and setters?**

> Yes. If a class's fields never need to be accessed externally (e.g., fully self-contained logic), encapsulation is complete without any getters/setters. Getters/setters are just the most common mechanism of controlled access, not a requirement.

**Q4. Is a class with all public fields encapsulated?**

> No. Public fields offer zero protection — any code can read or write them freely, bypassing any validation. This violates encapsulation.

**Q5. What is the output?**

```java
class Account {
    private int balance = 100;
    public int getBalance() { return balance; }
    public void setBalance(int b) { balance = b > 0 ? b : balance; }
}
Account a = new Account();
a.setBalance(-50);
System.out.println(a.getBalance());
```

> **Output: 100** — The setter rejects negative values, leaving balance unchanged.

---

---

# PILLAR 2 — ABSTRACTION

---

## WHO

Used by architects and senior developers designing **APIs, frameworks, and libraries**. Also used by any developer building systems where the **what** must be separated from the **how**.

## WHAT

Abstraction means **exposing only what is necessary** and **hiding how it works internally**. You define a contract (the interface or abstract class) without specifying implementation. Consumers code against the contract, not the concrete detail.

**Two mechanisms in Java:**

- `abstract class` — Partial abstraction (can have concrete methods + abstract methods)
- `interface` — Full abstraction (pure contract; all methods are implicitly abstract, except `default` and `static`)

## WHEN

- When you want **multiple implementations** of the same concept (e.g., multiple account types all being "accounts")
- When implementation details **may change** and callers shouldn't be affected
- When designing a **framework or API** others will implement
- When applying design patterns (Strategy, Template Method, Factory, etc.)

## WHERE

- API design (`List`, `Map`, `Comparable` in Java standard library)
- Service layers (e.g., `PaymentService` interface with `StripeService`, `PaypalService`)
- Repository pattern (database access abstracted behind an interface)
- Plugin architectures

## WHY

- **Decoupling:** Code against abstractions, not concrete types
- **Flexibility:** Swap implementations without changing callers
- **Testability:** Mock abstractions in unit tests easily
- **Design clarity:** Forces you to think about what a thing _does_, not how
- **Open/Closed Principle:** Open for extension, closed for modification

## HOW

### `interface` — Pure Contract

```java
// Defines WHAT a bank account does — not HOW
public interface Account {
    void deposit(double amount);
    void withdraw(double amount);
    double getBalance();
    String getAccountNumber();
    void printStatement();          // Every account must support a statement
}
```

```java
// Another interface — Accounts that earn interest
public interface InterestBearing {
    double calculateInterest();
    void applyInterest();
}
```

---

### `abstract class` — Partial Abstraction (Template)

```java
// Abstract class — provides common structure, defers specifics to children
public abstract class AbstractAccount implements Account {

    protected String accountNumber;
    protected String holderName;
    protected double balance;

    // Concrete — shared implementation (same for all accounts)
    public AbstractAccount(String accountNumber, String holderName, double initialDeposit) {
        this.accountNumber = accountNumber;
        this.holderName = holderName;
        this.balance = initialDeposit;
    }

    @Override
    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Invalid deposit amount.");
        balance += amount;
    }

    @Override
    public double getBalance() { return balance; }

    @Override
    public String getAccountNumber() { return accountNumber; }

    @Override
    public void printStatement() {
        System.out.printf("Account: %s | Holder: %s | Balance: ₹%.2f%n",
            accountNumber, holderName, balance);
    }

    // Abstract — each account type must define its own withdrawal rules
    @Override
    public abstract void withdraw(double amount);

    // Abstract — each account type has a different account type label
    public abstract String getAccountType();
}
```

---

### Concrete Implementations

```java
// Savings Account — has minimum balance rule
public class SavingsAccount extends AbstractAccount implements InterestBearing {

    private static final double MIN_BALANCE = 1000.0;
    private static final double INTEREST_RATE = 0.04;   // 4% p.a.

    public SavingsAccount(String accountNumber, String holderName, double deposit) {
        super(accountNumber, holderName, deposit);
    }

    @Override
    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Invalid amount.");
        if (balance - amount < MIN_BALANCE)
            throw new IllegalStateException(
                "Cannot withdraw. Must maintain minimum balance of ₹" + MIN_BALANCE);
        balance -= amount;
    }

    @Override
    public double calculateInterest() {
        return balance * INTEREST_RATE;
    }

    @Override
    public void applyInterest() {
        double interest = calculateInterest();
        balance += interest;
        System.out.printf("Interest ₹%.2f applied. New balance: ₹%.2f%n", interest, balance);
    }

    @Override
    public String getAccountType() { return "Savings Account"; }
}


// Current Account — overdraft allowed up to a limit
public class CurrentAccount extends AbstractAccount {

    private double overdraftLimit;

    public CurrentAccount(String accountNumber, String holderName,
                          double deposit, double overdraftLimit) {
        super(accountNumber, holderName, deposit);
        this.overdraftLimit = overdraftLimit;
    }

    @Override
    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Invalid amount.");
        if (balance - amount < -overdraftLimit)
            throw new IllegalStateException("Overdraft limit exceeded.");
        balance -= amount;
    }

    @Override
    public String getAccountType() { return "Current Account"; }
}


// Fixed Deposit Account — no withdrawals before maturity
public class FixedDepositAccount extends AbstractAccount implements InterestBearing {

    private int tenureMonths;
    private static final double FD_RATE = 0.065;   // 6.5% p.a.
    private boolean matured = false;

    public FixedDepositAccount(String accountNumber, String holderName,
                                double deposit, int tenureMonths) {
        super(accountNumber, holderName, deposit);
        this.tenureMonths = tenureMonths;
    }

    @Override
    public void withdraw(double amount) {
        if (!matured)
            throw new IllegalStateException("Cannot withdraw before FD maturity.");
        balance -= amount;
    }

    public void mature() { this.matured = true; }   // Called at maturity date

    @Override
    public double calculateInterest() {
        return balance * FD_RATE * (tenureMonths / 12.0);
    }

    @Override
    public void applyInterest() {
        double interest = calculateInterest();
        balance += interest;
        System.out.printf("FD matured. Interest ₹%.2f credited. Total: ₹%.2f%n",
            interest, balance);
    }

    @Override
    public String getAccountType() { return "Fixed Deposit"; }
}
```

```java
// ── USAGE: Code against the abstraction, not concrete types ──────────────

Account acc1 = new SavingsAccount("SAV001", "Alice", 5000);
Account acc2 = new CurrentAccount("CUR001", "Bob", 10000, 5000);
Account acc3 = new FixedDepositAccount("FD001", "Carol", 50000, 12);

// Same method call — different behaviour (abstraction in action)
acc1.deposit(2000);
acc2.deposit(3000);
acc3.deposit(0);    // → IllegalArgumentException

acc1.printStatement();
acc2.printStatement();

// Interest operations — only for InterestBearing accounts
if (acc1 instanceof InterestBearing ib) {
    ib.applyInterest();
}

// Type-specific behaviour
SavingsAccount sav = (SavingsAccount) acc1;
System.out.println("Interest: ₹" + sav.calculateInterest());
```

---

### `interface` vs `abstract class` — Decision Guide

||`interface`|`abstract class`|
|---|---|---|
|**Multiple implementation**|✅ A class can implement many|❌ Only one class can extend|
|**Constructor**|❌ Not allowed|✅ Allowed|
|**Instance fields**|❌ Not allowed (only constants)|✅ Allowed|
|**Default methods**|✅ Java 8+|✅ Always|
|**Use when**|Defining a capability/contract|Sharing common code + enforcing structure|
|**IS-A vs CAN-DO**|CAN-DO (`Flyable`, `Serializable`)|IS-A (`AbstractAccount`)|

---

### Interview Questions — Abstraction

**Q1. What is the difference between abstraction and encapsulation?**

> Encapsulation **protects data** — it hides the internal state using access modifiers. Abstraction **hides complexity** — it exposes only the necessary interface, hiding the implementation. Encapsulation is about _who can touch the data_; abstraction is about _what you need to know to use the class_.

**Q2. Can we instantiate an abstract class?**

> No. An abstract class is incomplete — it may have abstract methods without a body. The JVM cannot create an object of an incomplete class. You must subclass it and provide implementations for all abstract methods before instantiating.

**Q3. Can an interface have a constructor?**

> No. Interfaces cannot have constructors because they cannot be instantiated. They define contracts that other classes implement.

**Q4. What is a default method in an interface and why was it introduced?**

> A `default` method (Java 8+) is a method in an interface that has a concrete implementation. It was introduced to allow adding new methods to existing interfaces without breaking all implementing classes — enabling backward-compatible API evolution.

**Q5. An abstract class has 5 abstract methods. A concrete subclass implements only 3. Is this valid?**

> No — the subclass must also be declared `abstract`. A concrete class must implement **all** abstract methods from its parent abstract class and interfaces.

**Q6. What is the output?**

```java
interface Printable { default void print() { System.out.println("Interface"); } }
abstract class Base implements Printable {
    public void print() { System.out.println("Abstract Class"); }
}
class Child extends Base {}

Printable p = new Child();
p.print();
```

> **Output: Abstract Class** — `Base` overrides `Printable`'s default method. `Child` inherits `Base`'s version. Dynamic dispatch resolves to `Base::print()`.

---

---

# PILLAR 3 — INHERITANCE

---

## WHO

Used by every Java developer. Especially important in framework design, where base classes define common behavior and developers extend them.

## WHAT

Inheritance is the mechanism where a **child class (subclass)** acquires the fields, methods, and behavior of a **parent class (superclass)**. It models an **IS-A relationship**.

```
SavingsAccount IS-A AbstractAccount
CurrentAccount IS-A AbstractAccount
AbstractAccount IS-A Account (interface)
```

## WHEN

Use inheritance when:

- A clear IS-A relationship exists (not just "has similar code")
- You want **code reuse** without copy-paste
- You need to **extend** an existing class with additional behavior
- You're applying the **Template Method** design pattern

**Avoid inheritance when:**

- The relationship is HAS-A (use composition instead)
- You're inheriting just to reuse code with no IS-A relationship ("inheritance for convenience")

## WHERE

- Class hierarchies in domain models
- Framework extensions (`extends HttpServlet`, `extends JFrame`)
- Exception class hierarchies (`extends RuntimeException`)
- Test base classes

## WHY

- **Code reuse:** Common code in parent, specific code in child
- **Extensibility:** Add new types without modifying existing code
- **Method overriding:** Customize parent behavior in child
- **Polymorphic behavior:** Child objects can be used where parent is expected
- **Hierarchical classification:** Models real-world "kind of" relationships

## HOW

### Single Inheritance — Banking Hierarchy

```java
// ── LEVEL 1: Already defined above as AbstractAccount ──────────────────

// ── LEVEL 2: PremiumSavingsAccount extends SavingsAccount ────────────
public class PremiumSavingsAccount extends SavingsAccount {

    private double cashbackRate;
    private String relationshipManager;

    public PremiumSavingsAccount(String accountNumber, String holderName,
                                  double deposit, double cashbackRate,
                                  String manager) {
        super(accountNumber, holderName, deposit);  // Calls SavingsAccount constructor
        this.cashbackRate = cashbackRate;
        this.relationshipManager = manager;
    }

    // Additional premium feature
    public double getCashback(double transactionAmount) {
        return transactionAmount * cashbackRate;
    }

    // Extends parent's printStatement
    @Override
    public void printStatement() {
        super.printStatement();     // Reuse parent output
        System.out.printf("  RM: %s | Cashback Rate: %.1f%%%n",
            relationshipManager, cashbackRate * 100);
    }

    @Override
    public String getAccountType() { return "Premium Savings Account"; }
}
```

```java
PremiumSavingsAccount premium = new PremiumSavingsAccount(
    "PREM001", "Diana", 100000, 0.02, "Raj Kumar");

premium.deposit(5000);          // ✅ Inherited from AbstractAccount
premium.applyInterest();        // ✅ Inherited from SavingsAccount
premium.printStatement();       // ✅ Overridden — calls super first
System.out.println("Cashback: ₹" + premium.getCashback(10000)); // ₹200.0
```

---

### Multi-Level Inheritance

```java
AbstractAccount          // Level 1: General account rules
    └── SavingsAccount   // Level 2: Savings-specific rules (min balance, interest)
          └── PremiumSavingsAccount  // Level 3: Premium features on top of savings
```

```java
// Every level retains access to parent's protected members
PremiumSavingsAccount acc = new PremiumSavingsAccount(...);
acc.balance;         // ❌ private in AbstractAccount — not accessible
acc.getBalance();    // ✅ public getter — accessible
```

---

### The `super` Keyword

```java
public class SavingsAccount extends AbstractAccount {

    private static final double MIN_BALANCE = 1000.0;

    public SavingsAccount(String num, String name, double deposit) {
        super(num, name, deposit);   // 1. Call parent constructor (must be first line)
    }

    @Override
    public void printStatement() {
        super.printStatement();      // 2. Call parent method version
        System.out.println("  Min Balance Required: ₹" + MIN_BALANCE);
    }

    public void showParentBalance() {
        System.out.println(super.balance);   // 3. Access parent's protected field
        // (though using getter is preferred)
    }
}
```

---

### Constructor Chaining in Inheritance

```java
// When new PremiumSavingsAccount(...) is called:
// 1. PremiumSavingsAccount() calls super() → SavingsAccount()
// 2. SavingsAccount() calls super()        → AbstractAccount()
// 3. AbstractAccount() initializes fields
// 4. SavingsAccount() continues
// 5. PremiumSavingsAccount() continues

// Output order:
// "AbstractAccount created"
// "SavingsAccount initialized"
// "PremiumSavingsAccount initialized"
```

---

### `final` — Stopping Inheritance

```java
public final class FixedInterestAccount extends SavingsAccount {
    // This class can NOT be extended further — final seals the hierarchy
}

// Any attempt to extend FixedInterestAccount gives compiler error:
// public class Child extends FixedInterestAccount { }  ❌
```

```java
public class SavingsAccount extends AbstractAccount {
    // This specific method cannot be overridden in any subclass
    public final void freezeAccount() {
        System.out.println("Account frozen — cannot be overridden.");
    }
}
```

---

### Inheritance vs Composition

Inheritance is powerful but often overused. **Prefer composition when IS-A is not true.**

```java
// ❌ BAD — Engine IS-NOT-A Car
public class Car extends Engine { ... }

// ✅ GOOD — Car HAS-A Engine (composition)
public class Car {
    private Engine engine;      // Composition: HAS-A
    private Account linkedAccount;

    public Car(Engine engine) {
        this.engine = engine;
    }
}
```

For banking:

```java
// ❌ BAD — Customer IS-NOT-A BankAccount
public class Customer extends BankAccount { ... }

// ✅ GOOD — Customer HAS-A BankAccount (composition)
public class Customer {
    private String customerId;
    private String name;
    private List<Account> accounts;   // HAS-A (one customer, many accounts)

    public void addAccount(Account account) {
        accounts.add(account);
    }
}
```

---

### Interview Questions — Inheritance

**Q1. What is the difference between IS-A and HAS-A relationships?**

> IS-A is inheritance (`SavingsAccount extends AbstractAccount`). HAS-A is composition (`Customer` has a list of `Account`). Use IS-A only when the child truly is a specialized version of the parent. Use HAS-A when one object uses/contains another.

**Q2. Why does Java not support multiple inheritance of classes?**

> To avoid the **Diamond Problem** — if class C extends A and B, and both A and B have a method `print()`, the JVM can't determine which one C should inherit. Java solves this by allowing multiple interface implementation (interfaces can have `default` methods, and Java has explicit resolution rules for conflicts).

**Q3. What happens if a child class does not call `super()` explicitly?**

> The compiler automatically inserts a call to the no-arg `super()` as the first statement. If the parent has **no no-arg constructor**, the child class **must** explicitly call `super(args)` or a compiler error occurs.

**Q4. Can a constructor be inherited?**

> No. Constructors are not inherited. A child class can _call_ a parent constructor using `super()`, but it must define its own constructors.

**Q5. What is the output?**

```java
class A {
    A() { System.out.println("A"); }
}
class B extends A {
    B() { System.out.println("B"); }
}
class C extends B {
    C() { System.out.println("C"); }
}
new C();
```

> **Output: A → B → C** — Constructor chaining: `super()` is implicitly called at each level, so A's constructor runs first, then B's, then C's.

**Q6. What is method hiding?**

> Method hiding occurs when a child class defines a `static` method with the same signature as a `static` method in the parent. Unlike overriding, it is resolved at compile time based on the reference type, not the actual object. It is also called method shadowing.

---

---

# PILLAR 4 — POLYMORPHISM

---

## WHO

Every developer, but especially those designing systems that need to work with **multiple types uniformly** — collections, frameworks, design patterns.

## WHAT

Polymorphism (Greek: _poly_ = many, _morphe_ = form) means **one interface, many implementations**. The same method call behaves differently based on the actual object type at runtime (or the parameter type at compile time).

**Two types:**

- **Compile-time Polymorphism** (Static): Method Overloading — resolved by compiler
- **Runtime Polymorphism** (Dynamic): Method Overriding — resolved by JVM at runtime via Dynamic Dispatch

## WHEN

- When you have a **collection of different types** sharing a common parent
- When you want to **add new types** without changing existing code
- When implementing design patterns: Strategy, Command, Observer, Factory
- When writing **generic algorithms** that work on abstract types

## WHERE

- Collections of mixed subtypes (`List<Account>`)
- Service layers (process any `Account` type)
- Plugin systems (any implementation of an interface)
- Event handling systems

## WHY

- **Extensibility:** Add new account types without rewriting existing processing logic
- **Flexibility:** Switch implementations at runtime
- **Maintainability:** One processing loop handles all types
- **Open/Closed Principle:** Code is open for extension, closed for modification
- **Eliminates large if/else and switch chains**

## HOW

### Type 1 — Compile-Time Polymorphism (Method Overloading)

```java
public class TransactionProcessor {

    // Same method name — different parameter types/counts
    // Compiler picks the right version based on arguments at compile time

    // Overload 1: Simple amount transfer
    public void transfer(double amount) {
        System.out.printf("Transferring ₹%.2f to default account%n", amount);
    }

    // Overload 2: Transfer to specific account
    public void transfer(double amount, String targetAccountNumber) {
        System.out.printf("Transferring ₹%.2f to %s%n", amount, targetAccountNumber);
    }

    // Overload 3: Transfer with description
    public void transfer(double amount, String targetAccount, String description) {
        System.out.printf("Transferring ₹%.2f to %s [%s]%n",
            amount, targetAccount, description);
    }

    // Overload 4: Transfer between two Account objects
    public void transfer(Account from, Account to, double amount) {
        from.withdraw(amount);
        to.deposit(amount);
        System.out.printf("Transferred ₹%.2f from %s to %s%n",
            amount, from.getAccountNumber(), to.getAccountNumber());
    }
}
```

```java
TransactionProcessor processor = new TransactionProcessor();
Account savings  = new SavingsAccount("SAV001", "Alice", 10000);
Account current  = new CurrentAccount("CUR001", "Bob", 5000, 2000);

processor.transfer(500);                            // Overload 1
processor.transfer(1000, "CUR001");                 // Overload 2
processor.transfer(2000, "CUR001", "Rent");         // Overload 3
processor.transfer(savings, current, 3000);         // Overload 4 — actual objects
```

---

### Type 2 — Runtime Polymorphism (Method Overriding + Dynamic Dispatch)

```java
// The real power — one reference, many behaviours

List<Account> accounts = new ArrayList<>();
accounts.add(new SavingsAccount("SAV001", "Alice", 15000));
accounts.add(new CurrentAccount("CUR001", "Bob", 20000, 5000));
accounts.add(new FixedDepositAccount("FD001", "Carol", 50000, 12));
accounts.add(new PremiumSavingsAccount("PREM001", "Diana", 100000, 0.02, "Raj"));

// ONE loop — handles ALL account types
// JVM decides which withdraw/printStatement to call at RUNTIME
System.out.println("===== Account Statements =====");
for (Account acc : accounts) {
    acc.printStatement();       // Different output per type — dynamic dispatch
}

// Process interest for eligible accounts
System.out.println("\n===== Applying Interest =====");
for (Account acc : accounts) {
    if (acc instanceof InterestBearing ib) {    // Java 16+ pattern matching
        ib.applyInterest();
    }
}
```

---

### Dynamic Dispatch Internals — The vtable

```
When JVM executes: acc.printStatement() where acc holds a SavingsAccount:

   SavingsAccount.vtable
   ┌─────────────────────────────────────────────┐
   │ deposit()        → AbstractAccount::deposit  │  ← inherited
   │ withdraw()       → SavingsAccount::withdraw  │  ← overridden
   │ getBalance()     → AbstractAccount::getBalance│  ← inherited
   │ printStatement() → SavingsAccount::printStatement│← overridden
   └─────────────────────────────────────────────┘

   JVM: object is SavingsAccount → look up vtable → call SavingsAccount::printStatement()
```

---

### Polymorphism Eliminating if/else Chains

```java
// ❌ BAD — Non-polymorphic approach (breaks every time a new type is added)
public void processAccount(Object account) {
    if (account instanceof SavingsAccount) {
        ((SavingsAccount) account).withdraw(100);
        // savings logic...
    } else if (account instanceof CurrentAccount) {
        ((CurrentAccount) account).withdraw(100);
        // current logic...
    } else if (account instanceof FixedDepositAccount) {
        // fd logic...
    }
    // Add new type? Must modify this method. Violates Open/Closed Principle.
}

// ✅ GOOD — Polymorphic approach
public void processAccount(Account account) {
    account.withdraw(100);  // Correct version auto-selected by JVM
    account.printStatement();
    // Adding new account type? Just create the class. This method needs NO change.
}
```

---

### Polymorphism with Interface References

```java
// Same object — three different reference types
SavingsAccount sav = new SavingsAccount("SAV001", "Alice", 20000);

Account         accRef = sav;   // Via Account interface
InterestBearing ibRef  = sav;   // Via InterestBearing interface

// Same underlying object — different capabilities exposed
accRef.deposit(1000);           // Account capability
accRef.withdraw(500);           // Account capability
ibRef.applyInterest();          // InterestBearing capability
// ibRef.deposit(...)           // ❌ Not available via InterestBearing reference

System.out.println(accRef == ibRef);    // → true — SAME object!
```

---

### Practical Polymorphism — Monthly Batch Processing

```java
public class BankingBatchProcessor {

    private List<Account> allAccounts;

    public BankingBatchProcessor(List<Account> accounts) {
        this.allAccounts = accounts;
    }

    // Process monthly operations — works for ALL account types
    public void runMonthlyBatch() {
        System.out.println("===== Monthly Batch Processing =====");

        double totalDeposits    = 0;
        double totalInterestPaid = 0;

        for (Account acc : allAccounts) {
            // 1. Print statement — polymorphic (each type prints differently)
            acc.printStatement();

            // 2. Apply interest — only for eligible accounts
            if (acc instanceof InterestBearing ib) {
                double interest = ib.calculateInterest();
                ib.applyInterest();
                totalInterestPaid += interest;
            }

            totalDeposits += acc.getBalance();
        }

        System.out.printf("%nTotal Assets Under Management: ₹%.2f%n", totalDeposits);
        System.out.printf("Total Interest Distributed:    ₹%.2f%n", totalInterestPaid);
    }

    // Add new account — no other method needs to change
    public void addAccount(Account account) {
        allAccounts.add(account);
    }
}
```

```java
List<Account> accounts = List.of(
    new SavingsAccount("SAV001", "Alice", 50000),
    new CurrentAccount("CUR001", "Bob",   80000, 10000),
    new FixedDepositAccount("FD001", "Carol", 200000, 12),
    new PremiumSavingsAccount("PREM001", "Diana", 500000, 0.02, "Raj")
);

BankingBatchProcessor processor = new BankingBatchProcessor(new ArrayList<>(accounts));
processor.runMonthlyBatch();

// Tomorrow, if we add a new JuniorSavingsAccount type:
// — JuniorSavingsAccount extends SavingsAccount → done
// — addAccount(new JuniorSavingsAccount(...))   → done
// — runMonthlyBatch() needs ZERO changes         ← Power of polymorphism
```

---

### Overloading vs Overriding — Complete Comparison

||Method Overloading|Method Overriding|
|---|---|---|
|**Type**|Compile-time (Static) Polymorphism|Runtime (Dynamic) Polymorphism|
|**Location**|Same class|Parent → Child class|
|**Signature**|Must differ (params)|Must be identical|
|**Return type**|Can differ|Must be same or covariant|
|**`@Override`**|Not applicable|Strongly recommended|
|**Access modifier**|Can be anything|Cannot be more restrictive|
|**`static` methods**|Yes — overloading applies|No — static methods shadow, not override|
|**`private` methods**|Yes|No — private not inherited|
|**Resolution**|Compiler (based on arg types)|JVM (based on actual object)|

---

### Interview Questions — Polymorphism

**Q1. What is runtime polymorphism? How does it work internally?**

> Runtime polymorphism is the JVM's ability to resolve an overridden method call based on the actual object type at runtime, not the reference type. Internally, the JVM uses a **Virtual Method Table (vtable)** — a per-class table of method pointers. When a method is called on a reference, the JVM follows the hidden class pointer from the object to its vtable and invokes the correct method. This is called dynamic dispatch.

**Q2. What is the output?**

```java
class A { void show() { System.out.println("A"); } }
class B extends A { void show() { System.out.println("B"); } }
class C extends B { void show() { System.out.println("C"); } }

A obj = new C();
obj.show();
```

> **Output: C** — Object is `C`, and `C`'s vtable has `show()` pointing to `C::show()`. The reference type `A` is irrelevant at runtime.

**Q3. Can we override a static method?**

> No. Static methods are resolved at compile time based on the reference type. If a child class defines a static method with the same signature, it is **method shadowing**, not overriding. `@Override` on a static method causes a compiler error.

**Q4. What is covariant return type?**

> An overriding method can return a **subtype** of the parent method's return type. For example, if `Animal create()` is the parent, the child can override with `Dog create()` since `Dog` IS-A `Animal`. This is valid in Java 5+.

**Q5. What is the output?**

```java
class Parent {
    Parent() { display(); }
    void display() { System.out.println("Parent display"); }
}
class Child extends Parent {
    int x = 10;
    Child() { super(); }
    void display() { System.out.println("Child display, x = " + x); }
}
new Child();
```

> **Output: Child display, x = 0** — Dynamic dispatch calls `Child::display()` from `Parent`'s constructor, but `x = 10` hasn't been initialized yet (it's `0` by default at the time of the constructor call). **This is the constructor trap — never call overridable methods from constructors.**

**Q6. Can polymorphism be achieved without inheritance?**

> Not in the traditional Java sense. Polymorphism requires an override relationship, which requires inheritance (via `extends` or `implements`). Interface-based polymorphism is still inheritance — of the interface contract.

**Q7. What is the difference between method overloading and method overriding?**

> Overloading is compile-time polymorphism — same method name, different parameter signature, resolved by the compiler. Overriding is runtime polymorphism — same signature in parent and child, resolved by the JVM using dynamic dispatch. Overloading is about "multiple forms of the same operation." Overriding is about "child's custom behavior replacing parent's."

---

---

# ALL 4 PILLARS TOGETHER — Integrated Banking System

```java
/*
 * How all 4 pillars collaborate:
 *
 * ENCAPSULATION → BankAccount fields (balance, pin) are private
 *                 Controlled via deposit(), withdraw(), validatePin()
 *
 * ABSTRACTION   → Account interface defines the contract
 *                 AbstractAccount provides the template
 *                 Implementations hide their own complexity
 *
 * INHERITANCE   → SavingsAccount extends AbstractAccount
 *                 PremiumSavingsAccount extends SavingsAccount
 *                 Multi-level hierarchy with code reuse
 *
 * POLYMORPHISM  → List<Account> holds all types
 *                 Single loop processes all accounts correctly
 *                 Dynamic dispatch calls the right method per type
 */

public class BankingSystemDemo {
    public static void main(String[] args) {

        // POLYMORPHISM: Parent reference, child objects
        List<Account> portfolio = new ArrayList<>();
        portfolio.add(new SavingsAccount("SAV001", "Alice", 50000));
        portfolio.add(new CurrentAccount("CUR001", "Bob", 80000, 10000));
        portfolio.add(new FixedDepositAccount("FD001", "Carol", 200000, 12));
        portfolio.add(new PremiumSavingsAccount("PREM001", "Diana", 500000, 0.02, "Raj"));

        // ENCAPSULATION: Direct field access not possible from here
        // portfolio.get(0).balance = 9999999;  ← COMPILER ERROR

        // ABSTRACTION: Code uses Account interface — doesn't care about specific types
        System.out.println("==== Portfolio Overview ====");
        double totalAUM = 0;
        for (Account acc : portfolio) {
            acc.printStatement();           // POLYMORPHISM: correct print per type
            totalAUM += acc.getBalance();   // ABSTRACTION: same call, every type supports it
        }
        System.out.printf("Total AUM: ₹%.2f%n%n", totalAUM);

        // POLYMORPHISM + ABSTRACTION: interest-bearing accounts handled uniformly
        System.out.println("==== Applying Monthly Interest ====");
        for (Account acc : portfolio) {
            if (acc instanceof InterestBearing ib) {
                ib.applyInterest();
            }
        }

        // INHERITANCE: PremiumSavingsAccount retains all parent capabilities
        PremiumSavingsAccount premium = (PremiumSavingsAccount) portfolio.get(3);
        double cashback = premium.getCashback(50000);
        System.out.printf("%nCashback on ₹50000 transaction: ₹%.2f%n", cashback);
    }
}
```

---

---

# MASTER SUMMARY

## The 4 Pillars — One-Page Reference

|Pillar|Core Idea|Mechanism|Key Benefit|Memory Hook|
|---|---|---|---|---|
|**Encapsulation**|Protect data|`private` + getters/setters|Invalid state impossible|_"You can't touch that"_|
|**Abstraction**|Hide complexity|`interface` / `abstract class`|Swap implementations freely|_"You don't need to know that"_|
|**Inheritance**|Reuse structure|`extends` / `implements`|No code duplication|_"I get what you have"_|
|**Polymorphism**|One call, many forms|`@Override` + dynamic dispatch|Add types without changing code|_"Same call, different result"_|

---

## What Dispatches Dynamically vs Not

|Method Type|Dispatch|Reason|
|---|---|---|
|Instance method (overridden)|✅ Runtime (dynamic)|vtable lookup on actual object|
|`static` method|❌ Compile-time|Belongs to class, not object|
|`private` method|❌ Compile-time|Not in vtable (not inherited)|
|`final` method|❌ Compile-time|Can't be overridden — direct call|
|Overloaded method|❌ Compile-time|Resolved by argument types|
|Constructor|❌ Compile-time|Not inherited|

---

## Common Interview Traps — Quick Reference

|Trap|Answer|
|---|---|
|Can abstract class have constructor?|Yes — called via `super()` from child|
|Can interface have instance fields?|No — only `public static final` constants|
|Can we override private methods?|No — private is not inherited|
|Does field access use dynamic dispatch?|No — fields always resolved by reference type|
|Can static methods be overridden?|No — they are shadowed (compile-time)|
|What if child class doesn't call `super()`?|Compiler auto-inserts no-arg `super()` call|
|Can we instantiate an abstract class?|No — but can use anonymous inner class|
|Overloading vs Overriding?|Compile-time (same class, different params) vs Runtime (parent-child, same signature)|
|Constructor chaining order?|Parent constructor always runs before child|
|Calling overridable method in constructor?|Dangerous — child method runs before child fields initialize|

---

## The Car Analogy — All 4 Pillars at a Glance

```
ABSTRACTION   →  Steering wheel, pedals, gear shift
                 You know WHAT they do — not HOW

ENCAPSULATION →  Engine is sealed under the hood
                 You interact only through controlled interfaces (pedals)

INHERITANCE   →  ElectricCar IS-A Car IS-A Vehicle
                 ElectricCar gets all Vehicle and Car capabilities

POLYMORPHISM  →  vehicle.accelerate() works correctly whether
                 vehicle is a Car, Truck, or Motorcycle — same call, right behavior
```

---

_End of Notes — OOP in Java: Encapsulation · Abstraction · Inheritance · Polymorphism_