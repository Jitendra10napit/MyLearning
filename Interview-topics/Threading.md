Absolutely. For your Senior .NET Engineer / Technical Lead interviews, the best approach is to explain each concept using **a simple definition + real-world example + C# code + when to use it**.

I'll use practical examples from Employee APIs, Order Processing, Payment Services and Azure Service Bus so you can explain them confidently in interviews.

# **1. Thread vs Process vs Multithreading**

### **Q1. What is a process?**

A process is an executing application with its own memory space and resources.

**Example:** When you run an ASP.NET Core application, the operating system creates a process for it.

### **Q2. What is a thread?**

A thread is an execution unit inside a process.

**Example:** An ASP.NET Core application can handle multiple incoming requests using multiple ThreadPool threads.

### **Q3. What is multithreading?**

Multithreading means executing multiple threads within one process.

```
using System;
using System.Threading;

class Program
{
    static void Main()
    {
        Thread t1 = new Thread(() => Print("Employee"));
        Thread t2 = new Thread(() => Print("Order"));

        t1.Start();
        t2.Start();

        t1.Join();
        t2.Join();

        Console.WriteLine("Completed");
    }

    static void Print(string name)
    {
        for (int i = 1; i <= 3; i++)
        {
            Console.WriteLine($"{name}: {i}");
        }
    }
}
```

**How to explain in interview:**

"I can create multiple threads to execute independent operations concurrently. For example, one thread can process employee information while another processes order information. However, if both threads access shared data, I need synchronization to avoid race conditions."

# **2. Concurrency vs Parallelism**

### **Q4. What is the difference?**

**Concurrency**

Multiple tasks make progress during overlapping periods.

Task A

Task B

**Parallelism**

Multiple tasks execute simultaneously on different CPU cores.

**CPU Core 1**

Task A

**CPU Core 2**

Task B

**Example:**

- Concurrency: Calling Employee API and Department API asynchronously.
- Parallelism: Processing 10,000 images using multiple CPU cores.

```
// Concurrency
var employeeTask = GetEmployeesAsync();
var departmentTask = GetDepartmentsAsync();

await Task.WhenAll(employeeTask, departmentTask);
```

```
// Parallel CPU processing
Parallel.ForEach(images, image =>
{
    ResizeImage(image);
});
```

**Interview answer:** Concurrency is about managing multiple operations, whereas parallelism is about executing multiple operations at the same time.

# **3. Thread vs Task vs ThreadPool**

### **Q5. What is the difference between Thread, Task and ThreadPool?**

**Real-world example:** You need to process employee records.

Dedicated Thread:

```
var thread = new Thread(() =>
{
    ProcessEmployees();
});

thread.Start();
```

ThreadPool:

```
ThreadPool.QueueUserWorkItem(_ =>
{
    ProcessEmployees();
});
```

Task:

```
Task task = Task.Run(() =>
{
    ProcessEmployees();
});

await task;
```

**Explanation:**

- Thread: Creates and manages a dedicated thread.
- ThreadPool: Reuses worker threads.
- Task: Represents an asynchronous operation or unit of work. CPU-bound Task.Run work typically uses ThreadPool threads.

**When to use?**

In modern ASP.NET Core applications, I generally prefer Task-based programming rather than manually creating threads.

# **4. Async and Await**

### **Q6. Does async/await create a new thread?**

Consider an API calling a database.

```
public async Task<List<Employee>> GetEmployeesAsync()
{
    return await _dbContext.Employees.ToListAsync();
}
```

What happens?

1. API receives the request.
2. Database query starts asynchronously.
3. Application thread is not blocked while waiting for database I/O.
4. Database returns the result.
5. Execution resumes and returns employees.

**Interview answer:**

"Async/await doesn't necessarily create a new thread. It allows the current method to suspend while an asynchronous operation is pending, freeing the thread to handle other work."

# **5. Sequential vs Concurrent API Calls**

### **Q7. You have three independent APIs. How do you improve performance?**

Sequential:

```
var employees = await GetEmployeesAsync();
var orders = await GetOrdersAsync();
var payments = await GetPaymentsAsync();
```

Concurrent:

```
var employeeTask = GetEmployeesAsync();
var orderTask = GetOrdersAsync();
var paymentTask = GetPaymentsAsync();

await Task.WhenAll(
    employeeTask,
    orderTask,
    paymentTask);

var employees = await employeeTask;
var orders = await orderTask;
var payments = await paymentTask;
```

**Real-world example:** Your dashboard needs employee details, order details and payment summaries from three independent services.

**Interview answer:** "I use Task.WhenAll to execute independent I/O-bound operations concurrently. This reduces total waiting time compared with sequential execution."

# **6. Race Condition and Thread Safety**

### **Q8. What is a race condition?**

Suppose two threads update the same employee counter.

```
private static int counter = 0;

Parallel.For(0, 10000, i =>
{
    counter++;
});

Console.WriteLine(counter);
```

Expected output: 10000.

But actual output might be less than 10000.

**Why?**

`counter++` internally involves reading, incrementing and writing. Two threads can read the same value and overwrite each other's updates.

**Solution 1: Interlocked**

```
private static int counter = 0;

Parallel.For(0, 10000, i =>
{
    Interlocked.Increment(ref counter);
});
```

**Solution 2: Lock**

```
private static int counter = 0;
private static readonly object locker = new();

Parallel.For(0, 10000, i =>
{
    lock (locker)
    {
        counter++;
    }
});
```

**Interview answer:**

"I use Interlocked for simple atomic operations like incrementing counters. For multiple statements that must execute safely as one operation, I use lock."

# **7. Lock vs Monitor vs Mutex**

### **Q9. How do you protect shared resources?**

Suppose two threads are updating the same bank account balance.

Using lock:

```
private static decimal balance = 10000;
private static readonly object locker = new();

public static void Withdraw(decimal amount)
{
    lock (locker)
    {
        if (balance >= amount)
        {
            balance -= amount;
        }
    }
}
```

The lock ensures that only one thread executes the critical section at a time.

Other synchronization mechanisms:

| Mechanism     | Real-world example                         |
| ------------- | ------------------------------------------ |
| lock          | Updating shared in-memory data             |
| Monitor       | Lock with timeout and signaling            |
| Mutex         | Shared resource across different processes |
| Semaphore     | Allow maximum five concurrent operations   |
| SemaphoreSlim | Asynchronous concurrency control           |

**Important interview point:** A C# `lock` is implemented using Monitor.Enter and Monitor.Exit semantics. Always use `lock` for ordinary synchronous critical sections unless you need additional Monitor capabilities.

# **8. Deadlock**

### **Q10. What is a deadlock? Give an example.**

Suppose:

- Thread 1 locks Resource A and waits for Resource B.
- Thread 2 locks Resource B and waits for Resource A.

**Thread 1**

Holds Lock A

Waiting for Lock B

Deadlock

**Thread 2**

Holds Lock B

Waiting for Lock A

Problematic code:

```
// Thread 1
lock (lockA)
{
    lock (lockB)
    {
        // Work
    }
}

// Thread 2
lock (lockB)
{
    lock (lockA)
    {
        // Work
    }
}
```

**Solution:** Always acquire locks in the same order.

```
// Both threads
lock (lockA)
{
    lock (lockB)
    {
        // Work
    }
}
```

**Interview answer:**

"I prevent deadlocks by maintaining a consistent lock acquisition order, minimizing lock duration and avoiding nested locks wherever possible."

# **9. SemaphoreSlim**

### **Q11. Suppose 100 users call your API, but you want only five requests to execute a particular operation simultaneously. How?**

Use SemaphoreSlim.

```
public class EmployeeService
{
    private static readonly SemaphoreSlim
        _semaphore = new(5, 5);

    public async Task ProcessAsync()
    {
        await _semaphore.WaitAsync();

        try
        {
            await Task.Delay(2000);

            Console.WriteLine("Processing completed");
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

**How it works:**

- First five requests acquire permits.
- Remaining requests wait asynchronously.
- When one request finishes, it releases its permit.
- The next waiting request proceeds.

**Real-world example:** You have a third-party payment API that allows only five concurrent requests. SemaphoreSlim helps control concurrency inside your application instance.

# **10. Task.WhenAll vs Task.WhenAny**

### **Q12. What is the difference?**

Example: You have three payment providers and need to call them.

WhenAll:

```
var tasks = new[]
{
    GetPaymentFromProvider1Async(),
    GetPaymentFromProvider2Async(),
    GetPaymentFromProvider3Async()
};

var results = await Task.WhenAll(tasks);
```

Waits for all three operations.

WhenAny:

```
var tasks = new[]
{
    GetPaymentFromProvider1Async(),
    GetPaymentFromProvider2Async(),
    GetPaymentFromProvider3Async()
};

var firstCompleted = await Task.WhenAny(tasks);

var result = await firstCompleted;
```

Returns the first completed task, whether successful or faulted.

**Real-world example:**

- WhenAll: Collect information from all required microservices.
- WhenAny: Race multiple equivalent providers to get the earliest response.

Important: WhenAny does not automatically cancel the other tasks. You must implement cancellation and failure handling yourself.

# **11. CancellationToken**

### **Q13. How do you cancel a long-running operation?**

Example: An API is processing 10,000 employee records, but the user cancels the request.

```
public async Task ProcessEmployeesAsync(
    CancellationToken cancellationToken)
{
    for (int i = 0; i < 10000; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();

        await ProcessEmployeeAsync(i, cancellationToken);
    }
}
```

Controller:

```
[HttpPost]
public async Task<IActionResult> Process(
    CancellationToken cancellationToken)
{
    await _service.ProcessEmployeesAsync(cancellationToken);

    return Ok("Completed");
}
```

ASP.NET Core can bind the request cancellation token automatically.

**Interview answer:**

"I pass CancellationToken from the API layer to the service and repository layers. This allows cancellation to propagate throughout the operation when the client disconnects or a timeout occurs."

# **12. Parallel.ForEach vs Parallel.ForEachAsync**

### **Q14. You have 10,000 images to process. Which approach will you use?**

For CPU-bound image processing:

```
Parallel.ForEach(
    images,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 4
    },
    image =>
    {
        ResizeImage(image);
    });
```

For asynchronous operations:

```
await Parallel.ForEachAsync(
    images,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 5
    },
    async (image, token) =>
    {
        await UploadImageAsync(image, token);
    });
```

**Difference:**

- Parallel.ForEach: Synchronous parallel execution.
- Parallel.ForEachAsync: Supports asynchronous operations and limits concurrency.

**Interview answer:** "I select the approach based on whether processing is CPU-bound or I/O-bound. I also control maximum concurrency to avoid overloading resources."

# **13. Task.Run vs async/await**

### **Q15. When should you use Task.Run?**

Suppose you have a CPU-intensive report generation operation.

```
public async Task<IActionResult> GenerateReport()
{
    var report = await Task.Run(() =>
    {
        return GenerateComplexReport();
    });

    return Ok(report);
}
```

Here Task.Run moves CPU-bound work onto a ThreadPool worker.

But for database operations:

Preferred

```
public async Task<IActionResult> GetEmployees()
{
    var employees = await _dbContext.Employees
        .ToListAsync();

    return Ok(employees);
}
```

Do not wrap EF Core asynchronous calls inside Task.Run.

**Interview answer:** "I use Task.Run when I need to offload CPU-intensive work. For I/O-bound work, I use native asynchronous APIs."

# **14. ThreadPool and ThreadPool Starvation**

### **Q16. What is ThreadPool starvation?**

Imagine your API receives 1,000 requests. Each request blocks a ThreadPool thread using `.Result`.

```
public IActionResult GetEmployees()
{
    var employees = GetEmployeesAsync().Result;

    return Ok(employees);
}
```

If many requests do this, worker threads can become blocked waiting for I/O.

This can lead to ThreadPool starvation and increased response times.

**Solution:**

```
public async Task<IActionResult> GetEmployees()
{
    var employees = await GetEmployeesAsync();

    return Ok(employees);
}
```

**How would you identify it in production?**

- Monitor ThreadPool thread count.
- Check ThreadPool queue length.
- Monitor API response time.
- Use dotnet-counters and dotnet-trace.
- Use Application Insights and OpenTelemetry for request traces.

# **15. ConfigureAwait(false)**

### **Q17. What is ConfigureAwait(false)?**

```
public async Task<string> GetDataAsync()
{
    var result = await CallExternalApiAsync()
        .ConfigureAwait(false);

    return result;
}
```

Normally, await may capture the current SynchronizationContext and resume on it.

`ConfigureAwait(false)` tells the await operation not to require resuming on that captured context.

**Where is it useful?**

- Reusable class libraries.
- Library code where returning to the original context isn't necessary.

In ASP.NET Core, there generally isn't a custom SynchronizationContext by default, so ConfigureAwait(false) is less important than in older ASP.NET or UI applications.

# **16. Async void vs Task**

### **Q18. Why should we avoid async void?**

Incorrect:

```
public async void SaveEmployeeAsync()
{
    await Task.Delay(1000);
}
```

Preferred:

```
public async Task SaveEmployeeAsync()
{
    await Task.Delay(1000);
}
```

**Why?**

- Task can be awaited.
- Exceptions can be observed by the caller.
- The caller can manage execution flow.
- Async void is mainly reserved for event handlers.

# **17. Fire-and-forget**

### **Q19. What is fire-and-forget? Is it safe?**

Example:

```
public IActionResult SendNotification()
{
    _ = SendEmailAsync();

    return Ok("Request received");
}
```

Problem:

- Exception may go unobserved.
- Application may shut down before completion.
- Scoped dependencies may be disposed.
- No durable retry mechanism.

**Production recommendation:**

For important background work:

```
API
 |
Azure Service Bus
 |
Azure Function
 |
Email Service
```

The API publishes a message and a background consumer processes it.

Use a durable queue when the work must survive application restarts.

# **18. Thread-safe collections**

### **Q20. How do you handle shared collections across multiple threads?**

Incorrect:

```
Dictionary<int, string> employees = new();

Parallel.For(0, 1000, i =>
{
    employees.TryAdd(i, $"Employee-{i}");
});
```

A normal Dictionary is not safe for concurrent writes.

Correct:

```
ConcurrentDictionary<int, string> employees = new();

Parallel.For(0, 1000, i =>
{
    employees.TryAdd(i, $"Employee-{i}");
});
```

Other useful collections:

| Collection           | Purpose                                    |
| -------------------- | ------------------------------------------ |
| ConcurrentDictionary | Thread-safe key-value collection           |
| ConcurrentQueue      | Thread-safe FIFO queue                     |
| ConcurrentStack      | Thread-safe LIFO collection                |
| BlockingCollection   | Producer-consumer with blocking operations |

\| Channel
