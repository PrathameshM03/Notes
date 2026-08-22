# Core Java Complete Notes (Beginner → Advanced)
### Including Java 8 Features, Collections, Multithreading, Generics & JVM Internals

---

## Table of Contents
1. [Introduction & JVM/JRE/JDK](#1-introduction--jvmjrejdk)
2. [How Java Code Runs (JVM Internals)](#2-how-java-code-runs-jvm-internals)
3. [Data Types, Variables & Operators](#3-data-types-variables--operators)
4. [Control Flow](#4-control-flow)
5. [OOP Fundamentals](#5-oop-fundamentals)
6. [Inheritance, Polymorphism, Abstraction, Encapsulation](#6-inheritance-polymorphism-abstraction-encapsulation)
7. [Interfaces & Abstract Classes](#7-interfaces--abstract-classes)
8. [Arrays & Strings](#8-arrays--strings)
9. [Exception Handling](#9-exception-handling)
10. [Collections Framework](#10-collections-framework)
11. [Generics](#11-generics)
12. [Multithreading & Concurrency](#12-multithreading--concurrency)
13. [Java 8 Features (Complete)](#13-java-8-features-complete)
14. [Memory Management & Garbage Collection](#14-memory-management--garbage-collection)
15. [Nested & Inner Classes](#15-nested--inner-classes)
16. [Annotations & Reflection (Overview)](#16-annotations--reflection-overview)
17. [I/O & Serialization](#17-io--serialization)
18. [Common Interview Questions](#18-common-interview-questions)
19. [Cheat Sheet Summary](#19-cheat-sheet-summary)

---

## 1. Introduction & JVM/JRE/JDK

**Java** is a statically-typed, object-oriented, platform-independent language. "Write Once, Run Anywhere" (WORA) is achieved via the JVM.

```
   JDK (Java Development Kit)
   ├── JRE (Java Runtime Environment)
   │    ├── JVM (Java Virtual Machine)
   │    └── Core Libraries (java.lang, java.util, ...)
   └── Development Tools (javac, javadoc, jar, debugger)
```

| Component | Contains | Used for |
|---|---|---|
| **JDK** | JRE + compiler (`javac`) + dev tools | Writing & compiling Java code |
| **JRE** | JVM + libraries | Running Java applications |
| **JVM** | Bytecode interpreter/JIT compiler | Executing `.class` files |

### Compilation & Execution Flow
```
 MyClass.java (source code)
        │  javac (compiler)
        ▼
 MyClass.class (bytecode — platform-independent)
        │  java (launcher) → loads into JVM
        ▼
   JVM interprets/JIT-compiles bytecode
        │
        ▼
   Native machine code executed by OS/CPU
```
This is why the **same `.class` file runs on Windows, Linux, or Mac** — only the JVM implementation differs per platform, not your code.

---

## 2. How Java Code Runs (JVM Internals)

### JVM Architecture
```
┌─────────────────────────────────────────────────┐
│                      JVM                          │
│  ┌───────────────┐   ┌─────────────────────────┐ │
│  │ Class Loader   │   │   Runtime Data Areas     │ │
│  │  Subsystem     │──▶│  ┌─────────┐ ┌─────────┐│ │
│  └───────────────┘   │  │ Heap    │ │ Method   ││ │
│                        │  │(Objects)│ │  Area    ││ │
│                        │  └─────────┘ └─────────┘│ │
│                        │  ┌─────────┐ ┌─────────┐│ │
│                        │  │ Stack   │ │ PC       ││ │
│                        │  │(per     │ │ Register ││ │
│                        │  │ thread) │ │(per      ││ │
│                        │  └─────────┘ │ thread)  ││ │
│                        │              └─────────┘│ │
│                        └─────────────────────────┘ │
│  ┌────────────────────────────────────────────┐   │
│  │   Execution Engine (Interpreter + JIT +      │   │
│  │   Garbage Collector)                          │   │
│  └────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Class Loading — 3 Phases
```
1. Loading    → reads .class file, creates Class object
2. Linking    → Verify (bytecode valid?) → Prepare (default values for statics) → Resolve (symbolic refs → direct refs)
3. Initialization → static blocks and static variable initializers run
```

### Class Loaders (hierarchy, delegation model)
```
Bootstrap ClassLoader   (loads core java.* classes, written in native code)
        │
Extension/Platform ClassLoader   (loads javax.*, extension libs)
        │
Application/System ClassLoader   (loads your classpath classes)
```
Each loader **delegates upward first** — child asks parent before loading itself (prevents core classes from being overridden accidentally).

### Runtime Memory Areas
| Area | Purpose | Shared across threads? |
|---|---|---|
| **Heap** | All objects & instance variables live here | ✅ Yes (shared) |
| **Method Area (Metaspace in Java 8+)** | Class metadata, static variables, constant pool | ✅ Yes |
| **Stack** | Method call frames, local variables, references | ❌ No (one per thread) |
| **PC Register** | Address of currently executing instruction | ❌ No (one per thread) |
| **Native Method Stack** | For native (non-Java) method calls | ❌ No (one per thread) |

---

## 3. Data Types, Variables & Operators

### Primitive Data Types (8 total)
| Type | Size | Default | Example |
|---|---|---|---|
| `byte` | 1 byte | 0 | `byte b = 100;` |
| `short` | 2 bytes | 0 | `short s = 5000;` |
| `int` | 4 bytes | 0 | `int i = 100000;` |
| `long` | 8 bytes | 0L | `long l = 100000L;` |
| `float` | 4 bytes | 0.0f | `float f = 3.14f;` |
| `double` | 8 bytes | 0.0d | `double d = 3.14159;` |
| `char` | 2 bytes | '\u0000' | `char c = 'A';` |
| `boolean` | 1 bit (JVM-dependent) | false | `boolean flag = true;` |

### Reference Types
Everything else — `String`, arrays, custom classes — are **reference types**, stored on the heap, with variables holding a reference (pointer) to the object.

```java
int x = 10;              // primitive: stored directly
String s = "Hello";      // reference: s holds address of the String object on heap
```

### Type Casting
```java
// Widening (implicit, automatic) — smaller to bigger, safe
int i = 100;
long l = i;      // int → long, automatic

// Narrowing (explicit, manual) — bigger to smaller, may lose data
double d = 9.99;
int x = (int) d;  // 9 — decimal part truncated
```

### Operators
```java
// Arithmetic: + - * / %
// Relational: == != > < >= <=
// Logical: && || !
// Bitwise: & | ^ ~ << >> >>>
// Assignment: = += -= *= /=
// Ternary: condition ? valueIfTrue : valueIfFalse

int max = (a > b) ? a : b;
```

### `==` vs `.equals()`
```java
String s1 = new String("hello");
String s2 = new String("hello");

System.out.println(s1 == s2);        // false — different objects in memory
System.out.println(s1.equals(s2));   // true — same content

// String literals use the String Pool (interned) — reused automatically
String s3 = "hello";
String s4 = "hello";
System.out.println(s3 == s4);        // true — same pooled reference
```

---

## 4. Control Flow

```java
// if-else
if (score >= 90) {
    grade = 'A';
} else if (score >= 75) {
    grade = 'B';
} else {
    grade = 'C';
}

// switch (traditional)
switch (day) {
    case 1:
        System.out.println("Monday");
        break;      // without break, falls through to next case
    case 2:
        System.out.println("Tuesday");
        break;
    default:
        System.out.println("Unknown");
}

// switch expression (Java 14+, useful to know)
String dayName = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    default -> "Unknown";
};

// Loops
for (int i = 0; i < 5; i++) { System.out.println(i); }

int i = 0;
while (i < 5) { System.out.println(i); i++; }

int j = 0;
do { System.out.println(j); j++; } while (j < 5);   // runs at least once

// Enhanced for-loop (for-each)
int[] nums = {1, 2, 3};
for (int n : nums) { System.out.println(n); }

// break / continue
for (int k = 0; k < 10; k++) {
    if (k == 5) break;      // exits loop entirely
    if (k % 2 == 0) continue; // skips to next iteration
    System.out.println(k);
}
```

---

## 5. OOP Fundamentals

### Classes & Objects
```java
class Car {
    // fields (instance variables)
    String brand;
    int speed;

    // constructor
    Car(String brand, int speed) {
        this.brand = brand;  // 'this' refers to current instance
        this.speed = speed;
    }

    // method
    void accelerate() {
        speed += 10;
    }
}

// Creating an object (instance)
Car myCar = new Car("Toyota", 60);
myCar.accelerate();
```

### Object Creation Flow (Diagram)
```
   Car myCar = new Car("Toyota", 60);
          │
          ▼
   1. JVM allocates memory on HEAP for the object
   2. Fields set to default values (null/0)
   3. Constructor runs, assigns actual values
   4. Reference to the object stored in 'myCar' (on the STACK)

   Stack                     Heap
   ┌────────┐               ┌─────────────────┐
   │ myCar  │ ──────────▶  │ brand: "Toyota"   │
   │(ref)   │               │ speed: 60          │
   └────────┘               └─────────────────┘
```

### Constructors
```java
class Car {
    String brand;

    Car() {                          // default constructor
        this.brand = "Unknown";
    }

    Car(String brand) {              // parameterized constructor
        this.brand = brand;
    }

    Car(Car other) {                 // copy constructor (manual, Java has no built-in one)
        this.brand = other.brand;
    }
}
```

### Static vs Instance Members
```java
class Counter {
    static int totalCount = 0;   // shared across ALL instances (belongs to the class)
    int id;                       // unique per instance

    Counter() {
        totalCount++;
        id = totalCount;
    }
}

Counter c1 = new Counter();  // totalCount = 1, c1.id = 1
Counter c2 = new Counter();  // totalCount = 2, c2.id = 2
// Counter.totalCount == 2 (shared)
```

---

## 6. Inheritance, Polymorphism, Abstraction, Encapsulation

The **4 Pillars of OOP**:

```
┌───────────────┬──────────────────────────────────────────────┐
│ Encapsulation │ Bundling data + methods, hiding internals     │
│               │ (private fields + public getters/setters)    │
├───────────────┼──────────────────────────────────────────────┤
│ Abstraction   │ Showing only essential features, hiding       │
│               │ implementation details (abstract classes,    │
│               │ interfaces)                                   │
├───────────────┼──────────────────────────────────────────────┤
│ Inheritance   │ A class acquiring properties/behavior of      │
│               │ another class (extends)                       │
├───────────────┼──────────────────────────────────────────────┤
│ Polymorphism  │ One interface, many implementations            │
│               │ (method overriding & overloading)              │
└───────────────┴──────────────────────────────────────────────┘
```

### Encapsulation
```java
class BankAccount {
    private double balance;   // hidden from outside direct access

    public double getBalance() { return balance; }   // controlled read
    public void deposit(double amount) {              // controlled write
        if (amount > 0) balance += amount;
    }
}
```

### Inheritance
```java
class Animal {
    void eat() { System.out.println("Eating..."); }
}

class Dog extends Animal {
    void bark() { System.out.println("Barking..."); }
}

Dog d = new Dog();
d.eat();   // inherited from Animal
d.bark();  // own method
```

### Inheritance Diagram
```
           Animal
          (eat, sleep)
          /          \
       Dog             Cat
    (bark)           (meow)
```
> Java supports **single inheritance** for classes (one direct parent) but **multiple inheritance via interfaces**.

### Polymorphism — Overloading (compile-time) vs Overriding (runtime)
```java
// OVERLOADING — same method name, different parameters, SAME class
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
    int add(int a, int b, int c) { return a + b + c; }
}

// OVERRIDING — subclass redefines parent's method, SAME signature
class Animal {
    void sound() { System.out.println("Some sound"); }
}
class Dog extends Animal {
    @Override
    void sound() { System.out.println("Bark"); }   // @Override is optional but recommended
}

Animal a = new Dog();  // reference type Animal, actual object Dog
a.sound();              // "Bark" — decided at RUNTIME (dynamic dispatch)
```

### Overloading vs Overriding
| | Overloading | Overriding |
|---|---|---|
| Where | Same class (or subclass) | Parent-child classes |
| Signature | Must differ (params) | Must be identical |
| Binding | Compile-time (static) | Runtime (dynamic) |
| Return type | Can differ | Must be same or covariant |

### Abstraction
```java
abstract class Shape {
    abstract double area();     // no body — subclass MUST implement

    void describe() {           // concrete method — can be shared
        System.out.println("Area: " + area());
    }
}

class Circle extends Shape {
    double radius;
    Circle(double radius) { this.radius = radius; }

    @Override
    double area() { return Math.PI * radius * radius; }
}
```

---

## 7. Interfaces & Abstract Classes

```java
interface Drawable {
    void draw();                       // implicitly public abstract

    default void print() {             // default method (Java 8+) — has a body
        System.out.println("Printing...");
    }

    static void info() {               // static method (Java 8+)
        System.out.println("Drawable interface");
    }
}

class Square implements Drawable {
    @Override
    public void draw() { System.out.println("Drawing square"); }
}
```

### Interface vs Abstract Class
| | Interface | Abstract Class |
|---|---|---|
| Methods | Abstract + default + static (Java 8+) | Abstract + concrete |
| Fields | `public static final` only (constants) | Any type of field |
| Multiple inheritance | ✅ A class can implement many | ❌ Only one `extends` |
| Constructors | ❌ No | ✅ Yes |
| When to use | Define a **contract/capability** ("can do") | Share common code among closely related classes ("is a") |

### Multiple Interface Inheritance
```java
interface Flyable { void fly(); }
interface Swimmable { void swim(); }

class Duck implements Flyable, Swimmable {
    public void fly() { System.out.println("Duck flying"); }
    public void swim() { System.out.println("Duck swimming"); }
}
```

---

## 8. Arrays & Strings

### Arrays
```java
int[] nums = new int[5];              // fixed size, default 0s
int[] nums2 = {1, 2, 3, 4, 5};        // initialized directly

int[][] matrix = new int[3][3];        // 2D array
matrix[0][0] = 1;

// Arrays are OBJECTS in Java (stored on heap), fixed length once created
System.out.println(nums.length);       // property, not method (no parentheses)
```

### `Arrays` Utility Class
```java
import java.util.Arrays;

int[] arr = {5, 3, 1, 4, 2};
Arrays.sort(arr);                          // [1, 2, 3, 4, 5]
System.out.println(Arrays.toString(arr));  // readable print
int idx = Arrays.binarySearch(arr, 3);     // must be sorted first
int[] copy = Arrays.copyOf(arr, 3);        // [1, 2, 3]
```

### Strings — Immutability
```java
String s = "Hello";
s.concat(" World");     // does NOT change s — creates a NEW String
System.out.println(s);  // still "Hello"

s = s.concat(" World"); // must reassign to capture the new String
System.out.println(s);  // "Hello World"
```
> Strings are **immutable** — every "modification" creates a new object. This is why concatenating in a loop with `+` is inefficient — use `StringBuilder` instead.

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 5; i++) {
    sb.append(i).append(",");   // mutable — no new object each time
}
System.out.println(sb.toString());   // "0,1,2,3,4,"
```

### Common String Methods
```java
String s = "  Hello World  ";
s.length();            // 15
s.trim();               // "Hello World" (removes leading/trailing whitespace)
s.toUpperCase();        // "  HELLO WORLD  "
s.substring(2, 7);      // "Hello"
s.indexOf("World");     // position of first match
s.split(" ");           // array split by delimiter
s.replace("Hello", "Hi"); // replaces text
s.charAt(0);             // character at index
```

---

## 9. Exception Handling

### Exception Hierarchy
```
                    Throwable
                   /          \
              Error            Exception
          (OutOfMemory,      /            \
           StackOverflow —  Checked        Unchecked (RuntimeException)
           unrecoverable)   (IOException,   (NullPointerException,
                             SQLException — ArrayIndexOutOfBounds,
                             must handle    ArithmeticException —
                             or declare)    programmer errors)
```

### try-catch-finally
```java
try {
    int result = 10 / 0;         // throws ArithmeticException
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero: " + e.getMessage());
} finally {
    System.out.println("This always runs — cleanup code (closing files, connections)");
}
```

### Multi-catch & Catching Multiple Exception Types
```java
try {
    // risky code
} catch (IOException | SQLException e) {   // Java 7+ multi-catch
    System.out.println("Error: " + e.getMessage());
} catch (Exception e) {                     // generic fallback — always LAST
    System.out.println("Unknown error");
}
```

### Checked vs Unchecked
| | Checked | Unchecked |
|---|---|---|
| Checked at | Compile-time | Runtime |
| Must handle/declare? | ✅ Yes (`throws` or try-catch) | ❌ No (optional) |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `ArrayIndexOutOfBoundsException`, `ArithmeticException` |
| Represents | Recoverable external conditions | Programming bugs |

### `throw` vs `throws`
```java
// throw — actually throwing an exception instance
void withdraw(double amount) {
    if (amount > balance) {
        throw new IllegalArgumentException("Insufficient funds");
    }
}

// throws — declaring a method MIGHT throw a checked exception (in the signature)
void readFile() throws IOException {
    // ...
}
```

### Custom Exceptions
```java
class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String message) {
        super(message);
    }
}

void withdraw(double amount) throws InsufficientFundsException {
    if (amount > balance) {
        throw new InsufficientFundsException("Not enough balance");
    }
}
```

### try-with-resources (Java 7+, auto-closes resources)
```java
try (FileReader fr = new FileReader("file.txt");
     BufferedReader br = new BufferedReader(fr)) {
    System.out.println(br.readLine());
} catch (IOException e) {
    e.printStackTrace();
}
// fr and br are automatically closed, even if an exception occurs
```

---

## 10. Collections Framework

### Hierarchy Diagram
```
                          Iterable
                              │
                          Collection
             ┌────────────────┼────────────────┐
           List              Set              Queue
        (ordered,       (no duplicates)    (FIFO processing)
        duplicates OK)        │                  │
        │    │    │      ┌────┼────┐        ┌────┴────┐
     ArrayList │ Vector  Hash  Linked  Tree  PriorityQueue  Deque
        LinkedList       Set    HashSet Set                 ArrayDeque

                          Map (separate hierarchy — NOT a Collection)
             ┌────────────────┼────────────────┐
          HashMap         LinkedHashMap      TreeMap
       (no order)      (insertion order)   (sorted by key)
```

### List — ordered, allows duplicates
```java
List<String> list = new ArrayList<>();   // resizable array — fast random access
list.add("A");
list.add("B");
list.add("A");                            // duplicates allowed
list.get(0);                              // "A"
list.remove("A");                         // removes first occurrence

List<String> linked = new LinkedList<>(); // doubly-linked list — fast insert/delete
```
| | ArrayList | LinkedList |
|---|---|---|
| Access by index | O(1) fast | O(n) slow |
| Insert/delete (middle) | O(n) slow (shifting) | O(1) fast (once positioned) |
| Use when | Frequent reads | Frequent inserts/deletes |

### Set — no duplicates
```java
Set<String> set = new HashSet<>();        // no order guaranteed, O(1) operations
set.add("A");
set.add("A");                              // ignored — already exists

Set<String> linkedSet = new LinkedHashSet<>();  // preserves insertion order
Set<String> treeSet = new TreeSet<>();          // sorted (natural order or Comparator)
```

### Map — key-value pairs
```java
Map<String, Integer> map = new HashMap<>();
map.put("apple", 10);
map.put("banana", 20);
map.get("apple");                 // 10
map.getOrDefault("mango", 0);     // 0 (key doesn't exist)
map.containsKey("apple");         // true

for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

Map<String, Integer> sortedMap = new TreeMap<>(map);       // sorted by key
Map<String, Integer> orderedMap = new LinkedHashMap<>();    // insertion order preserved
```

### Queue / Deque
```java
Queue<Integer> queue = new LinkedList<>();
queue.offer(1);           // add to end
queue.poll();              // remove & return from front (FIFO)

Deque<Integer> stack = new ArrayDeque<>();   // used as a Stack (LIFO)
stack.push(1);
stack.pop();
```

### Comparable vs Comparator
```java
// Comparable — natural ordering, defined INSIDE the class
class Person implements Comparable<Person> {
    String name; int age;
    public int compareTo(Person other) { return this.age - other.age; } // sort by age
}
Collections.sort(people);   // uses compareTo()

// Comparator — external, flexible, multiple sort strategies
Comparator<Person> byName = (p1, p2) -> p1.name.compareTo(p2.name);
Collections.sort(people, byName);

// Java 8 style
people.sort(Comparator.comparing(p -> p.name));
people.sort(Comparator.comparing((Person p) -> p.age).reversed());
```

### `Iterator` & Fail-Fast Behavior
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String val = it.next();
    if (val.equals("B")) {
        it.remove();   // ✅ safe removal during iteration
    }
}

// ❌ Modifying a collection directly while iterating with for-each throws
// ConcurrentModificationException (fail-fast behavior)
```

---

## 11. Generics

Generics provide **compile-time type safety** — catch type errors before runtime, avoid manual casting.

### Without vs With Generics
```java
// Without generics (pre-Java 5) — unsafe, needs casting
List list = new ArrayList();
list.add("Hello");
list.add(123);              // no error — mixed types allowed!
String s = (String) list.get(1);  // ClassCastException at runtime!

// With generics — type-safe
List<String> list2 = new ArrayList<>();
list2.add("Hello");
// list2.add(123);           // ❌ compile-time error — caught immediately
String s2 = list2.get(0);    // no cast needed
```

### Generic Classes
```java
class Box<T> {                 // T = type parameter
    private T content;
    public void set(T content) { this.content = content; }
    public T get() { return content; }
}

Box<String> stringBox = new Box<>();
stringBox.set("Hello");

Box<Integer> intBox = new Box<>();
intBox.set(123);
```

### Generic Methods
```java
public <T> void printArray(T[] array) {
    for (T item : array) {
        System.out.println(item);
    }
}
```

### Bounded Type Parameters
```java
// T must be a Number or subclass (Integer, Double, etc.)
class Calculator<T extends Number> {
    T value;
    double doubleValue() { return value.doubleValue(); }
}
```

### Wildcards (`?`)
```java
// Upper bounded wildcard — accepts List<Integer>, List<Double>, etc.
public void printNumbers(List<? extends Number> list) {
    for (Number n : list) System.out.println(n);
}

// Lower bounded wildcard — accepts List<Integer>, List<Number>, List<Object>
public void addIntegers(List<? super Integer> list) {
    list.add(10);
}
```
### Wildcard Cheat Sheet
```
List<?>              → unknown type, read-only essentially
List<? extends T>     → T or subclass — can READ as T, cannot ADD (except null)
List<? super T>       → T or superclass — can ADD T, reading gives Object
PECS mnemonic: Producer Extends, Consumer Super
```

### Type Erasure (important internals concept)
```
At COMPILE TIME:              At RUNTIME (after erasure):
List<String> list             List list       (generic info removed!)
List<Integer> list2           List list2       (both become raw List)

This is why you CANNOT do:
- new T()                     (no info about T at runtime)
- array of generic type: new T[10]
- list instanceof List<String>  (only List possible)
```

---

## 12. Multithreading & Concurrency

### Process vs Thread
```
Process (e.g. a running JVM)
   ├── has its own memory space
   └── contains one or more Threads
         ├── Thread 1  ─┐
         ├── Thread 2   ├── share the SAME heap memory
         └── Thread 3  ─┘   but each has its OWN stack
```

### Creating Threads — 2 Ways
```java
// 1. Extending Thread class
class MyThread extends Thread {
    public void run() { System.out.println("Thread running"); }
}
new MyThread().start();     // start() creates a new call stack; NEVER call run() directly

// 2. Implementing Runnable (preferred — allows extending other classes too)
class MyTask implements Runnable {
    public void run() { System.out.println("Task running"); }
}
new Thread(new MyTask()).start();

// 3. Lambda (Java 8+, since Runnable is a functional interface)
new Thread(() -> System.out.println("Running via lambda")).start();
```

### Thread Lifecycle (Diagram)
```
        NEW
         │  start()
         ▼
     RUNNABLE ◀──────────────┐
       │    ▲                 │
       │    │ notify()/       │
       │    │ notifyAll()/    │ scheduler picks thread
       │    │ lock acquired   │
       ▼    │                 │
  RUNNING ──┴──────────────────┘
    │    │
    │    │ wait()/sleep()/blocked on I/O or lock
    │    ▼
    │  BLOCKED / WAITING / TIMED_WAITING
    │
    ▼ run() completes
 TERMINATED
```

### Synchronization — Preventing Race Conditions
```java
class Counter {
    private int count = 0;

    // synchronized method — only one thread can execute this at a time (per object)
    public synchronized void increment() {
        count++;   // without sync, two threads could read same value → lost update
    }

    // synchronized block — finer-grained locking
    public void incrementBlock() {
        synchronized (this) {
            count++;
        }
    }
}
```

### Race Condition Diagram
```
Without synchronization (count starts at 0):
  Thread A reads count = 0
  Thread B reads count = 0        ← both read BEFORE either writes
  Thread A writes count = 1
  Thread B writes count = 1        ← lost update! Should be 2.

With synchronized: Thread B must wait until Thread A's block finishes.
```

### `wait()`, `notify()`, `notifyAll()` (inter-thread communication)
```java
synchronized (lockObject) {
    while (!condition) {
        lockObject.wait();     // releases lock, pauses thread until notified
    }
    // proceed
}

// In another thread:
synchronized (lockObject) {
    condition = true;
    lockObject.notifyAll();    // wakes up waiting threads
}
```

### `ExecutorService` (preferred over manually managing threads)
```java
import java.util.concurrent.*;

ExecutorService executor = Executors.newFixedThreadPool(4);   // pool of 4 threads

executor.submit(() -> System.out.println("Task 1"));
executor.submit(() -> System.out.println("Task 2"));

executor.shutdown();   // no new tasks accepted, existing ones finish
```

### `Future` and `Callable` (tasks that return a result)
```java
Callable<Integer> task = () -> { return 42; };
Future<Integer> future = executor.submit(task);
Integer result = future.get();   // blocks until result is ready
```

### `synchronized` vs `Lock` (java.util.concurrent.locks)
```java
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();    // MUST unlock in finally to avoid deadlocks on exception
        }
    }
}
```
| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Flexibility | Basic | Advanced (tryLock, fairness, interruptible) |
| Auto-release | ✅ Yes (on block exit) | ❌ No (must unlock manually) |
| Use when | Simple cases | Need timeout, fairness, or multiple conditions |

### `volatile` Keyword
```java
private volatile boolean running = true;
// Ensures visibility — changes by one thread are immediately visible to others
// (without volatile, a thread might cache the value and never see updates)
```

### Deadlock
```
Thread 1: locks A, waits for B
Thread 2: locks B, waits for A
     → Neither can proceed → DEADLOCK

Prevention: always acquire locks in the SAME consistent order across all threads.
```

---

## 13. Java 8 Features (Complete)

Java 8 was a landmark release. Here's everything important:

### 1. Lambda Expressions
Concise syntax for implementing functional interfaces (interfaces with exactly one abstract method).
```java
// Before Java 8 (anonymous class)
Runnable r1 = new Runnable() {
    public void run() { System.out.println("Running"); }
};

// Java 8 lambda
Runnable r2 = () -> System.out.println("Running");

// With parameters
Comparator<Integer> comp = (a, b) -> a - b;

// Multi-statement body
Comparator<Integer> comp2 = (a, b) -> {
    System.out.println("Comparing");
    return a - b;
};
```

### 2. Functional Interfaces
```java
@FunctionalInterface   // optional annotation, enforces single abstract method
interface Calculator {
    int operate(int a, int b);
}

Calculator add = (a, b) -> a + b;
Calculator multiply = (a, b) -> a * b;
```

**Built-in Functional Interfaces (java.util.function):**
| Interface | Signature | Purpose |
|---|---|---|
| `Function<T,R>` | `R apply(T t)` | Transform T into R |
| `Predicate<T>` | `boolean test(T t)` | Test a condition |
| `Consumer<T>` | `void accept(T t)` | Consume a value, no return |
| `Supplier<T>` | `T get()` | Supply/produce a value, no input |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs, one output |
| `UnaryOperator<T>` | `T apply(T t)` | Function where input/output type match |

```java
Function<Integer, Integer> square = x -> x * x;
square.apply(5);   // 25

Predicate<Integer> isEven = x -> x % 2 == 0;
isEven.test(4);     // true

Consumer<String> printer = s -> System.out.println(s);
printer.accept("Hello");

Supplier<Double> randomValue = () -> Math.random();
randomValue.get();
```

### 3. Method References
Shorthand for lambdas that just call an existing method.
```java
// Lambda
list.forEach(s -> System.out.println(s));

// Method reference equivalent
list.forEach(System.out::println);

// Types of method references:
String::toUpperCase          // instance method of an arbitrary object (Type::method)
System.out::println          // instance method of a particular object (obj::method)
Integer::parseInt             // static method (Type::staticMethod)
ArrayList::new                 // constructor reference (Type::new)
```

### 4. Streams API
Process collections declaratively (functional-style pipelines) instead of manual loops.

```java
List<String> names = Arrays.asList("Charlie", "Alice", "Bob", "David");

List<String> result = names.stream()
    .filter(name -> name.length() > 3)     // keep names longer than 3 chars
    .map(String::toUpperCase)                // transform each
    .sorted()                                 // sort alphabetically
    .collect(Collectors.toList());            // gather into a List

// Without streams, this would be a multi-line loop with a temp list
```

### Stream Pipeline Diagram
```
 Source (Collection/Array)
        │
        ▼
 Intermediate Operations (lazy — don't execute until terminal op)
   filter()  →  map()  →  sorted()  →  distinct()  →  limit()
        │
        ▼
 Terminal Operation (triggers execution)
   collect() / forEach() / count() / reduce() / anyMatch()
        │
        ▼
   Result
```

### Common Stream Operations
```java
List<Integer> nums = Arrays.asList(5, 3, 8, 1, 9, 2);

// filter + collect
List<Integer> evens = nums.stream().filter(n -> n % 2 == 0).collect(Collectors.toList());

// map (transform)
List<Integer> squared = nums.stream().map(n -> n * n).collect(Collectors.toList());

// sorted
List<Integer> sorted = nums.stream().sorted().collect(Collectors.toList());

// reduce (combine into single value)
int sum = nums.stream().reduce(0, (a, b) -> a + b);       // 28
Optional<Integer> max = nums.stream().reduce(Integer::max);

// count, min, max
long count = nums.stream().filter(n -> n > 3).count();

// anyMatch, allMatch, noneMatch
boolean hasEven = nums.stream().anyMatch(n -> n % 2 == 0);

// distinct, limit, skip
List<Integer> firstThree = nums.stream().distinct().limit(3).collect(Collectors.toList());

// grouping (very common in interviews)
Map<Boolean, List<Integer>> grouped = nums.stream()
    .collect(Collectors.groupingBy(n -> n % 2 == 0));   // true → evens, false → odds

// joining strings
String joined = names.stream().collect(Collectors.joining(", "));

// IntStream for primitives (avoids boxing overhead)
int total = IntStream.rangeClosed(1, 10).sum();   // 1 to 10 inclusive
```

### Streams vs Collections
| | Collection | Stream |
|---|---|---|
| Stores data? | ✅ Yes | ❌ No — pipeline over a source |
| Reusable? | ✅ Yes | ❌ No — consumed once |
| Evaluation | Eager | Lazy (intermediate ops don't run until terminal op) |
| Mutability | Can be mutated | Doesn't mutate the source |

### Parallel Streams
```java
long count = nums.parallelStream().filter(n -> n > 3).count();
// Splits work across multiple threads automatically — use for LARGE datasets
// with CPU-intensive, independent operations. Overhead makes it slower for small data.
```

### 5. Optional — avoiding NullPointerException
```java
Optional<String> optName = Optional.ofNullable(getName());   // may or may not be null

optName.isPresent();                       // true/false check
optName.ifPresent(name -> System.out.println(name));   // run only if present
String name = optName.orElse("Default");    // fallback value
String name2 = optName.orElseGet(() -> computeDefault());   // lazy fallback
String name3 = optName.orElseThrow(() -> new RuntimeException("Missing"));

// Chaining
Optional<String> upper = optName.map(String::toUpperCase);
```

### 6. Default & Static Methods in Interfaces
```java
interface Vehicle {
    void drive();

    default void honk() {                 // has a body — implementing classes get it free
        System.out.println("Beep beep!");
    }

    static Vehicle create() {              // callable as Vehicle.create()
        return () -> System.out.println("Driving...");
    }
}
```
> This let interfaces evolve (add new methods) without breaking every existing implementation — critical for adding Stream methods to `Collection` without breaking old code.

### 7. New Date & Time API (`java.time`)
Replaces the old, mutable, thread-unsafe `Date`/`Calendar`.
```java
import java.time.*;

LocalDate date = LocalDate.now();                  // 2026-08-22
LocalDate specific = LocalDate.of(2026, 8, 22);
LocalTime time = LocalTime.now();
LocalDateTime dateTime = LocalDateTime.now();

LocalDate tomorrow = date.plusDays(1);              // immutable — returns NEW object
Period period = Period.between(date, specific);

DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd-MM-yyyy");
String formatted = date.format(formatter);
```
| Old API | New API (Java 8+) |
|---|---|
| `Date`, `Calendar` | `LocalDate`, `LocalTime`, `LocalDateTime` |
| Mutable | Immutable (thread-safe) |
| Month is 0-indexed (confusing bugs) | Month is 1-indexed |

### 8. `CompletableFuture` (async programming)
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return 42;   // runs on a separate thread
});

future.thenApply(result -> result * 2)
      .thenAccept(result -> System.out.println("Result: " + result));
```

### 9. Nashorn JavaScript Engine (deprecated later, historical mention)
Java 8 introduced a way to run JavaScript inside the JVM — removed in later versions, worth knowing it existed if seen in older code.

---

## 14. Memory Management & Garbage Collection

### Heap Structure (Generational GC)
```
┌─────────────────────────────────────────────────────┐
│                        HEAP                            │
│  ┌──────────────────────────┐  ┌────────────────────┐│
│  │      Young Generation      │  │  Old (Tenured) Gen  ││
│  │  ┌───────┐ ┌─────┐┌─────┐ │  │  Long-lived objects  ││
│  │  │ Eden   │ │ S0  ││ S1  │ │  │  (survived many      ││
│  │  │(new    │ │     ││     │ │  │   GC cycles)          ││
│  │  │objects)│ │     ││     │ │  │                        ││
│  │  └───────┘ └─────┘└─────┘ │  └────────────────────┘│
│  └──────────────────────────┘                          │
└─────────────────────────────────────────────────────┘
             Metaspace (class metadata — outside heap, Java 8+)
```

### GC Flow
```
1. New objects created in EDEN space
2. Eden fills up → MINOR GC runs
      - live objects moved to Survivor space (S0/S1, alternating)
      - dead objects reclaimed
3. Objects surviving several minor GCs → promoted to OLD GENERATION
4. Old Gen fills up → MAJOR/FULL GC runs (slower, stop-the-world)
```

### Garbage Collection Basics
- Objects with **no reachable references** become eligible for GC.
- You cannot force GC (`System.gc()` is only a *hint*, not a guarantee).
- Common collectors: **Serial, Parallel, CMS (deprecated), G1 (default since Java 9), ZGC/Shenandoah** (low-pause, newer JVMs).

### `finalize()` (legacy, avoid relying on it)
```java
protected void finalize() {
    // Called by GC before reclaiming — NOT guaranteed to run, deprecated since Java 9
    // Use try-with-resources / AutoCloseable instead for cleanup
}
```

### Memory Leaks in Java (yes, still possible!)
Common causes: static collections that keep growing, unclosed resources, listeners not removed, inner classes holding implicit outer references.

---

## 15. Nested & Inner Classes

```java
class Outer {
    int outerField = 10;

    // 1. Inner class (non-static) — holds implicit reference to Outer instance
    class Inner {
        void show() { System.out.println("Outer field: " + outerField); }
    }

    // 2. Static nested class — no reference to Outer instance needed
    static class StaticNested {
        void show() { System.out.println("Static nested class"); }
    }

    void method() {
        // 3. Local class — defined inside a method
        class Local {
            void show() { System.out.println("Local class"); }
        }
        new Local().show();
    }
}

// Usage
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();          // needs an Outer instance
Outer.StaticNested nested = new Outer.StaticNested();   // no Outer instance needed

// 4. Anonymous class — no name, one-time use
Runnable r = new Runnable() {
    public void run() { System.out.println("Anonymous class"); }
};
```

| Type | Needs Outer instance? | Common use |
|---|---|---|
| Inner (non-static) | ✅ Yes | Tightly coupled helper logic |
| Static nested | ❌ No | Logical grouping, builder patterns |
| Local | ❌ No (method-scoped) | Rarely used |
| Anonymous | ❌ No | One-off implementations (listeners, callbacks — largely replaced by lambdas now) |

---

## 16. Annotations & Reflection (Overview)

### Built-in Annotations
```java
@Override           // marks method as overriding a parent method (compiler checks)
@Deprecated          // marks as outdated, discouraged from use
@SuppressWarnings("unchecked")   // tells compiler to ignore specific warnings
@FunctionalInterface // enforces single abstract method
```

### Custom Annotations
```java
@Retention(RetentionPolicy.RUNTIME)   // available at runtime via reflection
@Target(ElementType.METHOD)            // can only be applied to methods
@interface Test {
    String description() default "";
}

class MyTests {
    @Test(description = "Checks addition")
    void testAdd() { /* ... */ }
}
```

### Reflection (inspecting classes at runtime)
```java
Class<?> clazz = MyClass.class;
System.out.println(clazz.getName());

for (Method method : clazz.getDeclaredMethods()) {
    System.out.println(method.getName());
}

// Used heavily by frameworks like Spring, Hibernate, JUnit —
// to scan annotations, inject dependencies, create proxies, etc.
```

---

## 17. I/O & Serialization

### Byte Streams vs Character Streams
```java
// Byte streams — for binary data (images, etc.) — InputStream/OutputStream
FileInputStream fis = new FileInputStream("file.dat");

// Character streams — for text — Reader/Writer
FileReader fr = new FileReader("file.txt");
BufferedReader br = new BufferedReader(fr);   // adds buffering for efficiency
String line = br.readLine();
```

### Serialization — converting an object to a byte stream (e.g. to save/send it)
```java
class User implements Serializable {   // marker interface, no methods to implement
    String name;
    transient String password;   // 'transient' fields are SKIPPED during serialization
}

// Writing
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.ser"));
oos.writeObject(new User());
oos.close();

// Reading
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.ser"));
User user = (User) ois.readObject();
```

---

## 18. Common Interview Questions

1. **Why is Java "platform-independent"?**
   → Source compiles to bytecode (`.class`), which any JVM (per-OS) can execute — not native machine code.

2. **Difference between `==` and `.equals()`?**
   → `==` compares references (memory addresses) for objects; `.equals()` compares logical content (if overridden).

3. **Why are Strings immutable?**
   → Security (used in class loading, network connections), thread-safety (shared safely), and enables the String Pool for memory efficiency.

4. **Difference between `ArrayList` and `LinkedList`?**
   → See Section 10 table — array-backed vs doubly-linked list; different Big-O trade-offs.

5. **What is the difference between `HashMap` and `Hashtable`?**
   → `HashMap` is not synchronized (faster, not thread-safe), allows one `null` key; `Hashtable` is synchronized (legacy, thread-safe), no `null` keys/values allowed. Prefer `ConcurrentHashMap` for thread-safe use today.

6. **Explain type erasure in generics.**
   → Generic type information exists only at compile-time for type checking; removed at runtime (see Section 11) — this is why you can't do `new T()` or check `instanceof List<String>`.

7. **Difference between `Comparable` and `Comparator`?**
   → `Comparable` defines natural/default ordering inside the class (`compareTo`); `Comparator` defines external, swappable ordering strategies (`compare`).

8. **What's the difference between checked and unchecked exceptions?**
   → See Section 9 table. Checked must be declared/handled (compile-time enforced); unchecked are typically programmer bugs.

9. **How does `HashMap` work internally?**
   → Uses an array of "buckets"; each key's `hashCode()` determines the bucket index; collisions within a bucket are handled via a linked list (or a balanced tree if the bucket gets large, since Java 8) — lookup is O(1) average, O(log n) worst case (post Java 8 treeification).

10. **What is the difference between `synchronized` method and `synchronized` block?**
    → Method-level locks the whole method (on `this` or the Class for static); block-level allows locking on a specific object, giving finer control and better concurrency.

11. **Explain the Java Memory Model briefly (Heap vs Stack).**
    → See Section 2's table — Heap stores objects (shared across threads), Stack stores method frames/local variables (one per thread).

12. **What are functional interfaces, and why do lambdas need them?**
    → Interfaces with exactly one abstract method; a lambda is essentially a compact instance of a functional interface — the compiler needs a target type to know what shape the lambda implements.

13. **Difference between `Stream.map()` and `Stream.flatMap()`?**
    → `map()` transforms each element 1-to-1; `flatMap()` transforms each element into a stream and flattens all resulting streams into one (useful for nested collections, e.g., `List<List<Integer>>` → `List<Integer>`).

---

## 19. Cheat Sheet Summary

| Concept | Key Idea |
|---|---|
| JVM | Executes bytecode, provides platform independence |
| Heap vs Stack | Heap: objects (shared); Stack: method frames (per-thread) |
| OOP 4 pillars | Encapsulation, Abstraction, Inheritance, Polymorphism |
| Overloading vs Overriding | Compile-time (same class) vs Runtime (parent-child) |
| Interface vs Abstract class | Contract/capability vs shared partial implementation |
| Checked vs Unchecked exception | Must handle (compile-time) vs optional (runtime) |
| ArrayList vs LinkedList | Fast access vs fast insert/delete |
| HashMap vs TreeMap vs LinkedHashMap | No order vs sorted vs insertion order |
| Generics | Compile-time type safety, erased at runtime |
| Thread creation | `Thread`, `Runnable`, or lambda (Java 8+) |
| synchronized / Lock | Prevent race conditions on shared mutable state |
| Java 8: Lambdas | Concise functional interface implementations |
| Java 8: Streams | Declarative, pipeline-based collection processing |
| Java 8: Optional | Explicit null-handling, avoids NPE |
| Java 8: java.time | Immutable, thread-safe date/time API |

### Golden Rules
```
1. Prefer composition over inheritance where possible.
2. Program to an interface, not an implementation.
3. Always close resources (use try-with-resources).
4. Never rely on finalize() for cleanup.
5. Use ConcurrentHashMap / synchronized / Locks for shared mutable state across threads.
6. Prefer immutable objects where possible — simpler, thread-safe by nature.
7. Use Streams for readability on collection transformations, but don't force-fit
   every loop into a stream if it hurts clarity.
```

---

### Suggested Learning Path
```
JVM/JDK/JRE → Data types → OOP basics → Inheritance/Polymorphism
   → Interfaces/Abstract classes → Exceptions → Collections
   → Generics → Multithreading basics → Java 8 (Lambdas → Streams → Optional → java.time)
   → Memory management/GC → Nested classes → Reflection/Annotations → I/O
```

*End of Notes — pair this with small practice programs (implement a LinkedList from scratch, build a thread-safe counter, refactor a loop into a Stream pipeline) to cement each concept.*
