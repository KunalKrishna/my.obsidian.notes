[Software Testing Tutorial - GeeksforGeeks](https://www.geeksforgeeks.org/software-testing/software-testing-tutorial/)

## 7 fundamental Principles of Software testing

[What are the 7 Testing principles? - YouTube](https://www.youtube.com/watch?v=idg7MOKWNCU)
![[7 fundamental principles of software testing.png]]
### 1. Testing Shows the Presence of Defects
### 2. Exhaustive Testing is Impossible

### 3. Early Testing
### 4. Defect Cluster Together (Pareto's Principle)

### 5. Pesticide Paradox
### 6. Testing is Context-Dependent

### 7. Absence of Errors Fallacy


## STLC Phases 
The Core Testing Process
1. **Analyze Requirements**: Understand the expected behavior through product user stories.
2. **Design Test Cases**: Document specific input data, steps, and expected outputs.
3. **Configure Environment**: Prepare a staging space that mimics the live production setup.
4. **Execute & Log**: Run the test cases and record any deviations as defects.
5. **Retest & Close**: Validate the developers' bug fixes and compile closure reports.

![[SDLC vs STLC.png|697]]

![[Pasted image 20260703024920.png]]
![[Pasted image 20260703014333.png]]

![[Pasted image 20260703014402.png]]
![[STLC.png]]
![[STLC2.png]]
####  1. Requirement Analysis
software requirement specification (SRS) document reviewed.
SRS ==> testable conditions.
decided whether to go with a **manual** or **automated** testing process.
#### 2. Test Planning
includes the test strategy, approach, schedule, and resources.
Testers define the **test strategy**, scope, resource requirements, tools, and timelines in a comprehensive **Test Plan document**. 
#### 3. Test case Development
A software test case is a set of conditions under which a software component or system should be tested.
The team creates detailed **test cases**, **test scripts**, and **test data**, and establishes the metrics that will be used to track testing progress.
#### 4. Test Environment Setup
 It includes the hardware, software, and network infrastructure required to test the software. Additionally, the **test data** is also created in this stage.
#### 5. Test Execution
carried out manually or automatically and can be done in an **iterative** or **non-iterative manner**.
#### 6. Test Cycle Closure
a software testing report is generated that contains information on the software defects and their status
Once all criteria are met, the team evaluates the overall testing quality, compiles **test metrics**, and delivers a **Test Summary Report** before the software is approved for production.

[software testing life cycle \| Edure Learning](https://edure.in/software-testing-life-cycle/)


## Types of Software Testing

### Manual Testing
#### 1. White Box testing
white box testing = clear box, glass box, or structural testing
Types of White Box Testing :  
- [Path Testing](https://www.geeksforgeeks.org/software-engineering/path-testing-in-software-engineering/): Focuses on testing all possible execution paths in a program to ensure each path behaves correctly and all branches are covered.
- [Loop Testing](https://www.geeksforgeeks.org/software-testing/loop-software-testing/): Ensures that loops work correctly by validating initialization, execution, and termination under different conditions.
- [Unit Testing](https://www.geeksforgeeks.org/software-testing/unit-testing-software-testing/): Checks individual functions or components in isolation to ensure each unit performs its intended functionality correctly.
- [Mutation Testing](https://www.geeksforgeeks.org/software-engineering/software-testing-mutation-testing/): Evaluates test case effectiveness by introducing small changes in the code to verify whether test cases can detect errors.
- [Integration Testing](https://www.geeksforgeeks.org/software-testing/software-engineering-integration-testing/): Verifies the interaction between different modules or components to ensure smooth data flow and correct communication.
- [Penetration Testing:](https://www.geeksforgeeks.org/penetration-testing-software-engineering/) Simulates cyber-attacks to identify security weaknesses and ensure the application is protected from unauthorized access.

a software testing method where the tester has **full visibility into the internal structure, design, and code** of the application.
##### Static Testing

Static testing is a software testing practice that evaluates software artifacts, such as source code and associated documentation, **without actually executing the program**.
also known as **dry run testing** and **verification testing**.

![[Static testing.png]]
#### 2. Black Box testing
##### a. Functional Testing (what)
Evaluates **what** the software does (e.g., logging in, checking out).
a broad category. It includes many different testing levels and techniques that check features, inputs, and outputs:
![[functional testing.png]]

- **Unit Testing:** Tests individual components. 
- **Integration Testing:** Tests combined components. 
- **System Testing:** Tests the complete application. 
- **Smoke Testing:** Checks core build stability. (broader & shallow)
	- verifies whether the critical functionalities of a new build are working correctly. 
- **Sanity Testing:** Checks specific bug fixes. (Narrow & Deep)
	- verifies whether the specific functionality works correctly. 
- **Regression Testing:** Checks for unintended side effects. to ensure that recent code changes do not negatively affect existing functionality 

```  
[ New Build Dropped ]
	│ 
	▼ 
1. SMOKE TESTING ────────► Failed? ──► REJECT BUILD (Stop Testing)
    │ Pass
    ▼
2. SANITY TESTING ───────► Failed? ──► REJECT FIX (Stop Testing)
    │ Pass
    ▼
3. REGRESSION TESTING ───► Failed? ──► FIX BUGS (Keep Testing)
    │ Pass
    ▼ 
[ Ready for Release ]
```

![[Pasted image 20260703030520.png]]
![[smoke test.png]]

![[sanity testing.png]]

```
[CODE DEV] ──►[LEVEL 1: UNIT ] ──► (Runs Unit Regression)
                     │
                     ▼
              [LEVEL 2: INTEGRATION ] ──► (Runs Integration Smoke)
                     │
                     ▼
              [LEVEL 3: SYSTEM (QA) ] 
                     │
                     ├─► Step 1: Run SMOKE TEST (Is the build stable?)
                     ├─► Step 2: Run SANITY TEST (Did the bug fixes work?)
                     └─► Step 3: Run REGRESSION TEST (Is the rest of the app safe?)
                     │
                     ▼
              [LEVEL 4: UAT ] ──► (Final client sign-off)

```
##### b. Non-Functional Testing (how)
Evaluates **how** the software performs (e.g., speed, security, scalability)
- **Performance/Load Testing:** Checking speed and scalability under stress.
- **Security Testing:** Checking for vulnerabilities and data leaks.
- **Usability Testing:** Checking how user-friendly the interface is.
- Reliability Testing
- **Compatibility Testing:** Checking how it runs on different browsers or devices

![[Pasted image 20260703025820.png]]


#### 3. Grey Box testing



### Automation Testing
integral part of CI/CD pipelines
![[Automation Testing.png]]


---

![[Pasted image 20260703025105.png]]
![[Pasted image 20260703025009.png]]


## Four Levels of Software Testing
![[FunctionalTesting.png]]

1. **Unit Testing** : focuses on verifying individual units or components of the application in isolation. The main goal is to identify and fix defects early before integrating these components with other parts of the system. `[JUnit, Mockito]`

2. **Integration Testing** : to verify the interaction and communication b/w software modules or components. It is important because even if individual parts work perfectly, issues may arise when they interact with one another.  `[JUnit - @BeforeAll/@AfterAll, @BeforeEach/@AfterEach, Mockito]`
	1. Big Bang Integration Testing : 
	2. Bottom-Up Integration Testing :
		1. A **driver** is a dummy module that simulates a **higher-level component** (a parent module). It is used to call, feed data to, and invoke the module you want to test.
		2. Drivers are used in **Bottom-Up integration testing**, where lower-level utility modules are built and tested first. Because the main interface or control layer doesn't exist yet to trigger these low-level features, a developer writes a temporary script (the driver) to act as the "boss" and run the test.
		3. **Role:** Acts as an **invoker / initiator**.
		4. **Analogy:** A test driver in a car factory turning the ignition switch on a bare engine block sitting on a workbench
		5. ```
		   [ Login Module ] (Under Test) 
		         │ 
		         ▼ (Calls) 
		   [ Database Stub ] 
		   (Temporary dummy code that always returns "Login Successful")
		   ```
	3. Top-Down Integration Testing  :
		1. A **stub** is a dummy module that simulates a **lower-level component** (a child module). It is called by the module you are currently testing. A stub doesn't contain real business logic; it merely accepts a call from the higher module and returns hardcoded, predictable data.
		2. - **Role:** Acts as a **receiver / responder**.
		3. **Analogy:** A phone operator who answers your call and reads from a fixed script, no matter what you ask. 
		4. ```
		   [ Payment Driver ] (Temporary script passing an order ID and $50.00) 
		        │ 
		        ▼ (Invokes) 
		   [ Payment Engine ] (Under Test)
		   ```
	4. Mixed
![[integration testing.png]]

3. **System Testing** : to evaluate the complete and fully integrated software system to ensure it meets the specified requirements. This stage checks whether the entire system functions as expected in a real-world environment. It includes both functional and non-functional tests.
![[system testing.png]]
Types of system testing : 
- Functional testing
- Performance testing
- Load
- Stress
- Security
- Usability
- Compatibility 
- Recovery
- Installation
- Reliability 

4. **Acceptance Testing (UAT)** : 


## Performance Testing

The 6 Core Types of Performance Testing

Performance testing is an umbrella term that includes several specialized variations:
1. **Load Testing:** Testing the system with a steadily increasing load up to its expected normal and peak capacity to see how it responds. Goal : **Validation**
2. **Stress Testing:** Pushing the system _beyond_ its normal operational capacity to find its breaking point and observe how it recovers. Goal : **Resilience**
3. **Spike Testing:** Suddenly and dramatically flooding the system with a massive surge of users to see if it handles sudden traffic spikes gracefully.
4. **Endurance (Soak) Testing:** Running a sustained, normal workload over a long period (e.g., 12 to 72 hours) to discover memory leaks or resource degradation. 
5. **Volume (Flood) Testing:** Flooding the system's database with massive amounts of data to evaluate file processing speeds and search indexing behavior. 
6. **Scalability Testing:** Adjusting the system's hardware or cloud infrastructure (e.g., adding CPUs or RAM) to measure how well the software scales upward to support more load.

![[Performance Testing.png]]

⚖️ 1. Load Testing vs. Stress Testing (Capacity vs. Failure)

This is the most common point of confusion. The key differentiator is the **threshold of traffic**. 
- **Load Testing** evaluates if the system meets its promised Service Level Agreements (SLAs) under expected real-world usage. You want to see if pages load within 2 seconds when 5,000 normal users browse simultaneously. The goal is **validation**. 
- **Stress Testing** purposefully tries to break the system. You throw 50,000 users at a system designed for 5,000. You want to answer two things: _At what number does it crash?_ and _When it crashes, does it fail safely (giving a clean error page) or catastrophically (corrupting the database)?_ The goal is **resilience**. 

⚡ 2. Stress Testing vs. Spike Testing (Magnitude vs. Velocity)

Both types subject the application to extreme, punishing workloads, but they differ entirely in **how the traffic arrives**. 
- **Stress Testing** applies a gradual, incremental ramp-up. Traffic climbs steadily over hours, allowing load balancers and auto-scaling cloud features time to react. It tests absolute **structural limits**. 
- **Spike Testing** applies a near-instantaneous shock to the system. Traffic goes from 0 to 100% in a matter of seconds (like a ticket drop for a major concert, or a breaking news alert). It tests the **elastic speed** of the architecture—whether the system can spin up new cloud servers fast enough before crashing under the sudden wall of requests. 

⏳ 3. Load Testing vs. Endurance Testing (Short Peak vs. Long Burn)

Both test the system under normal, acceptable workload boundaries, but they vary in **duration**.
- **Load Testing** usually runs for a brief window (e.g., 1 to 2 hours) to simulate a standard peak hour of business traffic. It answers if the app works right now.
- **Endurance (Soak) Testing** runs the same normal load continuously for days. This is the only way to catch **hidden time-based bugs**. For example, a tiny 1MB memory leak won't hurt a 1-hour load test, but over a 48-hour soak test, it will entirely consume the server's RAM and cause a system crash
![[perf test 2.png]]

![[perf test3.png]]
## Other Types of Testing
### Alpha Testing
### Beta Testing
performed by real users in a real-world environment before the final release.
### Exploratory Testing
### Ad Hoc Testing
### Installation Testing
### Globalization Testing
### Object Oriented Testing
### Localization Testing
### A/B Testing
### GUI Testing

## AAA Pattern

What is the AAA Pattern?

1. **Arrange:** Set up the test environment. This is where you instantiate the class under test, create your Mockito mocks, and define the behavior (stubbing) of those mocks.
2. **Act:** Execute the target method or action you want to test. This should generally be a single line of code.
3. **Assert:** Verify the outcome. This is where you check if the returned value is correct and use Mockito to verify that dependencies were called with the right parameters.

### ATDD (Acceptance Test-Driven Development)

a collaborative development methodology where the entire team defines precise acceptance criteria _before_ any code is written. It ensures that developers, testers, and business stakeholders share an identical understanding of what features the application must deliver. 

t is often summarized by the three "O" positions: **Our Business** (Product Owner), **Our Technology** (Developer), and **Our Quality** (QA) working together.

#### The 4-Step ATDD Cycle

The process follows a continuous loop during feature development, often built directly around Agile user stories

```
┌──────────────┐   ┌────────────┐   ┌────────────┐   ┌──────────┐ 
│ 1. Discuss   │──>│ 2. Distill │──>│ 3. Develop │──>│ 4. Demo  │
└──────────────┘   └────────────┘   └────────────┘   └──────────┘
```

- **Discuss**: The Product Owner, Developer, and QA engineer ("Three Amigos") meet to review a user story. They deliberate on edge cases, business rules, and expected behaviors.  
- **Distill**: The team converts these conversational business requirements into structured, human-readable acceptance tests. These tests are written in plain language (usually using **Gherkin syntax**: _Given/When/Then_). 
- **Develop**: The developer hooks these tests up to an automation framework (like Cucumber). Initially, the tests fail because the code does not exist. The developer writes production code until the tests pass.  
- **Demo**: The passing automated tests serve as living proof that the business requirements are met. The feature is demonstrated to stakeholders for final sign-off



# Testing technique - java tool


| Test                    | Java Tool                                                                                                                      | Other lang                                                                                            |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| Unit Testing            | Junit, Mockito, TestNG                                                                                                         | PHPUnit, `unittest`, `pytest`                                                                         |
| Integration Testing     | RestAssured, JUnit / TestNG                                                                                                    | Postman, Pytest                                                                                       |
| Big Bang IT             | Selenium, JUnit/TestNG, Apache JMeter                                                                                          | Postman                                                                                               |
|                         |                                                                                                                                |                                                                                                       |
| Performance Testing :   |                                                                                                                                |                                                                                                       |
| Load Testing            | Apache JMeter                                                                                                                  | WebLoad, NeoLoad, LoadUI Pro,                                                                         |
| Stress Testing:         | Apache JMeter                                                                                                                  | WebLoad, NeoLoad, SmartMeter                                                                          |
| Spike Testing           | Apache JMeter                                                                                                                  | Gatling, Locust, k6, Micro Focus LoadRunner                                                           |
| Endurance Testing       | Apache JMeter                                                                                                                  | Gatling, LoadRunner, New Relic                                                                        |
| Volume Testing          |                                                                                                                                | NeoLoad, LoadRunner, BlazeMeter                                                                       |
|                         |                                                                                                                                |                                                                                                       |
|                         |                                                                                                                                |                                                                                                       |
| Mutation Testing        | Jduy, Jester, Jumble, PIT, MuClipse                                                                                            |                                                                                                       |
| Penetration Testing     |                                                                                                                                | Nmap, Wireshark(packet analyzer), Nessus(vulnerability scanner), Burp Suite(web app security testing) |
|                         |                                                                                                                                |                                                                                                       |
| Alpha Testing           |                                                                                                                                |                                                                                                       |
| <br>Beta Testing        |                                                                                                                                | TestFairy, Centercode, TryMyUI, UserTesting, TestRail, TestFlight(iOS)                                |
| Ad hoc Testing          |                                                                                                                                |                                                                                                       |
| Object Oriented Testing | JUnit, TestNG, Selenium, Mockito, Apache JMeter, Eclipse & IntelliJ IDE                                                        |                                                                                                       |
| A/B Testing             |                                                                                                                                |                                                                                                       |
| GUI testing             |                                                                                                                                | Selenium, Playwright,Appium(Android+iOS)                                                              |
| Concurrency test        | **JCStress** (Java Concurrency Stress), Native Multi-Threaded Testing with `CountDownLatch`, ThreadWeaver (by Google), vmlens, |                                                                                                       |

## Core Strategies for Testing Concurrency
1. Stress Testing (High Volume)
- Execute the target code block concurrently across thousands of threads or processes simultaneously.
- Run tests inside deep nested loops for extended durations to force specific race condition windows.
- Flood shared state objects with rapid, overlapping read and write requests to verify atomicity.

2. Artificially Inducing Delays (Jitter)
- Inject microscopic, randomized sleep delays (`Thread.sleep()`, `await`) directly between shared memory operations.
- Insert temporary hook methods or conditional break points during development to deliberately pause specific threads.
- Force context switching by running tests on machines with heavily constrained CPU core configurations. 

3. Thread Synchronization Hooks
- Use primitive concurrency utilities like `CountDownLatch`, `CyclicBarrier`, or semaphores to line up multiple threads.
- Release all blocked worker threads simultaneously to ensure they hit the critical code section at the exact same millisecond.
- Assert that the final state of the object matches the exact mathematical total of all combined thread operations.

4. Specialized Tooling & Static Analysis
- Use specialized language-level frameworks designed to systematically explore interleavings (e.g., JCStress for Java, `go test -race` for Go).
- Run address, thread, and memory sanitizers during compilation to catch unsafe data sharing before runtime.
- Apply static code analyzers to scan source code for unguarded mutable state or improper lock ordering.

# Machine Learning Testing Tools

Implementing these tests manually is tedious, so teams rely on specialized open-source testing frameworks: 

|Tool|Core Strength|Primary Use Case|
|---|---|---|
|**[Deepchecks](https://dagshub.com/blog/top-machine-learning-model-testing-tools/)**|Data & Model Validation|Automatically checks data integrity, data splits, and performance tracking.|
|**[Great Expectations](https://greatexpectations.io/)**|Data Quality|Validates, documents, and profiles data pipelines to prevent bad inputs.|
|**[CheckList](https://github.com/marcotcr/checklist)**|Behavioral Testing|Specifically designed for behavioral testing (INV, DIR, MFT) of NLP models.|
|**[Evidently AI](https://www.evidentlyai.com/)**|Monitoring & Drift|Generates reports on data drift, target drift, and regression/classification quality.|