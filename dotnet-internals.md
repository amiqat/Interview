# .NET Internals

Deep-dive questions on Thread Pool, Garbage Collector, async/await, ArrayPool and the Large Object Heap.

---

### 1. How does the .NET Thread Pool manage worker threads, and what is the hill-climbing algorithm?

The Thread Pool maintains a pool of pre-created threads that grow or shrink based on workload. The hill-climbing algorithm adjusts the number of threads by measuring throughput: it adds a thread, observes if throughput improved, and continues or reverses the change.

**Hint:** Look for candidates who explain why creating a new OS thread per request is expensive (~1 MB stack), and how the pool reuses threads to amortise that cost. Bonus if they mention `ThreadPool.SetMinThreads` tuning and its effect on burst workloads.

**🚩 Red Signal:** Cannot explain why blocking Thread Pool threads (e.g., `Task.Result` inside an async pipeline) leads to thread starvation and deadlocks.

---

### 2. Describe the three generations of the .NET Garbage Collector and when each is collected.

- **Gen 0** – Short-lived objects (local variables, temporaries). Collected most frequently.
- **Gen 1** – Buffer between short-lived and long-lived. Collected less often.
- **Gen 2** – Long-lived objects (static data, caches). Full GC is expensive.

Collection is triggered when a generation's budget is exceeded. Gen 0 is fast (< 1 ms typically), while Gen 2 may involve a full compacting GC.

**Hint:** A strong answer includes the concept of generational hypothesis (most objects die young), the difference between workstation and server GC modes, and the impact of `GCSettings.LatencyMode`.

**🚩 Red Signal:** Confuses value types on the stack with reference types on the heap, or believes the GC runs on a fixed timer rather than on allocation pressure.

---

### 3. What is the Large Object Heap (LOH) and why does it matter for performance?

Objects ≥ 85,000 bytes are allocated on the LOH. The LOH is only collected during Gen 2 collections and is **not compacted by default**, which can cause fragmentation.

**Hint:** Great answers mention `GCSettings.LargeObjectHeapCompactionMode` introduced in .NET 4.5.1, pooling strategies to avoid LOH allocations (e.g., `ArrayPool<T>`), and how LOH fragmentation shows up in memory dumps.

**🚩 Red Signal:** Unaware that the LOH exists or thinks all heap allocations are treated equally by the GC.

---

### 4. Explain `async`/`await` in .NET. What happens under the hood when the compiler encounters an `await`?

The compiler transforms the async method into a state machine (`IAsyncStateMachine`). Each `await` is a suspension point: if the awaited task is not yet complete, the method yields control back to the caller and schedules a continuation. When the task completes, the continuation resumes on the captured `SynchronizationContext` (or the thread pool if there is none).

**Hint:** Look for understanding of `ConfigureAwait(false)` and why it matters in library code (avoids deadlocks, improves throughput by not marshalling back to the original context).

**🚩 Red Signal:** Believes `async` automatically runs code on a background thread, or cannot explain why `async void` is dangerous (unobserved exceptions crash the process).

---

### 5. What is `ConfigureAwait(false)` and when should you use it?

`ConfigureAwait(false)` tells the awaiter not to capture and resume on the original `SynchronizationContext`. This is critical in library code to avoid deadlocks (especially in legacy ASP.NET with a single-threaded `SynchronizationContext`) and to improve performance by avoiding unnecessary context switches.

**Hint:** In ASP.NET Core there is no `SynchronizationContext`, so `ConfigureAwait(false)` has less impact — but it is still a best practice in shared libraries that might also be consumed by UI or legacy ASP.NET apps.

**🚩 Red Signal:** Never heard of `ConfigureAwait` or dismisses it as irrelevant.

---

### 6. How does `ValueTask<T>` differ from `Task<T>` and when should you prefer it?

`ValueTask<T>` is a struct that can wrap either a `T` result (for synchronous completion) or a `Task<T>` (for asynchronous completion), avoiding a heap allocation when the result is available synchronously. It is ideal for hot paths where the async method completes synchronously most of the time, such as cached reads.

**Hint:** Candidates should know the restrictions: a `ValueTask<T>` can only be awaited once; you cannot call `.Result` or `.GetAwaiter().GetResult()` unless you know it has completed. `IValueTaskSource<T>` enables further pooling.

**🚩 Red Signal:** Cannot articulate any trade-off between `Task<T>` and `ValueTask<T>`, or believes `ValueTask<T>` is always better.

---

### 7. What is `ArrayPool<T>` and how does it help reduce GC pressure?

`ArrayPool<T>.Shared` rents and returns arrays from a pool instead of allocating new ones. This dramatically reduces Gen 0/Gen 1 allocations for transient buffers and avoids LOH allocations for large arrays.

**Hint:** Look for awareness of `ArrayPool<T>.Create()` for custom-sized pools, that rented arrays may be larger than requested, and the importance of always returning arrays (ideally in a `finally` block or with `using` via `MemoryPool<T>`).

**🚩 Red Signal:** Unaware that repeated large array allocations (e.g., `new byte[100_000]` per request) cause LOH fragmentation.

---

### 8. Explain thread starvation in the .NET Thread Pool. How do you diagnose and fix it?

Thread starvation occurs when all Thread Pool threads are blocked (e.g., synchronous waits on I/O, `Task.Wait()`, `Task.Result`) and the hill-climbing algorithm cannot inject new threads fast enough (it adds roughly 1-2 per second). Symptoms include increased latency, timeouts, and eventually deadlock.

**Hint:** Diagnosis: look at `ThreadPool.PendingWorkItemCount` or use PerfView / dotnet-counters to see thread pool queue length. Fix: replace blocking calls with `await`, increase `ThreadPool.SetMinThreads` as a temporary band-aid, or offload CPU-bound work to dedicated threads.

**🚩 Red Signal:** Suggests "just increase the thread count" without understanding the root cause or the cost of thousands of OS threads.

---

### 9. What are `Span<T>` and `Memory<T>` and how do they enable zero-allocation slicing?

`Span<T>` is a stack-only (`ref struct`) type that provides a type-safe, bounds-checked view over contiguous memory (arrays, stack-allocated buffers, native memory) without copying. `Memory<T>` is its heap-friendly counterpart that can be stored in fields and used across async boundaries.

**Hint:** Great answers include why `Span<T>` cannot be used in async methods (it is a ref struct), how `ReadOnlySpan<char>` eliminates `string.Substring` allocations, and the role of `MemoryPool<T>`.

**🚩 Red Signal:** Cannot explain why `Span<T>` is restricted to the stack or conflates it with `Array.

---

### 10. How does the .NET GC handle pinning, and what are the risks of excessive pinning?

Pinning (via `fixed` statement or `GCHandle.Alloc` with `GCHandleType.Pinned`) tells the GC not to move an object during compaction. This is required for P/Invoke and direct memory access but fragments the heap because the GC must work around pinned objects.

**Hint:** Look for knowledge of `Pinned Object Heap` (POH) introduced in .NET 5, which provides a dedicated heap for long-lived pinned objects and reduces Gen 0/Gen 1 fragmentation.

**🚩 Red Signal:** Uses `GCHandle.Alloc` for pinning but never calls `Free()`, causing memory leaks.
