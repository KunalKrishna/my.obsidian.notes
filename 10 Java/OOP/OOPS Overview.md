Why OOPS? 
4 pillars  : e-API
1. Encapsulation — Protecting Data
2. Abstraction — Hiding Complexity
3. Polymorphism — Adapting Behavior (**one interface, many implementations**)
4. Inheritance — Reusing Code

4 pillars
1. Encapsulation — Protecting Data
2. Abstraction — Two Implementation Mechanism in Java
	1. `abstract class` : partial abstraction
	2. `interface` : full abstraction 
3. Polymorphism — Two Types
	1. **Compile-time Polymorphism** (Static): Method Overloading — resolved by compiler
	2. **Runtime Polymorphism** (Dynamic): Method Overriding — resolved by JVM at runtime via **Dynamic Dispatch** (also called **Dynamic Method Dispatch** or **Late Binding**) 
4. Inheritance — IS-A relationship 
	1. Single Inheritance
	2. Multi-Level Inheritance ✅
	3. Multiple Inheritance ❌  — Java Does NOT Allow (**Diamond Problem**) 
	4. Inheritance vs Composition : Prefer composition(HAS-A) when IS-A is not true.


TL;DR
**Encapsulation** = _how_ you protect data | **Abstraction** = _what_ you expose to the world

## Encapsulation — Protecting Data
**Definition:** Bundling data (fields) and the methods that operate on that data into a single unit (class), and **restricting direct access** to the internals.
**The "why":** Prevent invalid state. You control how data is read or written.
**What's being hidden?** The raw _data_ (`balance`). The _implementation_ of the rules.
## Abstraction — Hiding Complexity
**Definition:** Exposing **only what's necessary** to the outside world and hiding _how_ something works internally. Achieved via `abstract` classes and `interfaces`.
**The "why":** Reduce complexity for the user of your code. They don't need to know the engine, just the steering wheel.
**What's being hidden?** The *complexity* of the implementation. The *details* behind the interface.

## The One-Line Memory Trick
- **Encapsulation** → _"You can't touch that."_ (data protection, access control)
- **Abstraction** → _"You don't need to know that."_ (complexity hiding, clean interfaces)

## Inheritance
**Definition:** A class (_child/subclass_) acquires the properties and behaviors of another class (_parent/superclass_), enabling **code reuse** and establishing an **"IS-A" relationship**.
**The "why":** Don't rewrite what already exists. Model real-world hierarchies naturally.


-----


