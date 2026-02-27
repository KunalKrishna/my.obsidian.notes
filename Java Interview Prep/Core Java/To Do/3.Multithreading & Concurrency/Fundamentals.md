- **Concurrency** = dealing with multiple tasks at the same time (conceptually).  
    Tasks may _interleave_ on a single CPU core.
- **Parallelism** = actually executing multiple tasks at the same time.  
    Requires multiple CPU cores (or processors).
Simple analogy
- **Concurrency**: One chef cooking 3 dishes by switching between them.
- **Parallelism**: Three chefs each cooking one dish at the same time.

Concurrency is about _dealing with_ multiple things at once.  
Parallelism is about _doing_ multiple things at once.

**Nagarro Softwares Pvt. Ltd.**   Gurugram, India _Junior Associate, Technology_ 

![[concurrency_parallelism.mp4]]


| Concept      | About Structure? | About Hardware? | About Waiting?       |
| ------------ | ---------------- | --------------- | -------------------- |
| Concurrency  | ✅ Yes            | ❌ Not required  | ❌ Not necessarily    |
| Parallelism  | ❌ Not design     | ✅ Yes           | ❌ Not about waiting  |
| Asynchronous | ✅ Yes            | ❌ No            | ✅ Yes (non-blocking) |


# 🧩 Simple Mental Model

- **Concurrency** → “Many things in progress.”
- **Parallelism** → “Many things happening at the exact same time.”
- **Asynchronous** → “Don’t wait; come back later.”



![[MultiTh-Concurrency TERMs.png]]


IMP PACKAGES : 

java.lang
- Why: basic threading primitives and monitor methods live here.
	- java.lang.Thread, 
	- java.lang.Runnable, 
	- java.lang.ThreadGroup, 
	- java.lang.ThreadLocal, 
	- java.lang.InterruptedException, 
	- Object.wait/notify/notifyAll (monitor methods)


- [ ] Block from low to high to master concurrency in java
- [ ] Producer/Consumer Queue


# Thread 🧵Lifecycle 🔁

six core states in the Java thread lifecycle:
1. `NEW`, 
2. `RUNNABLE`, 
3. `BLOCKED`, 
4. `WAITING`, 
5. `TIMED_WAITING`, and 
6. `TERMINATED`

**Mnemonic** : **N**ow **R**unning, **B**ut **W**aiting **T**ime **T**erminated.
acronym **NRBWTT** (pronounced _N-R-Bew-T-T_)  

 Think of a thread as a worker: it is **N**ewborn, starts **R**unning, gets **B**locked by a door, decides to **W**ait, takes a **T**imed break, and finally **T**erminates.
 
 ```js
 Think of a thread 🧵 as a worker: 
    it is Newborn 👶, 
				starts Running 🏃, 
						🚪gets Blocked by a door 🏃‍♀️, 
									decides to Wait ⏳,
											takes a Timed break ⏱️, and
													finally Terminates ☠️💀
 ```
