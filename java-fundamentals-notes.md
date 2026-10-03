# Java Fundamentals: Interview Study Notes

> Written in plain language with analogies. For each topic: **the idea → how it works inside → what interviewers ask.**
> Tip for the interview: always explain *why*, not just *what*. "HashMap is O(1) **because** it jumps straight to a bucket using the hash" beats "HashMap is O(1)".

## Table of Contents

1. [How Java code runs: JDK, JRE, JVM](#1-how-java-code-runs-jdk-jre-jvm)
2. [JVM memory: stack vs heap](#2-jvm-memory-stack-vs-heap)
3. [How a basic program works (step by step)](#3-how-a-basic-program-works-step-by-step)
4. [Pass by value vs pass by reference](#4-pass-by-value-vs-pass-by-reference)
5. [Primitives, wrappers, String, immutability](#5-primitives-wrappers-string-immutability)
6. [OOP: the four pillars](#6-oop-the-four-pillars)
7. [Keywords: static, final, abstract, interface, and more](#7-keywords-static-final-abstract-interface-and-more)
8. [equals() and hashCode()](#8-equals-and-hashcode)
9. [Error handling (exceptions)](#9-error-handling-exceptions)
10. [Garbage collection](#10-garbage-collection)
11. [Data structures: how they work inside](#11-data-structures-how-they-work-inside)
12. [Files and processing large files](#12-files-and-processing-large-files)
13. [Multithreading](#13-multithreading)
14. [Database indexing](#14-database-indexing)
15. [Generics, lambdas, streams (modern Java)](#15-generics-lambdas-streams-modern-java)
16. [Rapid-fire Q&A](#16-rapid-fire-qa)
17. [One-page cheat sheet](#17-one-page-cheat-sheet)

---

## 1. How Java code runs: JDK, JRE, JVM

### The big idea: "Write once, run anywhere"

Normal compiled languages (C/C++) turn your code into machine code for **one specific** CPU and OS. Java instead compiles to **bytecode**, a universal in-between language. Each machine has a **JVM** that translates bytecode for that machine.

```
Hello.java  --javac-->  Hello.class (bytecode)  --JVM-->  runs on Windows / Mac / Linux
 (source)    (compiler)   (platform-independent)   (platform-specific)
```

### JDK vs JRE vs JVM (Russian dolls)

```
┌───────────────────────────── JDK ─────────────────────────────┐
│  Dev tools: javac (compiler), jar, javadoc, jdb (debugger)    │
│  ┌──────────────────────── JRE ────────────────────────────┐  │
│  │  Class libraries (java.lang, java.util, java.io ...)    │  │
│  │  ┌──────────────────── JVM ──────────────────────────┐  │  │
│  │  │  Class Loader, Memory areas, Execution Engine     │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

| Term | What it is | Who needs it |
|------|-----------|--------------|
| **JVM** | The virtual machine that *runs* bytecode | Everyone running Java |
| **JRE** | JVM + standard libraries (needed to *run* programs) | End users |
| **JDK** | JRE + compiler and tools (needed to *build* programs) | Developers |

(Since Java 11, Oracle ships mostly just the JDK. The JRE as a separate download is mostly gone, but the concept is still asked.)

### What happens inside the JVM

**Step 1: Compile.** `javac` checks syntax and types, then produces `.class` files (bytecode). Compile-time errors (typos, type mismatches) get caught here.

**Step 2: Class Loading.** When your program needs a class, the **ClassLoader** finds the `.class` file and loads it into memory. It happens in 3 phases:

1. **Loading**: read the bytecode.
2. **Linking**: *verify* (is the bytecode safe/valid?), *prepare* (allocate memory for static variables with default values), *resolve* (turn symbolic references into real memory addresses).
3. **Initialization**: run static initializers and assign real static values.

Class loaders follow **parent delegation**: before loading a class itself, a loader asks its parent first.
- **Bootstrap**: loads core Java classes (`java.lang.*`).
- **Platform** (formerly Extension): loads platform modules.
- **Application/System**: loads *your* classes from the classpath.

Why delegation? Security. Nobody can sneak in a fake `java.lang.String`.

**Step 3: Execution Engine** runs the bytecode using two techniques:
- **Interpreter**: reads bytecode line by line and executes it. Starts fast, but slow for repeated code.
- **JIT (Just-In-Time) Compiler**: watches for **"hot" code** (methods or loops that run a lot) and compiles it into native machine code, which is cached and reused. This is why Java gets faster after a "warm-up."

Plus the **Garbage Collector** (Section 10) running in the background.

### Interview-ready answer

> "I write `.java` source. `javac` compiles it to platform-independent bytecode. The JVM's class loader loads classes, verifies them, and the execution engine runs them, interpreting at first and JIT-compiling hot code to native code. The JVM manages memory and garbage collects unused objects."

### Common questions
- **Is Java compiled or interpreted?** Both. Compiled to bytecode, then interpreted/JIT-compiled by the JVM.
- **Why is Java platform-independent but the JVM isn't?** Bytecode is universal; each OS has its own JVM implementation that understands it.
- **What is bytecode?** Instructions for the JVM (not for a real CPU). You can view it with `javap -c ClassName`.

---

## 2. JVM memory: stack vs heap

Think of the **heap** as a big shared warehouse and each thread's **stack** as that worker's personal desk.

```
┌───────────────────────── JVM Memory ─────────────────────────┐
│                                                              │
│  HEAP (shared by all threads)         METASPACE              │
│  - All objects live here              - Class info, method   │
│  - Garbage collected                    code, static metadata│
│                                                              │
│  Per-thread areas:                                           │
│  ┌─────────────┐ ┌─────────────┐   PC Register: which       │
│  │ Thread 1    │ │ Thread 2    │   bytecode instruction     │
│  │ STACK       │ │ STACK       │   this thread is on        │
│  │ (frames)    │ │ (frames)    │                            │
│  └─────────────┘ └─────────────┘                            │
└──────────────────────────────────────────────────────────────┘
```

| Area | Stores | Shared? | Cleaned by |
|------|--------|---------|-----------|
| **Stack** | Local variables, method call frames, references | One per thread | Auto, when the method returns |
| **Heap** | All objects and arrays | Shared | Garbage collector |
| **Metaspace** | Class metadata, static info (pre-Java 8 this was "PermGen") | Shared | Class unloading |
| **PC register** | Current instruction pointer | Per thread | n/a |

### Stack in detail
Every method call pushes a **frame** holding its local variables and parameters. When the method returns, the frame pops off. This is why recursion without an exit eventually throws **`StackOverflowError`**: too many frames.

### Heap in detail
`new` always allocates on the heap. The variable holding it is just a **reference** (like a remote control) that lives on the stack.

```java
void demo() {
    int x = 5;                    // x: stack (primitive value)
    Dog d = new Dog("Rex");       // d: stack (reference) ──> Dog object: heap
}
```

Running out of heap gives **`OutOfMemoryError`**.

---

## 3. How a basic program works (step by step)

```java
public class Main {
    static int square(int n) {          // (3)
        return n * n;
    }

    public static void main(String[] args) {   // (1) JVM starts here
        int a = 4;
        int result = square(a);          // (2)
        System.out.println(result);      // prints 16
    }
}
```

**What happens when you run `java Main`:**

1. JVM starts, loads the `Main` class (class loading, Section 1).
2. It looks for exactly `public static void main(String[] args)`.
   - `public`: JVM must be able to call it from outside.
   - `static`: JVM calls it *without* creating a `Main` object.
   - `void`: returns nothing to the JVM.
   - `String[] args`: command-line arguments.
3. A **stack frame** for `main` is created. `a = 4` is stored there.
4. `square(a)` is called: a new frame is pushed with `n = 4` (a **copy**). It computes 16, returns, and the frame pops.
5. `result = 16` is stored in main's frame. `println` prints it.
6. `main` returns, and when all non-daemon threads finish, the JVM exits.

### Order of initialization (a classic trick question)

For `new Child()`, the order is:

1. Parent **static** blocks, then Child **static** blocks (only the *first* time the class loads)
2. Parent **instance** initializers and fields
3. Parent **constructor**
4. Child instance initializers and fields
5. Child **constructor**

Rule of thumb: **the parent is always fully built before the child.**

---

## 4. Pass by value vs pass by reference

### The answer: **Java is ALWAYS pass by value.** Always.

What gets copied depends on the type:
- **Primitives**: the actual value is copied.
- **Objects**: the *reference* (the address) is copied, **not the object itself**.

### Analogy
You give a friend a **photocopy of your house address** (not your house).
- Your friend can walk to your house and repaint it (modify the object), and you'll see it.
- Your friend can scribble a new address on their photocopy (reassign the parameter). Your original address is unchanged.

### Example 1: primitives
```java
static void change(int x) { x = 99; }

int a = 5;
change(a);
System.out.println(a);   // 5  (only the copy changed)
```

### Example 2: objects, mutating the object (you DO see the change)
```java
static void rename(Dog d) { d.name = "Max"; }   // follows the copied address to the SAME object

Dog dog = new Dog("Rex");
rename(dog);
System.out.println(dog.name);  // "Max"
```

### Example 3: objects, reassigning the parameter (you do NOT see the change)
```java
static void replace(Dog d) { d = new Dog("Buddy"); }   // d now points elsewhere; caller's reference untouched

Dog dog = new Dog("Rex");
replace(dog);
System.out.println(dog.name);  // still "Rex"
```

```
Before replace():        Inside replace() after d = new Dog():
dog ───► [Rex]           dog ───► [Rex]
d   ───► [Rex]           d   ───► [Buddy]     (caller's `dog` still points to Rex)
```

### The test
> If Java were pass-by-reference, a classic `swap(a, b)` method would work. In Java it can't, because the method only swaps its own copies.

### Interview answer
> "Java is strictly pass by value. For objects, the value being copied is the reference, so a method can mutate the object the caller sees, but it cannot make the caller's variable point to a different object."

---

## 5. Primitives, wrappers, String, immutability

### Primitives (8 types, stored by value)
| Type | Size | Notes |
|------|------|-------|
| `byte` | 8-bit | -128 to 127 |
| `short` | 16-bit | |
| `int` | 32-bit | default integer type, about ±2.1 billion |
| `long` | 64-bit | suffix `L` |
| `float` | 32-bit | suffix `f` |
| `double` | 64-bit | default decimal type |
| `char` | 16-bit | one UTF-16 character |
| `boolean` | JVM-dependent | `true` / `false` |

### Wrapper classes (objects for primitives)
`Integer`, `Long`, `Double`, `Boolean`... Needed because collections (`List<Integer>`) can't hold primitives.
- **Autoboxing**: `int → Integer` automatically. **Unboxing**: the reverse.
- Danger: unboxing a `null` Integer throws `NullPointerException`.
- Trap: `Integer a = 127, b = 127; a == b` is `true` (cached, -128..127), but `Integer a = 1000, b = 1000; a == b` is `false`. **Use `.equals()` for objects.**

### `==` vs `.equals()`
- `==` compares **primitives' values** or **objects' references** (same object in memory?).
- `.equals()` compares **content** (if the class overrides it, as `String` does).

### String and the String Pool
Strings are **immutable**: once created, they never change. Every "modification" creates a new String.

```java
String a = "hi";               // goes to the String Pool
String b = "hi";               // reuses the same pooled object
String c = new String("hi");   // forces a NEW object on the heap

a == b        // true  (same pooled object)
a == c        // false (different objects)
a.equals(c)   // true  (same content)
```

**Why is String immutable?**
1. **Security**: file paths, DB URLs, and passwords can't be altered after validation.
2. **Thread-safe** for free.
3. **Pooling** is only safe if nobody can change a shared string.
4. **Hash caching**: the hash is computed once, so strings are great HashMap keys.

### String vs StringBuilder vs StringBuffer
| | String | StringBuilder | StringBuffer |
|--|--------|---------------|--------------|
| Mutable | No | Yes | Yes |
| Thread-safe | Yes (immutable) | No | Yes (synchronized) |
| Speed | Slow for repeated concat | Fastest | Slower than Builder |

Use `StringBuilder` when building strings in a loop (`s += x` in a loop creates a new String every iteration, which is O(n²)).

### Making your own immutable class
1. Make the class `final`. 2. Make all fields `private final`. 3. No setters. 4. Deep-copy any mutable fields in and out.

---

## 6. OOP: the four pillars

**Class** = blueprint. **Object** = a thing built from the blueprint.

### 6.1 Encapsulation: "hide the data, expose controlled access"

Keep fields `private`; expose them via public methods. The class stays in control of its own state.

```java
public class BankAccount {
    private double balance;                 // hidden

    public void deposit(double amt) {
        if (amt <= 0) throw new IllegalArgumentException("Must be positive");
        balance += amt;                     // validation lives in ONE place
    }
    public double getBalance() { return balance; }
}
```

**Why it matters:** nobody can set `balance = -1000000` directly. You can change the internals later without breaking users.
**Analogy:** a car. You use the pedals and wheel; you don't touch the engine.

**Access modifiers:**
| Modifier | Same class | Same package | Subclass | Everywhere |
|----------|:--:|:--:|:--:|:--:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *(default)* | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

### 6.2 Inheritance: "is-a" relationship, reuse code

A child class gets the parent's fields and methods using `extends`.

```java
class Animal {
    String name;
    void eat() { System.out.println(name + " eats"); }
}
class Dog extends Animal {          // Dog IS-A Animal
    void bark() { System.out.println("Woof"); }
}
```

Key facts:
- Java has **single inheritance for classes** (one parent only), which avoids the "diamond problem." A class can *implement many interfaces*.
- Everything ultimately extends **`Object`**.
- `super` refers to the parent (`super.eat()`, `super(args)` to call the parent constructor).
- Constructors are **not** inherited, but the parent constructor always runs first.
- `private` members exist in the child object but aren't accessible directly.

**Composition vs inheritance ("has-a" vs "is-a"):** prefer composition. A `Car` *has an* `Engine` (field), it doesn't extend it. Inheritance creates tight coupling; composition is more flexible. This is a very common interview talking point.

### 6.3 Polymorphism: "one interface, many forms"

**(a) Compile-time (static) polymorphism = method OVERLOADING**
Same name, different parameters, in the same class. The compiler picks which one.
```java
int add(int a, int b)       { return a + b; }
double add(double a, double b) { return a + b; }
int add(int a, int b, int c) { return a + b + c; }
```
(Changing only the return type is NOT valid overloading.)

**(b) Runtime (dynamic) polymorphism = method OVERRIDING**
A child provides its own version of a parent's method. The **JVM decides at runtime**, based on the *actual object type*, which one to run (dynamic dispatch).

```java
class Animal { void sound() { System.out.println("..."); } }
class Dog extends Animal { @Override void sound() { System.out.println("Woof"); } }
class Cat extends Animal { @Override void sound() { System.out.println("Meow"); } }

Animal a = new Dog();   // reference type: Animal, actual object: Dog
a.sound();              // "Woof"  ← decided at RUNTIME by the object, not the variable
```

This is why you can write `List<Animal>` and call `sound()` on each element without caring which animal it is.

**How dynamic dispatch works inside:** each class has a **vtable** (virtual method table), a lookup table of method addresses. The object points to its class's table, so calling `a.sound()` looks up `sound` in Dog's table.

**Overriding rules:**
- Same name, same parameters, return type same or a subtype (covariant).
- Can't reduce visibility (public → private is not allowed).
- Can't override `static`, `final`, or `private` methods. (Static methods get *hidden*, not overridden.)
- Can't throw broader *checked* exceptions than the parent.
- Use `@Override` so the compiler catches typos.

| Overloading | Overriding |
|-------------|-----------|
| Same class | Parent + child |
| Different parameters | Same signature |
| Resolved at compile time | Resolved at runtime |

**Upcasting / downcasting:**
```java
Animal a = new Dog();    // upcast: automatic and safe
Dog d = (Dog) a;         // downcast: explicit, can throw ClassCastException
if (a instanceof Dog dd) { dd.bark(); }   // safe check (pattern matching, Java 16+)
```

### 6.4 Abstraction: "show WHAT, hide HOW"

Expose only the essential idea; hide implementation details. Done with **abstract classes** and **interfaces**.

```java
abstract class Shape {                  // can't do new Shape()
    abstract double area();             // no body: children MUST implement
    void describe() { System.out.println("Area: " + area()); }   // concrete method is OK
}
class Circle extends Shape {
    double r;
    double area() { return Math.PI * r * r; }
}
```

**Abstract class vs Interface:**

| | Abstract class | Interface |
|--|----------------|-----------|
| Purpose | Shared base with some implementation ("is-a") | A contract/capability ("can-do") |
| Inherit from | Only **one** | **Many** |
| Fields | Any kind | Only `public static final` constants |
| Constructors | Yes | No |
| Methods | Abstract + concrete | Abstract, plus `default`, `static`, `private` (Java 8/9+) |
| Use when | Closely related classes share state/code | Unrelated classes share a behavior (`Comparable`, `Runnable`) |

```java
interface Flyable { void fly(); }
interface Swimmable { void swim(); }
class Duck implements Flyable, Swimmable { ... }   // multiple capabilities
```

### The four pillars in one breath
- **Encapsulation**: bundle data + methods, hide internals (`private` + getters).
- **Inheritance**: reuse code through "is-a" (`extends`).
- **Polymorphism**: same call, different behavior depending on the object (`@Override`).
- **Abstraction**: expose a simple contract, hide complexity (abstract classes and interfaces).

### SOLID in one line each (bonus, often asked)
- **S**ingle Responsibility: a class does one job.
- **O**pen/Closed: open to extension, closed to modification.
- **L**iskov Substitution: a child must be usable wherever the parent is expected.
- **I**nterface Segregation: many small interfaces beat one giant one.
- **D**ependency Inversion: depend on interfaces, not concrete classes.

---

## 7. Keywords: static, final, abstract, interface, and more

### `static`: "belongs to the CLASS, not to any object"

```java
class Counter {
    static int count = 0;     // ONE copy shared by ALL objects
    int id;                   // each object has its own
    Counter() { count++; id = count; }

    static void printCount() { System.out.println(count); }   // call: Counter.printCount()
}
```

| Where used | Meaning |
|-----------|---------|
| static **variable** | One shared copy, loaded when the class loads (lives in Metaspace/heap, not tied to an object) |
| static **method** | Callable without an object. **Cannot** use `this` or touch instance members directly |
| static **block** | Runs once when the class is loaded (init work) |
| static **nested class** | Doesn't need an outer-class instance |

Why is `main` static? So the JVM can call it before any object exists.
Static methods aren't polymorphic (they're resolved at compile time).

### `final`: "cannot change"

| On a... | Means |
|---------|-------|
| variable | Can be assigned only once (constant) |
| method | Cannot be overridden |
| class | Cannot be extended (`String`, `Integer` are final) |

**Common trap:** `final` on an object reference means the *reference* can't change, but the *object's contents* can.
```java
final List<String> list = new ArrayList<>();
list.add("ok");              // allowed: object is mutable
list = new ArrayList<>();    // ERROR: can't reassign the reference
```

**final vs finally vs finalize():**
- `final`: modifier (above).
- `finally`: block that always runs after try/catch.
- `finalize()`: old GC hook, deprecated. Don't use it.

### `abstract`
- On a class: can't be instantiated.
- On a method: no body; subclasses must implement.
- Can't combine `abstract` with `final` or `private` (it needs to be overridden).

### `interface` / `implements` / `extends`
- A class **extends** one class and **implements** many interfaces.
- An interface **extends** other interfaces.

### `this` and `super`
- `this`: the current object (`this.name = name;`, `this(...)` calls another constructor in the same class).
- `super`: the parent (`super.method()`, `super(...)`).
- Both must be the **first statement** in a constructor.

### `instanceof`
Checks the real type of an object at runtime.

### `volatile`, `synchronized`, `transient` (more in Section 13)
- `volatile`: reads/writes go to main memory (visibility across threads).
- `synchronized`: only one thread at a time (mutual exclusion).
- `transient`: skip this field during serialization.

### `var` (Java 10+)
Local type inference: `var list = new ArrayList<String>();`. Still statically typed.

### `enum`
A fixed set of constants; it's a full class (can have fields, methods, constructors). Good for states, days, types.

### `record` (Java 16+)
A compact immutable data class: `record Point(int x, int y) {}`. Auto-generates the constructor, getters, `equals`, `hashCode`, `toString`.

---

## 8. equals() and hashCode()

These two methods power every hash-based collection (`HashMap`, `HashSet`), so this is high-yield.

### The contract
1. If `a.equals(b)` is **true**, then `a.hashCode() == b.hashCode()` **must** be true.
2. If hashCodes are equal, the objects **may or may not** be equal (collisions happen).
3. `equals` must be reflexive, symmetric, transitive, and consistent.

### Why break the contract = bug
If you override `equals` but not `hashCode`, two "equal" objects land in **different buckets**, and `map.get(key)` can't find what you put in.

```java
class Point {
    final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }
    @Override public int hashCode() { return Objects.hash(x, y); }
}
```

**Defaults:** `Object.equals` is `==` (reference equality). `Object.hashCode` is typically derived from the memory identity.

**Tip:** use immutable objects as map keys. If a key mutates after insertion, its hash changes and the entry becomes unreachable.

---

## 9. Error handling (exceptions)

### The hierarchy

```
                    Throwable
                   /         \
               Error         Exception
        (JVM problems:        /          \
         don't catch)   RuntimeException   Other Exceptions
   OutOfMemoryError     (UNCHECKED)        (CHECKED)
   StackOverflowError   NullPointerException   IOException
                        ArrayIndexOutOfBounds   SQLException
                        IllegalArgumentException FileNotFoundException
                        ArithmeticException
                        ClassCastException
```

| Type | Checked at | Must handle? | Meaning | Examples |
|------|-----------|--------------|---------|----------|
| **Checked** | Compile time | **Yes** (catch or declare with `throws`) | Recoverable, external problems | `IOException`, `SQLException` |
| **Unchecked** (RuntimeException) | Not checked | No | Programmer bugs | `NullPointerException`, `IllegalArgumentException` |
| **Error** | Not checked | No (don't) | Serious JVM problems | `OutOfMemoryError`, `StackOverflowError` |

### try / catch / finally

```java
try {
    int result = 10 / divisor;
    readFile();
} catch (ArithmeticException e) {
    System.out.println("Can't divide by zero");
} catch (IOException e) {            // order: specific first, general last
    System.out.println("File problem: " + e.getMessage());
} finally {
    cleanUp();                        // ALWAYS runs (even with return/exception)
}
```

`finally` runs almost always. The exceptions: `System.exit()`, a JVM crash, or an infinite loop.

### try-with-resources (best practice for files, DB connections)

```java
try (BufferedReader br = new BufferedReader(new FileReader("a.txt"))) {
    return br.readLine();
}   // br.close() is called automatically, even on exceptions
```
Works for anything implementing `AutoCloseable`. This replaces the old "close in finally" pattern.

### throw vs throws
- `throw new IllegalStateException("bad")`: actually *throws* an exception object.
- `throws IOException` in a method signature: *declares* that it may throw.

### Custom exceptions
```java
class InsufficientFundsException extends Exception {          // checked
    InsufficientFundsException(String msg) { super(msg); }
}
```
Extend `Exception` for checked, `RuntimeException` for unchecked.

### Best practices (say these in the interview)
- Catch **specific** exceptions, not `Exception` or `Throwable`.
- Never swallow exceptions silently (`catch (Exception e) {}` is a bug factory). Log or rethrow.
- Preserve the cause: `throw new MyException("msg", e);`
- Use exceptions for **exceptional** situations, not normal control flow (they're slow).
- Fail fast: validate inputs early.
- Use try-with-resources for anything closeable.

### Classic gotchas
- A `return` in `finally` overrides a `return` in `try` (and swallows exceptions). Never do it.
- Multi-catch: `catch (IOException | SQLException e)`.
- `StackOverflowError` is an Error, not an Exception.

---

## 10. Garbage collection

### The idea
In C/C++ you free memory manually. In Java, the **garbage collector (GC)** automatically frees objects that **can no longer be reached**.

### How does GC know something is garbage? Reachability
Start from **GC roots** (local variables on stacks, static variables, active threads, JNI references). Anything you can reach by following references from roots is **alive**. Everything else is garbage.

```
 Stack: obj1 ──► [A] ──► [B]          [C] ──► [D]     ← no root reaches C or D
                                       (garbage, even though C and D reference each other)
```

Note: Java does **not** use simple reference counting, so circular references (C ↔ D) are still collected.

### Generational heap: "most objects die young"

```
┌──────────────── HEAP ───────────────────────────────┐
│  YOUNG GENERATION              │  OLD GENERATION    │
│  ┌──────┬────────┬────────┐   │                    │
│  │ Eden │ Surv 0 │ Surv 1 │   │  Long-lived objects│
│  └──────┴────────┴────────┘   │                    │
└─────────────────────────────────────────────────────┘
```

1. New objects start in **Eden**.
2. When Eden fills, a **Minor GC** runs: live objects are copied to a **Survivor** space; dead ones are dropped. It's fast.
3. Each survival increases the object's **age**. After enough survivals (a threshold), the object is **promoted** to the **Old Generation**.
4. When Old fills, a **Major / Full GC** runs. It's slower and can cause a longer pause.

Why this design? Studies show most objects die quickly (temp variables, short-lived objects), so scanning only the small young area is cheap and efficient.

### Basic algorithm: Mark → Sweep → Compact
1. **Mark**: trace from roots, flagging live objects.
2. **Sweep**: reclaim memory of unmarked objects.
3. **Compact** (optional): slide survivors together to remove fragmentation.

### Stop-the-world (STW)
During some GC phases, application threads **pause**. Modern collectors try to minimize these pauses.

### Common collectors
| Collector | Characteristic |
|-----------|----------------|
| **Serial** | Single-threaded; small apps |
| **Parallel** | Multi-threaded; maximizes throughput |
| **G1** (default since Java 9) | Splits heap into regions; balances pause time and throughput |
| **ZGC / Shenandoah** | Very low pause times, even with huge heaps |

### Can you force GC?
`System.gc()` is only a *suggestion*. The JVM may ignore it. Don't rely on it.

### Memory leaks in Java? Yes!
GC only frees *unreachable* objects. A leak happens when you keep references you no longer need:
- A `static` collection that keeps growing.
- Listeners/callbacks never removed.
- Caches without eviction.
- Unclosed resources (connections, streams).

### Reference types (bonus)
- **Strong** (normal): never collected while reachable.
- **Soft**: collected only when memory is low (caches).
- **Weak**: collected at the next GC (`WeakHashMap`).
- **Phantom**: used for cleanup tracking.

### Interview answer
> "GC automatically reclaims objects not reachable from GC roots. The heap is generational: new objects go to Eden, survivors get promoted to Old gen. Minor GCs are quick and frequent, major GCs are rarer. The default collector, G1, divides the heap into regions to keep pauses short."

---

## 11. Data structures: how they work inside

### Big-O quick reference (memorize)

| Structure | Access by index | Search | Insert | Delete | Notes |
|-----------|:-:|:-:|:-:|:-:|-------|
| **Array** | O(1) | O(n) | O(n) | O(n) | Fixed size |
| **ArrayList** | O(1) | O(n) | O(1) amortized at end, O(n) middle | O(n) | Resizable array |
| **LinkedList** | O(n) | O(n) | O(1) at ends / with node | O(1) with node | Doubly linked |
| **HashMap / HashSet** | n/a | O(1) avg | O(1) avg | O(1) avg | Worst O(log n) (Java 8+) |
| **TreeMap / TreeSet** | n/a | O(log n) | O(log n) | O(log n) | Sorted |
| **PriorityQueue** | peek O(1) | O(n) | O(log n) | O(log n) | Binary heap |
| **ArrayDeque** | n/a | O(n) | O(1) at ends | O(1) at ends | Stack/queue |

### The Collections hierarchy

```
Iterable
  └─ Collection
       ├─ List   (ordered, duplicates OK):  ArrayList, LinkedList, Vector
       ├─ Set    (no duplicates):           HashSet, LinkedHashSet, TreeSet
       └─ Queue  (FIFO-ish):                LinkedList, PriorityQueue, ArrayDeque

Map (separate; key → value):   HashMap, LinkedHashMap, TreeMap, ConcurrentHashMap
```

### 11.1 Array: the foundation
A **contiguous block of memory** with fixed size.

```
index:   0    1    2    3
       [ 10 | 20 | 30 | 40 ]    address of arr[i] = start + i × elementSize
```
- **O(1) access**: the address is computed directly by math, no searching.
- Fixed size. To grow, you must make a new array and copy.
- Insert/delete in the middle means shifting elements: O(n).
- Default values: `0`, `false`, `null`. Arrays are objects (live on the heap).

### 11.2 ArrayList: a resizable array
Internally holds an `Object[]` plus a `size` counter.

**How `add()` works:**
1. If the array has room, put the element at `size` and do `size++`. O(1).
2. If full, **allocate a bigger array** (about **1.5× the old capacity**), **copy** everything over, then add.

```
capacity 4: [a|b|c|d] full → add(e) → new array capacity 6: [a|b|c|d|e|_] (copied)
```
- The expensive resize is rare, so adding is **amortized O(1)**.
- Default initial capacity is 10, allocated lazily on first add.
- `add(index, x)` and `remove(index)` shift elements: O(n).
- `get(i)` is O(1).
- Tip: `new ArrayList<>(expectedSize)` avoids repeated resizing.

### 11.3 LinkedList: nodes with pointers
A **doubly linked list**: each node holds `prev`, `data`, `next`.

```
null ◄─ [prev|A|next] ⇄ [prev|B|next] ⇄ [prev|C|next] ─► null
         ▲head                                ▲tail
```
- Insert/remove at head/tail: O(1).
- Insert/remove at a *known node*: O(1) (just re-wire pointers).
- `get(i)`: O(n), because it must walk from the head or tail.
- More memory per element (two pointers + object overhead), and poor CPU cache behavior since nodes are scattered in memory.

**ArrayList vs LinkedList (interview favorite):**
In practice **ArrayList wins almost always**, thanks to cache-friendly contiguous memory. LinkedList only helps with many insert/removes at the ends via an iterator, and `ArrayDeque` usually beats it even there.

### 11.4 HashMap: the most-asked structure

**Goal:** store `key → value` with O(1) average lookup.

**Inside:** an array of **buckets** (`Node<K,V>[] table`). Default size is **16**.

```
index  bucket
 0     null
 1     [key=“cat”,val=1] → [key=“act”,val=7]   ← collision: chained as linked list
 2     null
 3     [key=“dog”,val=2]
 ...
15     null
```

**`put(key, value)` step by step:**
1. Compute `hash = key.hashCode()` (then mixed with a bit-spread: `h ^ (h >>> 16)`).
2. Find the bucket: `index = hash & (capacity - 1)` (capacity is always a power of 2, so this is a fast "modulo").
3. If the bucket is empty, place a new node there.
4. If not empty (**collision**), walk the chain: if a key with `.equals()` match is found, **replace its value**; otherwise **append** a new node.
5. If `size > capacity × loadFactor` (load factor **0.75**), **resize**: double the capacity and **rehash** all entries.

**`get(key)`:** hash → bucket index → compare keys with `.equals()` along the chain.

**Why both hashCode and equals?** `hashCode` finds the *bucket* (fast, narrows down). `equals` finds the *exact key* within the bucket.

**Java 8 improvement:** if a single bucket's chain grows past **8** nodes (and the table has ≥ 64 slots), the chain is converted to a **balanced red-black tree**. So worst-case lookup drops from O(n) to **O(log n)**. It converts back when it shrinks.

**Other facts:**
- Allows **one `null` key** and many `null` values.
- **Not thread-safe**; **not ordered**.
- Iteration order may change after resize.
- Use `ConcurrentHashMap` for multi-threaded use (Section 13).
- Resize is expensive: O(n). Pre-size if you know the count.

**Variants:**
| Map | Ordering | Underlying |
|-----|----------|-----------|
| `HashMap` | None | Hash table |
| `LinkedHashMap` | Insertion (or access) order | Hash table + doubly linked list. Can build an LRU cache |
| `TreeMap` | Sorted by key | Red-black tree, O(log n) |
| `Hashtable` | None | Legacy, synchronized, no nulls. Avoid |
| `ConcurrentHashMap` | None | Thread-safe, fine-grained locking/CAS |

### 11.5 HashSet: a HashMap wearing a disguise
`HashSet<E>` internally is a `HashMap<E, DUMMY_OBJECT>`. Elements are the **keys**; the value is a constant placeholder. So it inherits everything: O(1) average, no duplicates (via `hashCode` + `equals`), no ordering.

- `LinkedHashSet`: keeps insertion order.
- `TreeSet`: sorted, backed by a `TreeMap` (O(log n)).

`set.add(x)` returns `false` if the element was already present.

### 11.6 Other must-knows
- **PriorityQueue**: a **binary min-heap** stored in an array. `peek` O(1); `add`/`poll` O(log n) via "bubble up/down." Great for "top K" problems.
- **ArrayDeque**: a resizable circular array. Use it for both **stack** and **queue** (faster than `Stack` and `LinkedList`).
- **Stack** (class): legacy, synchronized. Prefer `Deque<Integer> stack = new ArrayDeque<>();`.
- **Vector**: legacy synchronized ArrayList. Avoid.

### Fail-fast iterators
Modifying a list while looping with for-each throws **`ConcurrentModificationException`**. Internally a `modCount` is checked. Fix: use `Iterator.remove()`, `removeIf()`, or iterate over a copy.

```java
list.removeIf(x -> x < 0);   // safe way to remove while "iterating"
```

### Comparable vs Comparator
- `Comparable<T>`: the class defines its **natural order** (`compareTo`). One order only.
- `Comparator<T>`: a **separate** comparison rule (`compare`). Many orders possible.
```java
people.sort(Comparator.comparing(Person::getAge).thenComparing(Person::getName));
```

### Choosing the right structure
| Need | Use |
|------|-----|
| Fast lookup by key | `HashMap` |
| Unique items | `HashSet` |
| Sorted keys / range queries | `TreeMap` / `TreeSet` |
| Index access, general list | `ArrayList` |
| Stack or queue | `ArrayDeque` |
| Always get min/max next | `PriorityQueue` |
| Preserve insertion order + fast lookup | `LinkedHashMap` |
| Thread-safe map | `ConcurrentHashMap` |

---

## 12. Files and processing large files

### The basic I/O model: streams
A **stream** is a flow of data in one direction, like a pipe.

| | Bytes (binary) | Characters (text) |
|--|---------------|------------------|
| Input | `InputStream` (`FileInputStream`) | `Reader` (`FileReader`) |
| Output | `OutputStream` (`FileOutputStream`) | `Writer` (`FileWriter`) |

### Buffering: the #1 performance idea
Reading from disk byte by byte is extremely slow (each call is an expensive OS request). A **buffer** reads a big **chunk** into memory, then serves your small reads from memory.

```java
// Slow-ish: unbuffered
// Fast: wrapped with a buffer
try (BufferedReader br = new BufferedReader(new FileReader("big.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        process(line);        // only ONE line in memory at a time
    }
}
```

### Modern API: java.nio.file (`Files`, `Path`)
```java
Path p = Path.of("data.txt");

String all = Files.readString(p);               // OK ONLY for small files
List<String> lines = Files.readAllLines(p);     // loads EVERYTHING into memory

try (Stream<String> s = Files.lines(p)) {       // LAZY: streams line by line
    s.filter(l -> l.contains("ERROR")).forEach(System.out::println);
}
```

### How to process a LARGE file (say 50 GB) without running out of memory

**Golden rule: never load the whole file. Process it in pieces.**

1. **Stream line by line** (`BufferedReader` or `Files.lines`). Memory use is constant, no matter how big the file.
2. **Read in fixed-size chunks** (e.g., 8 KB-1 MB byte buffers) for binary data.
3. **Keep only aggregates**, not raw data (running count, sum, a `Map` of counts, a top-K heap).
4. **Memory-mapped files (`MappedByteBuffer`)**: the OS maps file regions into memory, and the OS pages data in/out on demand. Very fast for random access.
   ```java
   try (FileChannel ch = FileChannel.open(path, StandardOpenOption.READ)) {
       MappedByteBuffer buf = ch.map(FileChannel.MapMode.READ_ONLY, 0, ch.size());
   }
   ```
   (A single mapping is limited to about 2 GB, so map in segments.)
5. **Parallelize**: split the file into byte ranges (aligned to line boundaries), process each on a different thread, then merge results (map-reduce style).
6. **External sorting** (when sorting data bigger than RAM):
   - Read chunks that fit in memory, sort each, and write each to a temp file.
   - **K-way merge** the sorted chunk files using a min-heap (`PriorityQueue`).
7. **Use compression/streaming formats** (`GZIPInputStream`) and efficient formats (e.g., Parquet, CSV streaming) when appropriate.
8. **At bigger scale**, use distributed tools (Hadoop/Spark) or load into a database.

### Heap-friendly patterns by task
| Task | Approach |
|------|---------|
| Count word frequencies | Stream lines, update a `HashMap` (fine unless there are billions of unique words) |
| Find top K items | Stream + a size-K min-heap |
| Check "does X exist" for huge data | Bloom filter / hash set / DB index |
| Sort a file bigger than RAM | External merge sort |
| Dedupe huge file | Hash-partition into many smaller files, dedupe each |

### NIO channels vs classic IO (bonus)
- Classic IO: stream-oriented, **blocking**.
- NIO: **buffer + channel** oriented, supports non-blocking I/O via `Selector`. Useful for servers handling many connections.

### Other file facts
- `File` (old) vs `Path`/`Files` (modern). Prefer the latter.
- Always close resources (try-with-resources).
- **Serialization**: turning objects into bytes (`Serializable`). `transient` fields are skipped; `serialVersionUID` guards version compatibility. Modern code often prefers JSON or Protobuf.
- Specify the charset explicitly (`StandardCharsets.UTF_8`) to avoid platform surprises.

---

## 13. Multithreading

### Process vs thread
- **Process**: an independent running program with its own memory.
- **Thread**: a lightweight path of execution *inside* a process. Threads **share the heap** but each has its **own stack**.

**Why use threads?** Do work in parallel (multi-core), keep UIs/servers responsive, and overlap waiting (I/O) with computing.
**The cost:** shared data means bugs.

### Creating threads (3 ways)

```java
// 1. Extend Thread (least preferred: uses up your single inheritance)
class MyThread extends Thread {
    public void run() { System.out.println("Hi from " + getName()); }
}
new MyThread().start();

// 2. Implement Runnable (better) / lambda
Runnable task = () -> System.out.println("Hi from " + Thread.currentThread().getName());
Thread t = new Thread(task);
t.start();          // NOT t.run()!

// 3. Callable + ExecutorService (best: returns a result, can throw)
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> f = pool.submit(() -> 21 * 2);
System.out.println(f.get());     // 42  (blocks until ready)
pool.shutdown();
```

**`start()` vs `run()`:** `start()` creates a **new thread** that calls `run()`. Calling `run()` directly just runs it on the *current* thread, with no concurrency.

### Thread lifecycle
```
NEW ──start()──► RUNNABLE ◄──► RUNNING
                    │  ▲
          wait/lock │  │ notify/lock acquired
                    ▼  │
        BLOCKED / WAITING / TIMED_WAITING ──► TERMINATED (after run() ends)
```

### The core problem: race conditions

```java
class Counter {
    int count = 0;
    void inc() { count++; }    // NOT atomic! It's 3 steps: read, add 1, write
}
```
Two threads can both read `5`, both write `6`, so one increment is lost. That's a **race condition**.

### Fix 1: `synchronized` (mutual exclusion)
Only one thread at a time can hold an object's **lock (monitor)**.

```java
synchronized void inc() { count++; }          // locks on `this`

void inc2() {
    synchronized (lockObject) { count++; }    // finer-grained block
}
```
- Also guarantees **visibility**: changes made inside become visible to the next thread that takes the lock.
- `static synchronized` locks on the Class object.

### Fix 2: Atomic classes (lock-free)
```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();      // atomic, uses CPU compare-and-swap (CAS)
```

### Fix 3: `ReentrantLock` (more control)
```java
Lock lock = new ReentrantLock();
lock.lock();
try { count++; }
finally { lock.unlock(); }    // ALWAYS unlock in finally
```
Extras vs `synchronized`: `tryLock()`, timeouts, fairness, multiple conditions.

### `volatile`: visibility only
Each CPU core caches variables. A `volatile` field forces reads/writes to go to main memory, so all threads see the latest value. It does **not** make `count++` atomic. Good for simple flags:
```java
volatile boolean running = true;   // one thread sets false; others see it
```

### Java Memory Model: "happens-before" (the idea)
Without synchronization, the compiler/CPU may reorder instructions and threads may see stale data. `synchronized`, `volatile`, `Lock`, and `Thread.join()` create **happens-before** guarantees: what one thread did is visible to the other.

### Deadlock
Two threads each hold a lock the other needs, so both wait forever.

```
Thread 1: holds A, waits for B
Thread 2: holds B, waits for A     → stuck forever
```
**Four conditions (all needed):** mutual exclusion, hold-and-wait, no preemption, circular wait.
**Prevention:** always acquire locks in the **same global order**; use `tryLock` with timeout; keep critical sections small; avoid nested locks.

Other problems: **livelock** (threads keep reacting to each other without progress), **starvation** (a thread never gets a turn).

### Thread coordination
- `join()`: wait for a thread to finish.
- `wait()` / `notify()` / `notifyAll()`: inside `synchronized`, one thread waits for a condition, another signals it. (Producer-consumer.)
- `BlockingQueue`: the easy, safe producer-consumer tool (`put` blocks when full, `take` blocks when empty).
- `CountDownLatch`, `CyclicBarrier`, `Semaphore`: coordination helpers.
- `sleep(ms)` pauses the thread but **keeps** its locks; `wait()` **releases** the lock.

### Producer-consumer (basic code)
```java
BlockingQueue<Integer> q = new ArrayBlockingQueue<>(5);

Thread producer = new Thread(() -> {
    try { for (int i = 0; i < 10; i++) { q.put(i); System.out.println("Produced " + i); } }
    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});
Thread consumer = new Thread(() -> {
    try { for (int i = 0; i < 10; i++) { System.out.println("Consumed " + q.take()); } }
    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
});
producer.start(); consumer.start();
```

### Thread pools (don't create raw threads in production)
Creating threads is expensive. A **pool** reuses a fixed set of worker threads that pull tasks from a queue.
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
for (int i = 0; i < 10; i++) {
    int id = i;
    pool.submit(() -> System.out.println("Task " + id + " on " + Thread.currentThread().getName()));
}
pool.shutdown();                       // stop accepting; finish queued tasks
pool.awaitTermination(1, TimeUnit.MINUTES);
```

### CompletableFuture (async pipelines)
```java
CompletableFuture.supplyAsync(() -> fetchUser())
    .thenApply(user -> user.getName())
    .thenAccept(System.out::println);
```

### Thread-safe collections
- `ConcurrentHashMap`: thread-safe, high concurrency (lock striping/CAS on buckets, not one big lock).
- `CopyOnWriteArrayList`: good for read-heavy; copies the array on each write.
- `Collections.synchronizedList(...)`: wraps with one lock (coarse).
- `Vector`/`Hashtable`: legacy, avoid.

### Thread-safety tools: pick the right one
| Situation | Use |
|-----------|-----|
| Simple counter | `AtomicInteger` |
| Visibility flag | `volatile` |
| Protect a block of logic | `synchronized` / `ReentrantLock` |
| Shared map | `ConcurrentHashMap` |
| Hand data between threads | `BlockingQueue` |
| Avoid sharing altogether | Immutable objects, `ThreadLocal`, confinement |

### Handy facts
- **Daemon threads** (background) don't stop the JVM from exiting.
- **Interrupt**: `t.interrupt()` politely asks a thread to stop; the thread must check/handle it (`InterruptedException`).
- **Virtual threads (Java 21)**: very lightweight threads managed by the JVM, for handling huge numbers of blocking tasks cheaply.
- **Concurrency vs parallelism**: *concurrency* = dealing with many things at once (interleaving); *parallelism* = actually running at the same instant on multiple cores.

---

## 14. Database indexing

### The problem
`SELECT * FROM users WHERE email = 'a@b.com'` on 10 million rows with no index means a **full table scan**: check every row, O(n).

### The idea: an index is like a book's index
Instead of reading every page to find "polymorphism," you look in the sorted index at the back, which points to the page numbers. A DB index is a **separate, sorted data structure** that maps column values to row locations.

### What's inside: B-Tree / B+ Tree
Most relational DBs (MySQL InnoDB, PostgreSQL, Oracle) use a **B+ tree**: a balanced, shallow, wide tree.

```
                   [ 30 | 60 ]                   ← root
                  /     |     \
        [10|20]    [40|50]     [70|80]           ← internal nodes (just routing)
        /  |  \     /  |  \     /  |  \
      leaf leaf leaf ...                          ← leaves hold keys + row pointers
      ↔ ↔ ↔ linked together for fast range scans
```
- Each node is sized to fit a **disk page**, so each step costs about one disk read.
- Tree height stays tiny (3-4 levels can index millions or billions of rows), so lookups are **O(log n)** with very few disk reads.
- Leaves are **linked**, so range queries (`BETWEEN`, `>`, `ORDER BY`) just scan along the leaves.

### Types of indexes

| Type | What it is |
|------|-----------|
| **Primary key index** | Unique, not null; identifies each row. Created automatically |
| **Clustered index** | The table's data is *physically stored in the order of the index*. **Only one per table** (usually the primary key). In InnoDB, the leaf nodes *are* the rows |
| **Non-clustered (secondary) index** | A separate structure whose leaves point to the row (or to the primary key). **Many per table** |
| **Unique index** | Enforces no duplicates |
| **Composite (multi-column) index** | Index on several columns, e.g., `(last_name, first_name)` |
| **Covering index** | Contains all the columns a query needs, so the DB never touches the table |
| **Hash index** | Hash table; great for equality (`=`) only, no ranges |
| **Full-text index** | For searching words inside text |

### Composite indexes and the "leftmost prefix" rule
An index on `(A, B, C)` is sorted by A, then B within A, then C within B. It helps queries filtering on:
- `A`, ✅
- `A, B` ✅
- `A, B, C` ✅
- `B` alone ❌ (skips the leftmost column)
- `A, C` partially (uses only `A`)

Think of a phone book sorted by last name, then first name. You can't find a first name alone efficiently.

### Why not index everything?
Indexes aren't free:
- **Writes get slower** (INSERT/UPDATE/DELETE must update every index).
- **Extra storage.**
- More indexes can confuse the optimizer.

**Rule:** index columns used in `WHERE`, `JOIN ON`, `ORDER BY`, and `GROUP BY`, especially those with **high cardinality** (many distinct values). Indexing a `gender` or boolean column barely helps (low selectivity).

### When an index is NOT used (common pitfalls)
- Function on the column: `WHERE LOWER(email) = ...` or `WHERE YEAR(date) = 2024`.
- Leading wildcard: `LIKE '%abc'` (but `LIKE 'abc%'` works).
- Type mismatch / implicit casts.
- Violating the leftmost-prefix rule.
- Very low selectivity: the optimizer may decide a scan is cheaper.

### Checking it: `EXPLAIN`
`EXPLAIN SELECT ...` shows whether the DB does an index seek/scan or a full table scan. This is the first tool for query tuning.

### Indexing and Java
- In JDBC/JPA, performance often comes down to *good indexes* and avoiding the **N+1 query problem** (1 query for the parent list + N queries for children; fix with joins or batch fetching).
- The same idea shows up in code: a `HashMap` lookup is like a hash index, and a `TreeMap` is like a tree index.

### Interview answer
> "An index is a separate sorted structure, typically a B+ tree, that lets the database find rows in O(log n) instead of scanning the whole table. It speeds up reads on indexed columns but slows writes and uses storage, so I index columns used in filters, joins and sorting, and I verify with EXPLAIN."

(Related: **ACID** = Atomicity, Consistency, Isolation, Durability, the guarantees of a transaction. **Normalization** reduces redundancy. **Transaction isolation levels** trade correctness against concurrency.)

---

## 15. Generics, lambdas, streams (modern Java)

### Generics: type safety at compile time
```java
List<String> names = new ArrayList<>();
names.add("Ann");
// names.add(5);   // compile error: caught early, no ClassCastException later

static <T extends Comparable<T>> T max(T a, T b) { return a.compareTo(b) > 0 ? a : b; }
```
- **Type erasure**: generics exist only at compile time. At runtime `List<String>` and `List<Integer>` are both just `List`. So you can't do `new T()` or `instanceof List<String>`.
- Wildcards: `? extends T` (read-only "producer") and `? super T` (write-only "consumer"). Memory aid: **PECS**, Producer Extends, Consumer Super.

### Lambdas and functional interfaces
A **functional interface** has exactly one abstract method. A lambda is a short way to implement it.
```java
Runnable r = () -> System.out.println("hi");
Comparator<String> byLen = (a, b) -> a.length() - b.length();
Function<Integer,Integer> sq = x -> x * x;
```
Built-ins: `Predicate<T>` (T → boolean), `Function<T,R>`, `Consumer<T>` (T → void), `Supplier<T>` (() → T).

### Streams: declarative data pipelines
```java
List<String> result = names.stream()
    .filter(n -> n.length() > 3)       // intermediate (lazy)
    .map(String::toUpperCase)          // intermediate (lazy)
    .sorted()
    .collect(Collectors.toList());     // terminal (triggers execution)
```
- Streams are **lazy**: nothing runs until a terminal operation.
- Streams **don't modify** the source and can only be consumed once.
- `parallelStream()` splits work across cores (use carefully).

### Optional: avoid null checks
```java
Optional<String> o = Optional.ofNullable(maybeNull);
String v = o.map(String::trim).orElse("default");
```

---

## 16. Rapid-fire Q&A

**Q: Difference between JDK, JRE, JVM?**
JVM runs bytecode. JRE = JVM + libraries. JDK = JRE + dev tools (javac).

**Q: Is Java pass by value or reference?**
Always by value. For objects, the reference is copied.

**Q: Why is String immutable?**
Security, thread safety, string-pool sharing, and cached hash codes.

**Q: `==` vs `equals()`?**
`==` compares references (or primitive values); `equals()` compares logical content.

**Q: Overloading vs overriding?**
Overloading: same name, different params, compile-time. Overriding: same signature in a child class, runtime.

**Q: Abstract class vs interface?**
Abstract class: partial implementation, state, single inheritance. Interface: a contract, multiple inheritance of type, default methods.

**Q: Can you override a static or private method?**
No. Static methods are hidden, and private methods aren't visible to the child.

**Q: Can a constructor be inherited / overridden?**
No. It can be overloaded and chained via `super(...)`/`this(...)`.

**Q: What does `final` do on a class/method/variable?**
No subclassing / no overriding / no reassignment.

**Q: Checked vs unchecked exceptions?**
Checked are enforced by the compiler (`IOException`); unchecked are runtime bugs (`NullPointerException`).

**Q: Does `finally` always run?**
Almost always. Not after `System.exit()` or a JVM crash.

**Q: How does HashMap work internally?**
Array of buckets; `hashCode` picks the bucket, `equals` finds the key, collisions are chained (tree after 8 nodes), and it doubles and rehashes at 75% load.

**Q: HashMap vs Hashtable vs ConcurrentHashMap?**
HashMap: fast, not thread-safe, allows null. Hashtable: legacy, fully synchronized, no null. ConcurrentHashMap: thread-safe with fine-grained concurrency.

**Q: ArrayList vs LinkedList?**
ArrayList: array-backed, O(1) get, amortized O(1) append. LinkedList: O(1) at ends, O(n) get, more memory. ArrayList is the default choice.

**Q: How does garbage collection work?**
Reachability from GC roots; generational heap (Eden → Survivor → Old); mark-sweep-compact; G1 by default.

**Q: Can Java have memory leaks?**
Yes, via lingering references (static collections, unremoved listeners, unbounded caches).

**Q: `synchronized` vs `volatile`?**
`synchronized` gives mutual exclusion + visibility. `volatile` gives visibility only, no atomicity.

**Q: `wait()` vs `sleep()`?**
`wait()` releases the lock and needs `notify`; `sleep()` keeps the lock and just pauses.

**Q: What is a deadlock and how do you avoid it?**
Circular waiting on locks. Avoid it with consistent lock ordering, timeouts (`tryLock`), and minimal nested locking.

**Q: What's the difference between Thread `start()` and `run()`?**
`start()` spawns a new thread; `run()` is just a normal method call.

**Q: What is a database index?**
A sorted structure (usually B+ tree) that turns O(n) scans into O(log n) lookups, at the cost of slower writes and extra space.

**Q: Clustered vs non-clustered index?**
Clustered: defines the physical order of the data (one per table). Non-clustered: a separate lookup structure pointing to rows (many allowed).

**Q: How would you process a 100 GB file with 8 GB RAM?**
Stream it in chunks/lines, keep only aggregates, parallelize by splitting ranges, and use external merge sort if sorting is needed.

---

## 17. One-page cheat sheet

**Execution:** `.java` → (javac) → `.class` bytecode → (JVM: class loader → interpreter + JIT) → runs.
**Memory:** Stack (per-thread: locals, frames) | Heap (shared: objects, GC'd) | Metaspace (class info).
**Parameters:** Always pass by value (object references are copied).
**String:** Immutable, pooled. `StringBuilder` for loops.
**OOP:** Encapsulation (private + getters), Inheritance (`extends`, is-a), Polymorphism (overload = compile time, override = runtime), Abstraction (abstract/interface).
**Keywords:** `static` = class-level. `final` = unchangeable. `abstract` = incomplete, must be extended.
**equals/hashCode:** Equal objects ⇒ equal hash codes. Override both or neither.
**Exceptions:** Throwable → Error | Exception → (checked) | RuntimeException (unchecked). Use try-with-resources.
**GC:** Reachability from roots. Eden → Survivor → Old. Minor (fast) vs Major (slow). G1 default.
**ArrayList:** Array, grows ×1.5, amortized O(1) add.
**LinkedList:** Doubly linked, O(1) at ends, O(n) get.
**HashMap:** Buckets (16), `hash & (n-1)`, chain → tree at 8, resize ×2 at 0.75 load.
**HashSet:** HashMap with dummy values.
**Big files:** Stream/buffer/chunk, keep aggregates, parallelize, external sort.
**Threads:** `Runnable`/`ExecutorService`. Race conditions → `synchronized`/atomics/locks. `volatile` = visibility. Deadlock = circular wait → lock ordering.
**DB index:** B+ tree, O(log n), speeds reads, slows writes, leftmost-prefix rule, check with `EXPLAIN`.

---

### How to practice
1. **Say each section out loud** in 60 seconds as if explaining to a teammate.
2. **Code from memory:** a mini `HashMap` (array + linked nodes), a producer-consumer with `BlockingQueue`, a custom exception, and an immutable class.
3. **Trace by hand:** draw the stack/heap for the pass-by-value examples and the initialization order for a parent/child.
4. **Be ready for "why?" follow-ups:** why is HashMap capacity a power of 2? Why prefer composition? Why does the load factor default to 0.75 (a balance of space vs collisions)?
