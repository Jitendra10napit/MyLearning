For your **13+ years of C#/.NET experience and Staff/Lead Engineer interviews**, I would prepare C# at **three levels**: language fundamentals, runtime/concurrency, and architecture-level usage. At your level, interviewers are likely to ask **“why/when/trade-off”**, not just syntax.

# C# Interview FAQ — Staff/Lead Level

## 🔥 1. `class` vs `struct`

### Class

Reference type:

```csharp
var e1 = new Employee();
var e2 = e1;

e2.Name = "Jitendra";
```

Both variables refer to the same object.

### Struct

Value type:

```csharp
var p1 = new Point { X = 10 };
var p2 = p1;

p2.X = 20;
```

`p1.X` remains `10`.

### Staff-level answer

> "Classes are reference types and are generally suitable for entities with identity and mutable state. Structs are value types and can be useful for small, logically value-like data, but I avoid large mutable structs because copying them can be expensive and confusing."

---

# 🔥 2. Value type vs Reference type

```text
Value Type
    ↓
Contains the value
    ↓
int, double, bool, struct

Reference Type
    ↓
Contains reference to object
    ↓
class, array, string, delegate
```

Important interview point:

> "Value type doesn't simply mean stack and reference type doesn't simply mean heap."

The actual storage behavior depends on context, lifetime, boxing, captured variables, etc.

---

# 🔥 3. `ref`, `out`, and `in`

### `ref`

Must be initialized before passing:

```csharp
void Update(ref int value)
{
    value++;
}
```

### `out`

Doesn't need initialization:

```csharp
void GetValue(out int value)
{
    value = 10;
}
```

### `in`

Passes by readonly reference:

```csharp
void Print(in LargeStruct value)
{
}
```

Useful when you want to avoid copying a potentially large value type while preventing modification through that parameter.

---

# 🔥 4. `const` vs `readonly` vs `static readonly`

### const

Compile-time constant:

```csharp
public const int MaxRetries = 3;
```

### readonly

Can be assigned during declaration or constructor:

```csharp
public readonly int MaxRetries;

public Service()
{
    MaxRetries = 3;
}
```

### static readonly

One value associated with the type:

```csharp
public static readonly string Version = "1.0";
```

### Interview trap

`const` values are embedded into consuming assemblies at compile time. Therefore, changing a public constant can require consumers to be recompiled.

---

# 🔥 5. Abstract class vs Interface

### Abstract class

Can contain:

* fields
* constructors
* implemented methods
* abstract methods
* state

### Interface

Primarily defines a contract.

```csharp
public interface IPaymentService
{
    Task PayAsync();
}
```

### Staff-level answer

> "I use an interface when I need a contract/capability and multiple unrelated implementations may implement it. I use an abstract class when there is meaningful shared behavior or state that belongs to a common abstraction."

---

# 🔥 6. What is polymorphism?

Two major forms:

### Compile-time

Method overloading:

```csharp
void Calculate(int x)
void Calculate(double x)
```

### Runtime

Method overriding:

```csharp
Animal animal = new Dog();

animal.Speak();
```

The overridden `Dog.Speak()` executes.

---

# 🔥 7. What is method overloading vs overriding?

### Overloading

Same method name, different parameters.

```csharp
void Process(int id)
void Process(string name)
```

### Overriding

Derived class changes base implementation.

```csharp
public override void Process()
{
}
```

Requires virtual/abstract member in the base class.

---

# 🔥 8. What is virtual method dispatch?

Example:

```csharp
Animal animal = new Dog();

animal.Speak();
```

If `Speak()` is virtual and `Dog` overrides it, runtime determines the implementation based on the actual object.

```text
Reference Type → Animal
Actual Object  → Dog
                    ↓
              Dog.Speak()
```

This is runtime polymorphism.

---

# 🔥 9. What is encapsulation?

Keeping internal implementation details hidden and exposing controlled behavior.

Bad:

```csharp
public decimal Balance;
```

Better:

```csharp
public class Account
{
    private decimal _balance;

    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException();

        _balance += amount;
    }

    public decimal GetBalance() => _balance;
}
```

The class controls how its state changes.

---

# 🔥 10. What is SOLID?

You should know all five with practical examples.

```text
S → Single Responsibility
O → Open/Closed
L → Liskov Substitution
I → Interface Segregation
D → Dependency Inversion
```

At your experience level, don't just define them.

Explain:

> "I use SOLID to control coupling, make components independently testable, and make the system easier to extend without modifying stable code."

---

# 🔥 11. Explain Liskov Substitution Principle

If:

```csharp
Bird bird = new Sparrow();
bird.Fly();
```

works for Sparrow but:

```csharp
Bird bird = new Penguin();
bird.Fly();
```

throws `NotSupportedException`, the abstraction is probably wrong.

The issue isn't that Penguin is a bad implementation; the base abstraction incorrectly assumes all birds can fly.

Better:

```text
Bird
 ├── Sparrow
 └── Penguin

FlyingBird
 └── Sparrow
```

---

# 🔥 12. Composition vs Inheritance

Inheritance:

```csharp
class EmailService : NotificationService
{
}
```

Composition:

```csharp
class NotificationService
{
    private readonly IEmailSender _emailSender;
}
```

For large systems, composition often provides lower coupling and greater flexibility.

Staff-level answer:

> "I generally prefer composition over inheritance when behavior needs to vary independently because it avoids deep inheritance hierarchies and makes dependencies explicit."

---

# 🔥 13. What are delegates?

A delegate is a type-safe reference to a method.

```csharp
delegate int Calculator(int x, int y);
```

Then:

```csharp
Calculator add = (x, y) => x + y;
```

Common built-in delegates:

```csharp
Action
Func<T>
Predicate<T>
```

---

# 🔥 14. Action vs Func vs Predicate

### Action

Returns nothing:

```csharp
Action<string> log = message =>
{
    Console.WriteLine(message);
};
```

### Func

Returns a value:

```csharp
Func<int, int> square = x => x * x;
```

### Predicate

Returns bool:

```csharp
Predicate<int> isEven = x => x % 2 == 0;
```

---

# 🔥 15. What are events?

Events provide a controlled publish/subscribe mechanism around delegates.

```csharp
public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
```

Subscriber:

```csharp
service.OrderCreated += HandleOrderCreated;
```

Publisher:

```csharp
OrderCreated?.Invoke(this, args);
```

Important distinction:

> A delegate can generally be invoked by its owner; an event restricts external consumers primarily to subscribing/unsubscribing.

---

# 🔥 16. What is a lambda expression?

```csharp
x => x * 2
```

Example:

```csharp
var result = numbers
    .Where(x => x > 10)
    .Select(x => x * 2);
```

Lambdas are heavily used with:

* LINQ
* delegates
* callbacks
* async code
* dependency injection configuration

---

# 🔥 17. What is closure?

Example:

```csharp
int multiplier = 10;

Func<int, int> calculate =
    x => x * multiplier;
```

The lambda captures `multiplier`.

The compiler creates a closure object to preserve the captured state.

This matters because captured variables can extend object lifetimes and introduce subtle bugs in loops or asynchronous code.

---

# 🔥 18. What is boxing and unboxing?

Boxing:

```csharp
int x = 10;

object obj = x;
```

The value is represented as an object.

Unboxing:

```csharp
int y = (int)obj;
```

Frequent boxing can cause allocations and performance overhead.

Avoid unnecessary boxing in performance-critical paths.

---

# 🔥 19. What is `string` immutability?

```csharp
string name = "Jitendra";

name += " Napit";
```

The original string isn't modified. A new string is created.

For repeated string manipulation:

```csharp
var builder = new StringBuilder();
```

can be more appropriate.

---

# 🔥 20. String vs StringBuilder

If you repeatedly concatenate:

```csharp
string result = "";

for (int i = 0; i < 10000; i++)
{
    result += i;
}
```

you can generate many intermediate string objects.

Instead:

```csharp
var builder = new StringBuilder();

for (int i = 0; i < 10000; i++)
{
    builder.Append(i);
}

string result = builder.ToString();
```

But don't say `StringBuilder` is always faster. For small/simple concatenations, normal string interpolation/concatenation can be perfectly appropriate.

---

# 🔥 21. `==` vs `Equals()` vs `ReferenceEquals()`

For reference types:

```csharp
== 
```

may represent reference equality or overloaded value equality depending on the type.

`Equals()` is intended to represent logical equality and can be overridden.

```csharp
ReferenceEquals(a, b)
```

checks whether both references point to the exact same object.

This becomes especially important when implementing value objects.

---

# 🔥 22. What are records?

Records are designed for data-centric/value-oriented models and provide convenient value-based equality semantics.

```csharp
public record EmployeeDto(
    int Id,
    string Name);
```

Two records with the same values can compare equal.

Useful for:

* DTOs
* immutable data
* value objects
* messages

---

# 🔥 23. Class vs Record

```csharp
public class Employee
{
    public int Id { get; set; }
}
```

versus:

```csharp
public record Employee(
    int Id,
    string Name);
```

A record is often preferable when **value-based equality and immutable/data-oriented semantics** are desired.

Don't automatically use records for EF Core entities; entity identity and change tracking often make classes more natural.

---

# 🔥 24. What is pattern matching?

Modern C# supports powerful pattern matching:

```csharp
if (employee is { Salary: > 100000 })
{
}
```

Or:

```csharp
return employee switch
{
    Manager m => $"Manager: {m.Name}",
    Developer d => $"Developer: {d.Name}",
    _ => "Unknown"
};
```

Useful for clean conditional logic and type/state inspection.

---

# 🔥 25. What are nullable reference types?

Example:

```csharp
string name;
string? optionalName;
```

`string?` explicitly indicates that `null` is expected.

Nullable reference types help detect potential null problems at compile time.

```csharp
string? name = GetName();

Console.WriteLine(name.Length);
```

The compiler warns because `name` could be null.

---

# 🔥 26. `??` vs `??=`

Null coalescing:

```csharp
var name = input ?? "Unknown";
```

Null coalescing assignment:

```csharp
name ??= "Unknown";
```

Meaning:

> Assign only if the current value is null.

---

# 🔥 27. What is exception handling best practice?

Avoid:

```csharp
try
{
}
catch(Exception ex)
{
    throw ex;
}
```

This can reset the stack trace.

Prefer:

```csharp
catch (Exception)
{
    throw;
}
```

Or, preferably, add meaningful context:

```csharp
catch (Exception ex)
{
    throw new OrderProcessingException(
        "Failed to process order.", ex);
}
```

At architecture level:

> Don't use exceptions for normal business flow. Handle exceptions at appropriate boundaries and preserve the original exception as the inner exception when wrapping.

---

# 🔥 28. `throw` vs `throw ex`

Bad:

```csharp
catch(Exception ex)
{
    throw ex;
}
```

Good:

```csharp
catch(Exception)
{
    throw;
}
```

`throw;` preserves the original stack trace.

---

# 🔥 29. What is `IDisposable`?

Used when an object owns resources that need deterministic cleanup.

```csharp
public class FileProcessor : IDisposable
{
    public void Dispose()
    {
        // cleanup
    }
}
```

Usage:

```csharp
using var stream = File.OpenRead(path);
```

Common examples:

* streams
* database connections
* unmanaged resource wrappers
* some HTTP/networking resources

---

# 🔥 30. What is `using`?

```csharp
using var connection = new SqlConnection(connectionString);
```

The compiler generates disposal semantics roughly equivalent to a `try/finally`.

For async disposable resources:

```csharp
await using var resource = ...;
```

uses `IAsyncDisposable`.

---

# 🔥 31. `IDisposable` vs `IAsyncDisposable`

`IDisposable`:

```csharp
void Dispose();
```

`IAsyncDisposable`:

```csharp
ValueTask DisposeAsync();
```

Use `IAsyncDisposable` when cleanup itself can involve asynchronous I/O.

---

# 🔥 32. What is `async/await`?

Example:

```csharp
public async Task<Employee> GetEmployeeAsync(int id)
{
    return await repository.GetEmployeeAsync(id);
}
```

`await` doesn't mean:

> "Create a new thread."

For I/O-bound operations, the thread can return to the ThreadPool while the operation is pending.

```text
Request
  ↓
await DB call
  ↓
Thread released
  ↓
DB completes
  ↓
Continuation resumes
```

This is one of the **most important C# topics for your interview**.

---

# 🔥 33. Task vs Thread

### Thread

An OS/runtime execution resource.

### Task

Represents an asynchronous operation/computation.

```csharp
Task.Run(...)
```

does not mean "Task = Thread."

For I/O:

```csharp
await httpClient.GetAsync(url);
```

you normally don't need `Task.Run`.

---

# 🔥 34. When should you use Task.Run?

For CPU-bound work in an appropriate application scenario:

```csharp
await Task.Run(() => HeavyCalculation());
```

But don't use:

```csharp
await Task.Run(() => await db.GetDataAsync());
```

as a general technique to make I/O asynchronous.

The database API is already asynchronous.

---

# 🔥 35. `Task.WhenAll` vs sequential await

Sequential:

```csharp
var users = await GetUsersAsync();
var orders = await GetOrdersAsync();
```

Potentially:

```text
5 sec + 5 sec = 10 sec
```

If independent:

```csharp
var usersTask = GetUsersAsync();
var ordersTask = GetOrdersAsync();

await Task.WhenAll(usersTask, ordersTask);
```

Potentially:

```text
max(5, 5) ≈ 5 sec
```

This is **concurrency**, not necessarily parallel CPU execution.

---

# 🔥 36. What is `CancellationToken`?

Allows cooperative cancellation.

```csharp
public async Task ProcessAsync(
    CancellationToken cancellationToken)
{
    await db.SaveChangesAsync(cancellationToken);
}
```

ASP.NET Core can propagate request cancellation:

```text
Client
 ↓
Request cancelled
 ↓
CancellationToken
 ↓
Service
 ↓
Database/API call cancelled
```

This is very important for scalable APIs.

---

# 🔥 37. What is `lock`?

Used to protect shared state from concurrent access.

```csharp
lock (_syncObject)
{
    _counter++;
}
```

Only one thread can enter the critical section at a time for that lock object.

Avoid:

```csharp
lock(this)
```

or locking on publicly accessible objects.

---

# 🔥 38. `lock` vs `SemaphoreSlim`

`lock` is synchronous mutual exclusion.

```csharp
lock (_lock)
{
}
```

`SemaphoreSlim` can support asynchronous waiting:

```csharp
await _semaphore.WaitAsync();

try
{
    // protected work
}
finally
{
    _semaphore.Release();
}
```

For async code, don't block threads using `lock` when the protected operation itself needs asynchronous waiting.

---

# 🔥 39. What is a race condition?

Suppose:

```csharp
_counter++;
```

looks like one operation but conceptually involves read/modify/write.

Two threads can interleave:

```text
Thread A → Read 10
Thread B → Read 10
Thread A → Write 11
Thread B → Write 11
```

Expected:

```text
12
```

Actual:

```text
11
```

This is a race condition.

Solutions include:

* `lock`
* `Interlocked`
* concurrent collections
* immutable state
* appropriate synchronization

---

# 🔥 40. What is `Interlocked`?

For simple atomic operations:

```csharp
Interlocked.Increment(ref _counter);
```

This can avoid a full lock for certain operations.

Good answer:

> "I use Interlocked for simple atomic state transitions and locks or other synchronization primitives when I need to protect a larger critical section."

---

# 🔥 41. What are concurrent collections?

Examples:

```csharp
ConcurrentDictionary<TKey,TValue>
ConcurrentQueue<T>
ConcurrentBag<T>
ConcurrentStack<T>
```

Example:

```csharp
var cache = new ConcurrentDictionary<string, Employee>();

cache.TryAdd("E1", employee);
```

They are designed for concurrent access and can be preferable to manually locking ordinary collections in suitable scenarios.

---

# 🔥 42. What is `volatile`?

`volatile` tells the compiler/runtime that reads and writes to a field have specific visibility/ordering semantics across threads.

```csharp
private volatile bool _stop;
```

But **volatile does not make compound operations atomic**.

This is not made safe merely by `volatile`:

```csharp
_counter++;
```

For atomic counters use:

```csharp
Interlocked.Increment(ref _counter);
```

---

# 🔥 43. What is dependency injection?

Instead of:

```csharp
public class OrderService
{
    private readonly EmailService _email = new();
}
```

use:

```csharp
public class OrderService
{
    private readonly IEmailService _email;

    public OrderService(IEmailService email)
    {
        _email = email;
    }
}
```

Benefits:

* loose coupling
* testability
* replaceable implementations
* centralized configuration

---

# 🔥 44. Singleton vs Scoped vs Transient

In ASP.NET Core:

```text
Transient → new instance each resolution
Scoped    → one per scope/request
Singleton → one for application lifetime
```

Typical:

```text
DbContext       → Scoped
Stateless service → often Scoped/Transient depending on design
Configuration  → Singleton
```

### Critical Staff-level issue

Don't inject a **Scoped** service directly into a **Singleton**.

That creates a lifetime mismatch and can cause incorrect state/resource behavior.

---

# 🔥 45. What is the difference between `IServiceProvider` and dependency injection?

`IServiceProvider` is the service-resolution abstraction.

But don't turn application code into:

```csharp
var service = provider.GetService<IMyService>();
```

everywhere.

Prefer constructor injection:

```csharp
public OrderService(IOrderRepository repository)
{
}
```

Use explicit service location only where the architecture genuinely requires it.

---

# 🔥 46. What is garbage collection?

.NET GC automatically manages managed memory.

Conceptually:

```text
Managed Objects
      ↓
Generation 0
      ↓
Generation 1
      ↓
Generation 2
```

Short-lived objects generally die young.

Long-lived objects can survive into older generations.

---

# 🔥 47. What are Gen 0, Gen 1 and Gen 2?

### Gen 0

Short-lived objects.

### Gen 1

Intermediate objects.

### Gen 2

Long-lived objects.

Large Object Heap is used for sufficiently large allocations and has its own GC behavior.

At your level, also understand that excessive allocations can increase GC pressure and latency.

---

# 🔥 48. What is LOH?

**Large Object Heap** stores sufficiently large objects.

Examples can include:

```csharp
byte[]
large arrays
large strings
```

Large allocations can create memory pressure and affect GC behavior.

For high-throughput video/file processing, this becomes particularly important, although for your current Staff interviews I would emphasize the general memory-management principles.

---

# 🔥 49. What is `Span<T>`?

`Span<T>` provides a type-safe view over contiguous memory without necessarily allocating a new object.

```csharp
Span<int> numbers = stackalloc int[5];

numbers[0] = 10;
```

Useful in performance-sensitive code involving:

* parsing
* buffers
* serialization
* memory manipulation

Important constraint:

> `Span<T>` is a `ref struct` and cannot generally be stored on the managed heap or used across an `await`.

---

# 🔥 50. `Span<T>` vs `Memory<T>`

A very common advanced question.

```text
Span<T>
 ↓
Fast synchronous memory access
 ↓
ref struct
 ↓
Cannot cross await

Memory<T>
 ↓
Can live on managed heap
 ↓
Can be used across async boundaries
```

So if asynchronous APIs need to retain the memory:

```csharp
Memory<byte>
```

may be appropriate.

---

# 🔥 51. What are generics?

Generics provide type-safe reusable code.

```csharp
public class Repository<T>
{
    public T Get(int id)
    {
        ...
    }
}
```

Instead of:

```csharp
object
```

you get compile-time type safety.

Generics also avoid boxing in many value-type scenarios.

---

# 🔥 52. What are generic constraints?

Example:

```csharp
public class Repository<T>
    where T : class
{
}
```

Other constraints:

```csharp
where T : struct
where T : new()
where T : BaseEntity
where T : IDisposable
```

They constrain what types can be used.

---

# 🔥 53. What is covariance and contravariance?

Advanced but important for your experience.

Covariance:

```csharp
IEnumerable<string>
```

can be assigned to:

```csharp
IEnumerable<object>
```

because `IEnumerable<out T>` is covariant.

Contravariance:

```csharp
Action<object>
```

can be assigned to:

```csharp
Action<string>
```

because `Action<in T>` is contravariant.

Remember:

```text
out → covariance
in  → contravariance
```

---

# 🔥 54. What are extension methods?

Example:

```csharp
public static class StringExtensions
{
    public static bool IsValidEmail(this string value)
    {
        ...
    }
}
```

Usage:

```csharp
email.IsValidEmail();
```

The method is statically dispatched; extension methods don't actually modify the original type.

---

# 🔥 55. What is reflection?

Reflection allows inspecting types and metadata at runtime.

```csharp
var type = typeof(Employee);

var properties = type.GetProperties();
```

Used by frameworks such as:

* dependency injection
* serializers
* ORMs
* test frameworks

Trade-off:

> Reflection is flexible but can introduce runtime overhead and reduced compile-time safety.

---

# 🔥 56. What are attributes?

Metadata attached to code elements.

```csharp
[Obsolete("Use NewMethod")]
public void OldMethod()
{
}
```

Custom attribute:

```csharp
[Authorize]
public IActionResult Get()
{
}
```

Frameworks can inspect attributes through reflection.

---

# 🔥 57. What is `dynamic`?

```csharp
dynamic value = GetSomething();

value.SomeMethod();
```

The compiler defers certain member resolution to runtime.

Difference:

```text
var
 ↓
Compile-time type inference

dynamic
 ↓
Runtime binding
```

`dynamic` sacrifices compile-time safety and can introduce runtime errors.

---

# 🔥 58. `var` vs `dynamic`

```csharp
var employee = new Employee();
```

The compiler knows the type at compile time.

```csharp
dynamic employee = GetEmployee();
```

Member binding happens at runtime.

Important:

> `var` is strongly typed; it does not mean dynamically typed.

---

# 🔥 59. What is an immutable object?

Once created, its state cannot be changed.

Example:

```csharp
public record Employee(
    int Id,
    string Name);
```

Instead of modifying the existing object, create another value.

Benefits:

* thread safety
* easier reasoning
* fewer side effects
* easier concurrency

---

# 🔥 60. What is the difference between shallow copy and deep copy?

Shallow copy copies the object structure but referenced objects may still be shared.

```text
Object A
 ├── Name
 └── Address ─────┐
                  ↓
                Object B
```

A deep copy creates independent nested objects.

This matters when dealing with mutable object graphs.

---

# ⭐ C# Coding Questions You Should Prepare

For your interview, I'd also expect coding questions around:

### Easy/Medium

1. Reverse a string
2. Find duplicate characters
3. First non-repeating character
4. Palindrome
5. Anagram
6. Fibonacci
7. Factorial
8. Two Sum
9. Remove duplicates
10. Find second-largest number

### LINQ

11. Group employees by department
12. Find highest salary per department
13. Find second-highest salary
14. Find duplicate records
15. Join two collections
16. Flatten nested collections using `SelectMany`
17. Find employees whose salary is greater than department average
18. Find top 3 employees per department

### Advanced C#

19. Implement a thread-safe Singleton
20. Implement an LRU cache
21. Implement producer-consumer
22. Implement retry with exponential backoff
23. Implement a custom middleware
24. Implement a thread-safe counter
25. Implement an async batch processor

---

# 🎯 What I would prioritize for your Staff interview

Don't spend equal time on all 60.

### Tier 1 — Must know

```text
async/await
Task
Task.WhenAll
CancellationToken
Thread vs Task
lock
SemaphoreSlim
race condition
deadlock
Interlocked
IEnumerable vs IQueryable
delegates/events
interface vs abstract class
SOLID
DI lifetimes
GC
IDisposable
exception handling
generics
records
```

### Tier 2 — Strong Staff-level topics

```text
Span<T>
Memory<T>
boxing/unboxing
closure
covariance/contravariance
immutable objects
reflection
expression trees
concurrent collections
value vs reference semantics
```

### Tier 3 — Architecture discussion

Be prepared to answer:

> **"You have a high-volume .NET API receiving 10,000 requests/sec. How would you design the C# application to handle concurrency, memory, database access and failures?"**

A strong answer should connect:

```text
                    API
                     │
              Async/Await
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
   Cancellation   Concurrency   Validation
        │            │
        │       Semaphore/Queue
        │            │
        └────────────┼────────────┘
                     ↓
                  Service
                     ↓
              EF Core / DB
                     ↓
              Connection Pool
                     ↓
                  Cache
```

And then discuss **backpressure, connection-pool limits, ThreadPool starvation, cancellation, retries, idempotency, distributed locks, caching and observability**.

That is the level at which I'd expect your **13+ years of C# experience** to be evaluated in a Staff/Lead interview.
