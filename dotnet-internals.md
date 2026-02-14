# .NET Internals — Senior Interview Questions

> **Time:** ~20 min · **Pick:** 5-7 questions · **Start with** 🟢 then go deeper

---

## Thread Pool

### 🟢 1. What is the .NET ThreadPool and how does it manage threads?

**Hint:** Pre-allocated pool of worker threads managed by the CLR. Uses hill-climbing algorithm to adjust thread count dynamically. Two types of threads: worker threads (for `Task.Run`, `ThreadPool.QueueUserWorkItem`) and I/O completion port (IOCP) threads (for async I/O callbacks). Avoids the overhead of creating/destroying OS threads per task.

**Follow-up:** What's the default min/max thread count? How does the hill-climbing algorithm decide to add or remove threads?

**🚩 Red Signal:** Cannot explain the difference between ThreadPool threads and dedicated threads, or thinks every `new Thread()` uses the pool.

---

### 🟡 2. What happens when the ThreadPool is starved? How do you detect and fix it?

**Hint:** New threads are injected slowly (~1 per 500ms). Long-running or blocking operations on pool threads delay other queued work items. Symptoms: increasing response times, timeouts, thread count growing unbounded.

**Detection:** Monitor `ThreadPool.ThreadCount`, `ThreadPool.PendingWorkItemCount`. Use `dotnet-counters` or Application Insights to track thread pool queue length.

**Fix:** Avoid blocking calls on pool threads. Use true async I/O. Use `Task.Run` with `TaskCreationOptions.LongRunning` for CPU-bound long work (creates a dedicated thread). Consider `System.Threading.Channels` for producer-consumer patterns.

**Follow-up:** Walk me through a real scenario where you encountered thread pool starvation. How did you diagnose it?

**🚩 Red Signal:** No awareness that blocking calls (`Thread.Sleep`, sync-over-async, `Task.Result`) on pool threads cause starvation.

---

### 🟡 3. How does `ThreadPool.SetMinThreads` / `SetMaxThreads` affect behavior?

**Hint:** `SetMinThreads` controls how many threads are immediately available before the hill-climbing algorithm kicks in. Useful to handle burst loads. `SetMaxThreads` caps the total threads. Misconfiguration can waste resources or cause starvation.

```csharp
// Example: Handle burst of 100 concurrent requests
ThreadPool.SetMinThreads(workerThreads: 100, completionPortThreads: 100);
```

**Follow-up:** In what scenario would you increase `SetMinThreads`? What's the downside of setting it too high?

**🚩 Red Signal:** Sets min threads to an arbitrarily high number without profiling, or doesn't know these APIs exist.

---

## Garbage Collection (GC)

### 🟢 4. Explain the generational model of .NET GC.

**Hint:** Gen 0 (short-lived, ~256 KB), Gen 1 (buffer between short/long-lived), Gen 2 (long-lived objects). Objects promoted on survival. Gen 0/1 collections are fast (milliseconds); Gen 2 is expensive (full GC, can pause the app). LOH is collected alongside Gen 2.

**Key point:** ~90% of objects die in Gen 0. The generational hypothesis: most objects are short-lived, so collecting young objects frequently is efficient.

**Follow-up:** What is a "GC root"? Name some common GC roots. What happens during the "mark" phase?

**🚩 Red Signal:** Cannot describe why generations exist or says GC runs on every allocation.

---

### 🟡 5. What is the difference between Workstation and Server GC?

**Hint:** Workstation GC: single GC heap + single GC thread. Optimized for low latency on client apps. Server GC: one heap per logical CPU + one GC thread per heap. Higher throughput for server workloads, but more memory.

```xml
<!-- csproj -->
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

**Concurrent vs background GC:** Concurrent GC (default with Server GC) does most of the work on background threads, reducing pause times. Non-concurrent GC suspends all managed threads during collection.

**Follow-up:** When would you switch from Server GC to Workstation GC? What about containerized workloads with limited CPUs?

**🚩 Red Signal:** Doesn't know Server GC exists or cannot explain when to choose one over the other.

---

### 🟡 6. What triggers a GC collection and how can you minimize GC pressure?

**Hint:** Triggered by: Gen 0 budget exceeded, `GC.Collect()` called, OS low memory notification. Reduce pressure by:
- Pool objects (`ObjectPool<T>`, `ArrayPool<T>`)
- Use `Span<T>` / `Memory<T>` for slicing without allocations
- Use `stackalloc` for small, short-lived buffers
- Avoid allocations in hot paths (e.g., closures in LINQ, string concatenation in loops)
- Prefer `struct` over `class` for small value-like types

**Follow-up:** How do you profile allocations? What tools do you use? (dotnet-trace, dotnet-gcdump, PerfView, BenchmarkDotNet `[MemoryDiagnoser]`)

**🚩 Red Signal:** Suggests calling `GC.Collect()` manually in production code as a standard practice.

---

### 🔴 7. Explain Pinned Object Heap (POH) introduced in .NET 5.

**Hint:** Objects that are pinned (for interop with native code or `fixed` keyword) were previously problematic — they prevented compaction and caused fragmentation. POH is a dedicated heap for pinned objects, keeping them out of the normal GC heaps to allow compaction of Gen 0/1/2.

**Follow-up:** When would pinning be needed? Give an example with P/Invoke or `fixed` statement.

**🚩 Red Signal:** Unaware that pinned objects exist or that they affect GC compaction.

---

## Large Object Heap (LOH)

### 🟢 8. What is the LOH and why does it matter?

**Hint:** Objects ≥ 85,000 bytes (~85 KB) go to LOH. LOH is NOT compacted by default — causes fragmentation over time. Collected only during Gen 2 collections (expensive). Arrays and large strings are the most common LOH allocations.

**Follow-up:** Why is the threshold 85,000 bytes? (Historical implementation detail — cost of copying large objects exceeds the benefit of compaction.)

**🚩 Red Signal:** Unaware of the 85 KB threshold or doesn't know LOH can fragment.

---

### 🟡 9. How can you mitigate LOH fragmentation?

**Hint:** 
- Use `ArrayPool<T>` to rent/return arrays (avoid repeated allocations)
- Enable one-shot LOH compaction: `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce`
- Avoid frequent large short-lived allocations
- Use `Span<T>` / `Memory<T>` to slice existing buffers
- Use `RecyclableMemoryStream` (Microsoft library) instead of `MemoryStream` for large streams

**Follow-up:** How would you detect LOH fragmentation in production? What metrics would you monitor?

**🚩 Red Signal:** No mention of `ArrayPool<T>` or doesn't understand that LOH compaction is not automatic.

---

## ArrayPool

### 🟢 10. What is `ArrayPool<T>` and when should you use it?

**Hint:** Provides reusable array buffers to avoid repeated allocation/GC pressure. `ArrayPool<T>.Shared` is the default thread-safe pool. Always return rented arrays via `.Return()`. Rented arrays may be **larger** than requested — use the requested length, not `array.Length`.

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(minimumLength: 4096);
try
{
    // Use buffer[0..4096], NOT buffer.Length (may be larger)
    int bytesRead = stream.Read(buffer, 0, 4096);
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
}
```

**Follow-up:** What happens if you forget to return a rented buffer? (Not a memory leak per se — GC will collect it — but the pool can't reuse it, defeating the purpose.)

**🚩 Red Signal:** Never heard of `ArrayPool<T>`, or forgets to return rented buffers.

---

### 🟡 11. What is the difference between `ArrayPool<T>.Shared` and `ArrayPool<T>.Create()`?

**Hint:** `Shared` is a global singleton, thread-safe, suitable for most cases. Max array length: ~1 MB. `Create()` lets you configure `maxArrayLength` and `maxArraysPerBucket` — useful for isolation or when you need larger arrays. Custom pools are also thread-safe.

**Follow-up:** When would you need `ArrayPool<T>.Create()` over `Shared`? (When the shared pool's max length is too small, or when you want to control memory budget per subsystem.)

**🚩 Red Signal:** Uses `Create()` everywhere without justification or doesn't understand the pool lifecycle.

---

### 🟡 12. How does `MemoryPool<T>` differ from `ArrayPool<T>`?

**Hint:** `MemoryPool<T>` returns `IMemoryOwner<T>` which wraps `Memory<T>`. Supports `IDisposable` for deterministic return. Default implementation delegates to `ArrayPool<T>.Shared`. More convenient for async code where `Memory<T>` is preferred over `T[]`.

**Follow-up:** What is the relationship between `Span<T>`, `Memory<T>`, and `ArrayPool<T>`?

**🚩 Red Signal:** Cannot explain the Span/Memory ecosystem at all.

---

## Async/Await

### 🟢 13. What does `async`/`await` compile down to?

**Hint:** The compiler generates a state machine struct implementing `IAsyncStateMachine`. Each `await` is a suspension point. When the awaited task completes, the continuation is posted to the captured `SynchronizationContext` or `TaskScheduler` (unless `ConfigureAwait(false)`).

```csharp
// This:
async Task<int> GetDataAsync()
{
    var result = await httpClient.GetStringAsync(url);
    return result.Length;
}

// Compiles roughly to a state machine:
// struct GetDataAsync_StateMachine : IAsyncStateMachine
// {
//     int state;
//     AsyncTaskMethodBuilder<int> builder;
//     TaskAwaiter<string> awaiter;
//     void MoveNext() { /* switch on state */ }
// }
```

**Follow-up:** How many allocations does a typical `async` method produce? When does it allocate zero? (When the task completes synchronously — the state machine stays on the stack.)

**🚩 Red Signal:** Thinks `async` creates a new thread, or cannot explain state machine generation.

---

### 🟡 14. What is `ConfigureAwait(false)` and when should you use it?

**Hint:** Avoids capturing the `SynchronizationContext`, so the continuation runs on any available thread pool thread. 

**Use in:** Library code (always), to avoid deadlocks and improve performance.
**Don't use in:** UI code (needs UI thread), ASP.NET code that accesses `HttpContext` after await.

**Note:** In ASP.NET Core there is no `SynchronizationContext` by default, so `ConfigureAwait(false)` has no effect on context capture — but it's still good practice in library code for portability.

**Follow-up:** What deadlock scenario does `ConfigureAwait(false)` prevent? (Classic `.Result`/`.Wait()` on a UI/legacy ASP.NET thread holding the sync context.)

**🚩 Red Signal:** Uses `ConfigureAwait(false)` everywhere including UI code, or never uses it in library code.

---

### 🟡 15. Explain `ValueTask<T>` vs `Task<T>`.

**Hint:** `ValueTask<T>` is a `struct` that avoids heap allocation when the result is already available synchronously. Wraps either a `T` value or a `Task<T>`.

**Rules:**
- Never await a `ValueTask` more than once
- Never call `.GetAwaiter().GetResult()` before completion
- Never use `Task.WhenAll` / `Task.WhenAny` with `ValueTask` (convert with `.AsTask()` first)

**Use when:** Hot paths that frequently complete synchronously (e.g., reading from a cache, `PipeReader.ReadAsync`).

**Follow-up:** What is `IValueTaskSource<T>` and when would you implement it? (Advanced: for custom async state machines that want to reuse the same object for multiple operations.)

**🚩 Red Signal:** Replaces all `Task<T>` with `ValueTask<T>` without profiling, or awaits a `ValueTask` more than once.

---

### 🟢 16. How can `async void` cause problems?

**Hint:** Exceptions in `async void` methods are posted to the `SynchronizationContext` — if there's no handler, they crash the process. Cannot be awaited, so the caller doesn't know when it completes or if it failed.

**Only acceptable for:** Event handlers (`button_Click`, `Loaded`, etc.)

```csharp
// BAD — crashes the process on exception
async void ProcessData() { await FetchAsync(); }

// GOOD — exception is captured in the Task
async Task ProcessDataAsync() { await FetchAsync(); }
```

**Follow-up:** How do unobserved task exceptions behave in .NET 6+? (`TaskScheduler.UnobservedTaskException` fires but does NOT crash the process by default since .NET 4.5.)

**🚩 Red Signal:** Uses `async void` outside of event handlers or is unaware of exception behavior.

---

### 🟡 17. What is the difference between `Task.Run` and `Task.Factory.StartNew`?

**Hint:** `Task.Run` is a shortcut that uses the default scheduler and unwraps nested tasks. `StartNew` offers more control (scheduler, creation options, cancellation) but does NOT unwrap — can produce `Task<Task>`.

```csharp
// Task.Run — simple and safe
var result = await Task.Run(() => ComputeExpensive());

// StartNew — DANGER: returns Task<Task<int>>, must Unwrap
var task = Task.Factory.StartNew(() => ComputeExpensiveAsync());
var result = await task.Unwrap(); // Easy to forget!
```

**Follow-up:** When would you genuinely need `StartNew` over `Task.Run`? (When you need a custom `TaskScheduler` or `LongRunning` flag.)

**🚩 Red Signal:** Uses `StartNew` without understanding the unwrapping issue, or mixes them without reason.

---

### 🔴 18. What are `System.Threading.Channels` and how do they compare to `BlockingCollection<T>`?

**Hint:** Channels are a high-performance, async-native producer-consumer data structure introduced in .NET Core 3.0. Bounded or unbounded. `BlockingCollection<T>` blocks threads (bad for async code). Channels use `await` for reading/writing, so they integrate with the async model without wasting thread pool threads.

```csharp
var channel = Channel.CreateBounded<WorkItem>(capacity: 100);

// Producer
await channel.Writer.WriteAsync(new WorkItem(...));

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync())
{
    await ProcessAsync(item);
}
```

**Follow-up:** How do you handle backpressure with bounded channels? What happens when the channel is full?

**🚩 Red Signal:** Uses `BlockingCollection<T>` in async code or doesn't know `Channel<T>` exists.

---

## Scenario Questions

### 💡 19. Your API is experiencing intermittent timeouts under load. Thread count keeps growing. What do you investigate?

**Expected approach:**
1. Check for sync-over-async (`Task.Result`, `.Wait()`, `.GetAwaiter().GetResult()`) causing thread pool starvation
2. Profile with `dotnet-counters` — monitor `ThreadPool Thread Count`, `ThreadPool Queue Length`, `Monitor Lock Contention Count`
3. Check for blocking I/O (sync database calls, sync HTTP calls)
4. Look for `Thread.Sleep` or long-running work on pool threads
5. Consider if `SetMinThreads` needs adjustment as a temporary fix while root cause is addressed

**🚩 Red Signal:** Immediately suggests adding more servers instead of diagnosing the root cause.

---

### 💡 20. You have a service processing 10,000 messages/sec. Memory keeps growing until OOM. Where do you look?

**Expected approach:**
1. Take a memory dump and analyze with `dotnet-gcdump` or Visual Studio
2. Check Gen 2 / LOH size — likely LOH fragmentation from large arrays
3. Look for missing `ArrayPool<T>` usage — large byte arrays being allocated and discarded
4. Check for `MemoryStream` growth — switch to `RecyclableMemoryStream`
5. Look for event handler leaks (subscribing without unsubscribing)
6. Check if `Span<T>` / `Memory<T>` could eliminate intermediate array allocations

**🚩 Red Signal:** Suggests increasing server memory or calling `GC.Collect()` as the fix.

---

### 💡 21. A colleague wraps every async call in `Task.Run`. Is this correct?

**Expected answer:** No. `Task.Run` offloads work to the thread pool — for I/O-bound async operations, the work is already non-blocking. Wrapping `await httpClient.GetAsync()` in `Task.Run` just wastes a thread pool thread. `Task.Run` is for CPU-bound work that you want to move off the calling thread (e.g., UI thread).

**🚩 Red Signal:** Thinks `Task.Run` is required to make something async.

---

### 💡 22. How would you reduce the memory footprint of a service that handles large file uploads?

**Expected approach:**
1. Stream the file — don't load the entire file into memory (`IFormFile.OpenReadStream()`)
2. Use `ArrayPool<byte>.Shared` for buffer allocation
3. Use `PipeReader` / `PipeWriter` (System.IO.Pipelines) for efficient streaming
4. Set upload size limits in Kestrel config
5. Process in chunks, write to disk/blob storage incrementally
6. Avoid `byte[]` copying — use `Memory<T>` / `ReadOnlyMemory<T>`

**🚩 Red Signal:** Reads the entire file into a `byte[]` with `ReadAllBytesAsync` and holds it in memory.
