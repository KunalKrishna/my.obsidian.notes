---
type: concept 
area: java 
topic: testing 
status: learning # todo | learning | solid | rusty 
confidence: 2 # 1-5, honest 
last-reviewed: 2026-07-23 
tags: [high-yield]
---
Testing Java applications ensures that your code works as expected, behaves predictably under edge cases, and doesn't break when you make changes later.

Here is a breakdown of how Java testing works, structured for a developer who knows Java code but is new to testing concepts.

---
## 1. The Core Concept: Assertions

In standard Java, you write code that produces output. In testing, you write code that **compares actual output against expected output**.

This comparison is done using **Assertions**. If an assertion passes, the test passes. If an assertion fails, the test throws an error, flagging the code as broken.

- **Production Code:**
```Java
public class Calculator {
	public int add(int a, int b) {
		return a + b;
	}
}
```
- **Test Code:**
```Java
// Checking if 2 + 3 equals 5
assertEquals(5, calculator.add(2, 3)); 
```

---
## 2. Anatomy of a Test Method: AAA (Arrange, Act, Assert)

Tests are written in standard Java classes, typically placed in the `src/test/java` directory (separate from your production code in `src/main/java`).

A typical unit test follows the **AAA (Arrange, Act, Assert)** pattern:
1. **Arrange:** Set up the test data or objects required.
2. **Act:** Call the method you are testing.
3. **Assert:** Verify that the result matches your expectations.

A great alternative to **AAA** is the **FIRST** principles framework, but for replacing the three-phase structure itself, a popular and highly intuitive standard is **GWT**:
### **GWT (Given - When - Then)**
Originating from Behavior-Driven Development (BDD), **GWT** maps directly onto the three testing phases while using natural language that clearly describes the _nature_ of each phase:
#### **1. G — Given (Setup / Context)**
- **Nature:** The initial state of the system before any action happens.
- **What you do:** Instantiate objects, set up test data, configure mocks, or establish preconditions.
- **Mental Check:** _"Given this starting scenario..."_
#### **2. W — When (Execution / Action)**
- **Nature:** The specific trigger or operation being evaluated.
- **What you do:** Call the exact method or invoke the target behavior under test (ideally just one line of code).
- **Mental Check:** _"When I trigger this action..."_
#### **3. T — Then (Outcome / Verification)**
- **Nature:** The observation and validation of results.
- **What you do:** Run assertions (`assertEquals`, `assertTrue`) or verify interactions to ensure the expected state or return value matches reality.
- **Mental Check:** _"Then this result must occur."_

---
#### Code Comparison

````tabs
tab: AAA
```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;

public class CalculatorTest {

    @Test // Marks this method as a runnable test
    void testAddition() {
        // 1. Arrange
        Calculator calc = new Calculator();

        // 2. Act
        int result = calc.add(10, 20);

        // 3. Assert
        assertEquals(30, result, "10 + 20 should equal 30");
    }
}
```

tab: GWT
```java 
@Test
void testCalculatorAddition() {
    // GIVEN: A fresh Calculator instance and input values
    Calculator calc = new Calculator();
    int a = 10;
    int b = 20;

    // WHEN: The add method is executed
    int result = calc.add(a, b);

    // THEN: The output must equal 30
    assertEquals(30, result);
}
```
````

---
#### Why **GWT** is easier to remember:
- **Narrative Flow:** It reads like a complete sentence: _"Given X, When Y happens, Then expect Z."_
- **Clearer Distinction:** Unlike _Arrange_ vs _Act_ (which both start with 'A'), **Given**, **When**, and **Then** are visually and phonetically distinct.

---
## 3. The Testing Ecosystem (Tools You Need)
To write and run tests in Java, you rely on a few standard tools:
### A. Testing Frameworks (e.g., JUnit 5)
JUnit is the industry-standard framework for writing Java tests. It provides:
- **Annotations:** `@Test` (defines a test), `@BeforeEach` (runs setup code before each test), `@AfterEach` (cleanup), `@Disabled` (skips a test).
- **Assertions:** `assertEquals(expected, actual)`, `assertTrue(condition)`, `assertNotNull(object)`, `assertThrows(Exception.class, () -> method())`.
### B. Mocking Frameworks (e.g., Mockito)

Real-world code has dependencies (databases, external APIs, services). To test a single unit of code without invoking a real database or network call, you use **Mocking**.
- Mockito creates fake ("mock") objects that simulate real behavior.
- **Example:** Telling a mock database service to return a dummy user without connecting to a real database.

```Java
// Fake user repository
UserRepository mockRepo = Mockito.mock(UserRepository.class);

// Define behavior: when repo.findById(1) is called, return a dummy user
Mockito.when(mockRepo.findById(1)).thenReturn(new User("Alice"));
```
### C. Build Tools & Test Runners (Maven / Gradle)

Your IDE (IntelliJ IDEA, Eclipse) or build tool (Maven/Gradle) automatically scans `src/test/java`, executes every method annotated with `@Test`, and generates a report showing which tests passed or failed.

---
## 4. Levels of Testing

|**Type**|**What it Tests**|**Speed**|**Dependencies**|
|---|---|---|---|
|**Unit Testing**|A single method or class in isolation.|Very Fast|All dependencies are mocked.|
|**Integration Testing**|Multiple components working together (e.g., Spring Service + Database).|Medium|Uses real or embedded databases/services.|
|**End-to-End (E2E) Testing**|The full application flow from API/UI down to the database.|Slow|Uses real running environments.|

---
## 5. Key Best Practices for Beginners

- **One Concept Per Test:** Each test should verify a specific behavior or edge case.
- **Independent Tests:** Tests should never depend on each other or run in a specific order.
- **Test Edge Cases:** Test for null values, empty collections, negative numbers, or thrown exceptions, not just the "happy path."


# JUnit Concepts

Here is an eagle-eye view of JUnit concepts to help build testing intuition and start thinking like a test engineer.

---
## 1. Test Lifecycle Methods & Annotations

When JUnit runs your test class, it doesn't just execute `@Test` methods randomly—it manages a strict **lifecycle** to ensure tests run in clean, predictable environments.

Think of annotations as **instructions to the JUnit test runner** telling it _when_ and _how_ to execute a method:

```
[ @BeforeAll ] (Runs ONCE before any test starts - e.g., start database server)
      │
      ├─── [ @BeforeEach ] (Runs BEFORE Test 1 - e.g., insert fresh test data)
      ├─── [ @Test ] Method 1
      ├─── [ @AfterEach ]  (Runs AFTER Test 1 - e.g., clear database tables)
      │
      ├─── [ @BeforeEach ] (Runs BEFORE Test 2 - e.g., insert fresh test data)
      ├─── [ @Test ] Method 2
      ├─── [ @AfterEach ]  (Runs AFTER Test 2 - e.g., clear database tables)
      │
[ @AfterAll ]  (Runs ONCE after all tests finish - e.g., shut down database)
```
### Core Lifecycle Annotations
- `@Test`: Marks a method as an executable test case.
- `@BeforeEach`: Prepares a fresh state _before every single test_ to prevent tests from affecting one another.
- `@AfterEach`: Cleans up temporary resources _after every single test_.
- `@BeforeAll`: Runs **once** before any tests in the class start (must be a `static` method). Used for expensive, shared setups (e.g., starting an in-memory database).
- `@AfterAll`: Runs **once** after all tests finish (must be `static`). Used for global cleanup.
### Utility Annotations
- `@DisplayName("...")`: Replaces generic Java method names in test reports with clear, human-readable titles (e.g., `"Should throw exception when account balance is negative"`).
- `@Disabled`: Temporarily skips a test without deleting the code (e.g., if a feature is temporarily broken or under active redesign).

---
## 2. Assertions: Verifying Expected Outcomes

In production code, you perform computations. In testing, **assertions** are the heart of verification: they check if the _actual result_ matches the _expected result_. If an assertion fails, the test fails immediately and reports the discrepancy.

Common JUnit 5 assertions:
- `assertEquals(expected, actual)`: Verifies two values or objects are equal (`.equals()`).
- `assertTrue(condition)` / `assertFalse(condition)`: Verifies boolean logic.
- `assertNotNull(object)` / `assertNull(object)`: Validates object state.
- `assertThrows(ExpectedException.class, executable)`: Verifies that a specific piece of code throws an expected exception when invalid input is provided.

---
## 3. Assumptions: Conditional Execution

While **assertions** check whether code works correctly, **assumptions** check whether the **environment or preconditions** are suitable to run the test in the first place.

If an **assertion** fails, the test is marked as **FAILED** ❌.
If an **assumption** fails, the test is marked as **SKIPPED** ⏭️ (not a failure).
### Common Assumption Methods:
- `assumeTrue(condition)`: Evaluates a condition (e.g., checking if the test is running on Linux, or if a specific environment variable is set). If `false`, JUnit halts and skips the rest of the test without marking it as a failure.
- `assumingThat(condition, executable)`: Conditionally executes a block of assertions only if the assumption holds true, while allowing the rest of the test to continue regardless.

---
## Developing Testing Intuition: How to Think Like a QA

Transitioning from a developer mindset ("_How do I build this?_") to a QA/Testing mindset ("_How could this break?_") requires focusing on four main areas:

```less
                  ┌──────────────────────────┐
                  │    HAPPY PATH FIRST      │
                  │ Does it work as intended?│
                  └───────────┬──────────────┘
                              │
         ┌────────────────────┴──────────────────────┐
         ▼                                           ▼
┌───────────────────┐                       ┌──────────────────┐
│   EDGE CASES      │                       │  NEGATIVE PATHS  │
│ Nulls, boundaries,│                       │ Invalid inputs & │
│ empty inputs      │                       │ expected errors  │
└────────┬──────────┘                       └────────┬─────────┘
         │                                           │
         └────────────────────┬──────────────────────┘
                              ▼
                  ┌────────────────────────┐
                  │    TEST ISOLATION      │
                  │ No shared state across │
                  │     test executions    │
                  └────────────────────────┘
```

1. **Test the Happy Path First, Then Try to Break It:**
    Start by verifying that correct input yields correct output. Then immediately switch gears:  What happens if input is `null`, empty, negative, or enormously large?
2. **Verify Failure Behavior (Negative Testing):**
    Good testing isn't just about verifying success; it's about making sure your application fails _gracefully_. Always write tests to confirm that invalid operations throw the expected exceptions (e.g., attempting to withdraw more money than an account holds).
3. **Ensure Test Independence:**
    Never rely on state left over from a previous test. Use `@BeforeEach` to reset state so tests can be run in any order, concurrently, or individually without failing.