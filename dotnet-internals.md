# .NET Internals — Senior Interview Questions

## Thread Pool

### 1. What is the .NET ThreadPool and how does it manage threads?

**Hint:** Pre-allocated pool of worker threads managed by the CLR. Uses hill-climbing algorithm to adjust thread count. Avoids the cost of creating/destroying threads per task.

**🚩 Red Signal:** Cannot explain the difference between ThreadPool threads and dedicated threads, or thinks every `new Thread()` uses the pool.

---

### 2. What happens when the ThreadPool is starved?

**Hint:** New threads are injected slowly (~1 per 500ms). Long-running or blocking operations on pool threads delay other queued work items. Use `Task.Run` with `TaskCreationOptions.LongRunning` to avoid starvation.

**🚩 Red Signal:** No awareness that blocking calls (e.g., `Thread.Sleep`, synchronous I/O) on pool threads cause starvation.

---

### 3. How does `ThreadPool.SetMinThreads` / `SetMaxThreads` affect behavior?

**Hint:** `SetMinThreads` controls how many threads are immediately available before the hill-climbing algorithm kicks in. Useful to handle burst loads. Misconfiguration can waste resources or cause starvation.

**🚩 Red Signal:** Sets min threads to an arbitrarily high number without understanding implications or profiling.

---

## Garbage Collection (GC)

### 4. Explain the generational model of .NET GC.

**Hint:** Gen 0 (short-lived), Gen 1 (buffer), Gen 2 (long-lived). Objects promoted on survival. Gen 0/1 collections are fast; Gen 2 is expensive (full GC). LOH is collected with Gen 2.

**🚩 Red Signal:** Cannot describe why generations exist or says GC runs on every allocation.

---

### 5. What is the difference between Workstation and Server GC?

**Hint:** Workstation GC uses a single GC thread, optimized for low latency on client apps. Server GC uses one GC thread per logical CPU, higher throughput for server workloads. Configured via `<GCServer>` in project file or `runtimeconfig.json`.

**🚩 Red Signal:** Doesn't know Server GC exists or cannot explain when to choose one over the other.

---

### 6. What triggers a GC collection and how can you minimize GC pressure?

**Hint:** Triggered by Gen 0 threshold, `GC.Collect()`, or low memory. Reduce pressure by: pooling objects, using `Span<T>`/`Memory<T>`, avoiding allocations in hot paths, using value types where appropriate.

**🚩 Red Signal:** Suggests calling `GC.Collect()` manually in production code as a normal practice.

---

## Large Object Heap (LOH)

### 7. What is the LOH and why does it matter?

**Hint:** Objects ≥ 85,000 bytes go to LOH. LOH is not compacted by default (causes fragmentation). Collected only during Gen 2 collections. Arrays are the most common LOH allocations.

**🚩 Red Signal:** Unaware of the 85KB threshold or doesn't know LOH can fragment.

---

### 8. How can you mitigate LOH fragmentation?

**Hint:** Use `ArrayPool<T>` to rent/return arrays. Enable LOH compaction via `GCSettings.LargeObjectHeapCompactionMode` (one-shot). Avoid frequent large allocations. Consider `Span<T>` to slice existing memory.

**🚩 Red Signal:** No mention of `ArrayPool<T>` or doesn't understand that LOH compaction is not automatic.

---

## ArrayPool

### 9. What is `ArrayPool<T>` and when should you use it?

**Hint:** Provides reusable array buffers to avoid repeated allocations/GC pressure. `ArrayPool<T>.Shared` is the default pool. Always return rented arrays via `.Return()`. Rented arrays may be larger than requested.

**🚩 Red Signal:** Never heard of `ArrayPool<T>`, or forgets to return rented buffers (memory leak).

---

### 10. What is the difference between `ArrayPool<T>.Shared` and `ArrayPool<T>.Create()`?

**Hint:** `Shared` is a global singleton, thread-safe, suitable for most use cases. `Create()` lets you configure max array length and max arrays per bucket — useful when you need isolation or custom sizing.

**🚩 Red Signal:** Uses `Create()` everywhere without justification or doesn't understand the pooling lifecycle.

---

## Async/Await

### 11. What does `async`/`await` compile down to?

**Hint:** The compiler generates a state machine (`IAsyncStateMachine`). Each `await` is a suspension point. Continuations are posted to the captured `SynchronizationContext` or `TaskScheduler` (unless `ConfigureAwait(false)`).

**🚩 Red Signal:** Thinks `async` creates a new thread, or cannot explain state machine generation.

---

### 12. What is `ConfigureAwait(false)` and when should you use it?

**Hint:** Avoids capturing the `SynchronizationContext`, so the continuation runs on any thread pool thread. Use in library code to avoid deadlocks and improve performance. Do NOT use in UI code or ASP.NET contexts where context matters.

**🚩 Red Signal:** Uses `ConfigureAwait(false)` everywhere including UI code, or never uses it in library code.

---

### 13. Explain `ValueTask<T>` vs `Task<T>`.

**Hint:** `ValueTask<T>` is a struct that avoids heap allocation when the result is already available synchronously. Should not be awaited multiple times or stored. Use when hot paths frequently complete synchronously (e.g., cached results).

**🚩 Red Signal:** Replaces all `Task<T>` with `ValueTask<T>` without profiling, or awaits a `ValueTask` more than once.

---

### 14. How can `async void` cause problems?

**Hint:** Exceptions in `async void` methods crash the process (unobserved). Cannot be awaited. Only acceptable for event handlers. Always prefer `async Task`.

**🚩 Red Signal:** Uses `async void` outside of event handlers or is unaware of exception behavior.

---

### 15. What is the difference between `Task.Run` and `Task.Factory.StartNew`?

**Hint:** `Task.Run` is a shortcut that uses default scheduler and unwraps nested tasks. `StartNew` offers more control (scheduler, creation options) but does NOT unwrap — can produce `Task<Task>`. Prefer `Task.Run` for simple offloading.

**🚩 Red Signal:** Uses `StartNew` without understanding the unwrapping issue, or mixes them without reason.
