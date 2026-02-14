# .NET Internals — Thread Pool, GC, async/await & Memory

---

### 1. 🟢 Name the three GC generations and what kind of objects each holds.

**Gen 0** holds short-lived objects (local variables, temporaries). **Gen 1** is a buffer — objects that survived one Gen 0 collection but haven't proven long-lived yet. **Gen 2** holds long-lived objects (static data, caches, objects that survived multiple collections). Gen 0/1 collections are fast (milliseconds); Gen 2 collections are expensive because they scan the entire managed heap.

**Hint:** Look for awareness that Gen 0 is collected most frequently and Gen 2 least. Strong follow-up: "What triggers a Gen 2 collection?" (LOH allocation, memory pressure, or explicit `GC.Collect()`.)

**🚩 Red Signal:** Cannot name the generations or believes all collections scan the entire heap equally.

---

### 2. 🟢 What does `async` do to a method at compile time?

The `async` keyword tells the C# compiler to transform the method into a state machine — a struct (or class) implementing `IAsyncStateMachine`. Each `await` becomes a suspension point: the state machine records which `await` it's at, yields control to the caller, and registers a continuation to resume when the awaited task completes. The `async` keyword itself does **not** create a thread or make anything run in the background.

**Hint:** A strong answer mentions the state machine by name and knows that `async` without `await` is just a synchronous method with overhead. Follow-up: "What's the difference between `async void` and `async Task`?" (Unobserved exceptions in `async void` crash the process.)

**🚩 Red Signal:** Believes `async` automatically runs code on a background thread, or has never heard of the state machine transformation.

---

### 3. 🟢 What is the Large Object Heap (LOH) threshold, and why does it matter?

Objects ≥ 85,000 bytes are allocated directly on the Large Object Heap. The LOH is only collected during Gen 2 collections, which are expensive. More critically, the LOH is **not compacted by default** — freed space leaves holes, causing fragmentation. Over time, you can have plenty of total free LOH memory but no single contiguous block large enough for a new allocation, leading to `OutOfMemoryException`.

**Hint:** Follow-up: "How can you compact the LOH?" (`GCSettings.LargeObjectHeapCompactionMode = CompactOnce` before a `GC.Collect()`.) "What about .NET's Pinned Object Heap?" (POH in .NET 5+ separates pinned objects to reduce fragmentation.)

**🚩 Red Signal:** Doesn't know the 85K threshold or is unaware that the LOH is not compacted by default.

---

### 4. 🟢 What's the difference between `Task.Run` and `Task.Factory.StartNew`?

`Task.Run` is a shorthand that queues work to the Thread Pool with sensible defaults (`TaskScheduler.Default`, `DenyChildAttach`). `Task.Factory.StartNew` offers more control — you can specify `TaskCreationOptions` (e.g., `LongRunning` to get a dedicated thread) and a custom `TaskScheduler` — but it returns `Task<Task>` when given an async delegate, which is a common bug. Prefer `Task.Run` for simple Thread Pool offloading; use `StartNew` only when you need `LongRunning` or a specific scheduler.

**Hint:** Ask: "When would you use `LongRunning`?" (Blocking I/O that can't be made async — it creates a dedicated OS thread instead of starving the pool.) "What's the `Task<Task>` unwrapping gotcha with `StartNew`?"

**🚩 Red Signal:** Uses `Task.Factory.StartNew` everywhere without knowing the unwrapping issue, or doesn't know `Task.Run` exists.

---

### 5. 🟡 How does the Thread Pool decide when to add or remove threads?

The Thread Pool uses a **hill-climbing algorithm**: it experimentally adds a thread, measures whether overall throughput improved, and continues in the same direction if it did — or reverses if it didn't. This converges on the optimal thread count for the current workload. The pool starts with `Environment.ProcessorCount` threads and injects new ones slowly (~1–2 per second) once all existing threads are busy. This intentionally slow ramp prevents over-subscription but means sudden bursts can exhaust the pool before new threads arrive.

Removing threads works the same way in reverse — if throughput stays stable with fewer threads, the algorithm removes them. You can override the minimum with `ThreadPool.SetMinThreads` for burst-heavy workloads, but this is a tuning lever, not a fix for blocking code.

**Hint:** A strong answer explains why creating threads on demand is expensive (~1 MB stack per OS thread) and mentions monitoring tools (`dotnet-counters` for `ThreadPool.ThreadCount` and `ThreadPool.PendingWorkItemCount`). Follow-up: "What's the relationship between the Thread Pool and I/O completion ports?"

**🚩 Red Signal:** Suggests solving thread starvation by setting `SetMinThreads(10000, 10000)` without investigating why threads are blocked, or doesn't know about the hill-climbing algorithm.

---

### 6. 🟡 Why is `ConfigureAwait(false)` important in library code?

When you `await` without `ConfigureAwait(false)`, the continuation is posted back to the captured `SynchronizationContext` (if one exists). In frameworks with a single-threaded context — Blazor Server, WPF, WinForms — this can cause deadlocks if the caller blocks synchronously (`.Result`, `.Wait()`). `ConfigureAwait(false)` tells the awaiter to skip capturing the context, allowing the continuation to run on any Thread Pool thread.

ASP.NET Core has **no** `SynchronizationContext`, so the deadlock pattern doesn't manifest there. But library code should always use `ConfigureAwait(false)` because you don't control who calls your library — a WPF app, a Blazor Server circuit, or legacy ASP.NET could all be consumers.

```csharp
// ✅ Library code — always ConfigureAwait(false)
public async Task<Data> GetDataAsync()
{
    var response = await _httpClient.GetAsync("/api/data").ConfigureAwait(false);
    return await response.Content.ReadFromJsonAsync<Data>().ConfigureAwait(false);
}
```

**Hint:** Follow-up: "If ASP.NET Core has no SynchronizationContext, is ConfigureAwait(false) pointless there?" (Not entirely — it's still best practice in libraries for portability and avoids the minor overhead of checking for a context.) A strong answer names which frameworks have a context (Blazor Server, WPF, WinForms, legacy ASP.NET).

**🚩 Red Signal:** Never heard of `ConfigureAwait`, dismisses it as irrelevant, or cannot explain why the same code deadlocks in one host but not another.

---

### 7. 🟡 What is `ArrayPool<T>` and why would you use it instead of `new byte[]`?

`ArrayPool<T>.Shared` rents and returns arrays from a reusable pool, avoiding repeated heap allocations. Every `new byte[N]` allocates on the managed heap — if `N ≥ 85,000` it goes straight to the LOH, and even smaller arrays create GC pressure at high throughput. With `ArrayPool<T>`, you rent a buffer, use it, and return it — the same memory is reused across requests.

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
try
{
    int bytesRead = await stream.ReadAsync(buffer.AsMemory(0, 4096));
    // process bytesRead
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

The rented array may be **larger** than requested (the pool rounds up to bucket sizes), so always track the actual length you need rather than using `buffer.Length`. Failing to return buffers causes the pool to allocate new ones, defeating the purpose.

**Hint:** Follow-up: "What happens if you forget to return the buffer?" (The pool allocates a new one — no crash, but you lose the pooling benefit and add GC pressure.) "What about `MemoryPool<T>`?" (It wraps `ArrayPool<T>` and returns `IMemoryOwner<T>` with `IDisposable` for safer lifetime management.)

**🚩 Red Signal:** Has never used `ArrayPool<T>`, or doesn't understand why avoiding allocations matters in high-throughput services.

---

### 8. 🟡 How does `ValueTask<T>` reduce allocations compared to `Task<T>`?

`ValueTask<T>` is a struct that can wrap either a `T` result directly (synchronous completion) or an underlying `Task<T>` (asynchronous completion). When a method completes synchronously — e.g., a cache hit — `ValueTask<T>` returns the result with zero heap allocation because the struct lives on the stack. A `Task<T>` would always allocate a heap object even for an immediately-available result.

```csharp
// ✅ Zero allocation on cache hit (the ~95% path)
public ValueTask<Item> GetFromCacheAsync(string key)
{
    if (_cache.TryGetValue(key, out var item))
        return new ValueTask<Item>(item);           // no allocation
    return new ValueTask<Item>(LoadFromDatabaseAsync(key)); // fallback to Task<T>
}
```

The constraint: a `ValueTask<T>` can only be awaited **once**. You cannot `.Result` it multiple times, store it, or await it concurrently — behavior is undefined if you do. Call `.AsTask()` if you need multi-consumption. Use `ValueTask<T>` when the synchronous completion path is the hot path; if the method almost always completes asynchronously, stick with `Task<T>`.

**Hint:** Ask: "What is `IValueTaskSource<T>` and when would you implement it?" (For extreme perf — it enables pooling the objects backing the async path, avoiding even the fallback `Task<T>` allocation.) Strong answer mentions the single-consumption rule unprompted.

**🚩 Red Signal:** Believes `ValueTask<T>` is always better than `Task<T>`, or is unaware of the single-consumption restriction.

---

### 9. 🟡 Your API's memory usage grows steadily over hours; a memory dump shows LOH fragmentation with many free gaps between large `byte[]` arrays. What's happening and how do you fix it?

The application is repeatedly allocating large byte arrays (≥ 85,000 bytes) on the LOH. Because the LOH is not compacted by default, when these arrays are freed they leave holes. Over time, the heap is full of gaps — total free memory may be sufficient but no contiguous block is large enough for the next allocation, so the runtime requests more memory from the OS. This pattern is common in services that process file uploads, image pipelines, or CSV parsing with large buffers.

**Fix:** Replace `new byte[N]` with `ArrayPool<byte>.Shared.Rent(N)` and return buffers after use — this reuses the same LOH memory instead of fragmenting it. For one-time compaction, set `GCSettings.LargeObjectHeapCompactionMode = CompactOnce` before triggering `GC.Collect()`, but this is a stop-gap. If you're on .NET 5+, pinned buffers (e.g., for P/Invoke or socket I/O) should use `GC.AllocateArray<byte>(size, pinned: true)` to allocate on the **Pinned Object Heap (POH)** instead, isolating pinned allocations from the regular LOH.

**Hint:** Strong answer mentions `ArrayPool<T>` as the primary fix and LOH compaction as emergency-only. Follow-up: "How would you identify which allocations are going to the LOH?" (Use `dotnet-dump` + `dumpheap -stat` filtered by size, or PerfView's GC heap allocation traces.) "What's the POH?" (.NET 5+ heap specifically for pinned objects, reducing LOH fragmentation.)

**🚩 Red Signal:** Suggests calling `GC.Collect()` in a loop as the fix, or doesn't know what the LOH is.

---

### 10. 🔴 Your production API starts returning HTTP 503s under moderate load. Thread Pool queue length is spiking above 500, thread count is climbing slowly (1–2/second), and most threads are blocked in `Task.Wait()` or `Task.Result` inside a legacy data access layer. Walk through diagnosis and fix.

This is classic **Thread Pool starvation**. The legacy layer makes synchronous blocking calls (`.Wait()`, `.Result`) on Thread Pool threads, consuming them while waiting for I/O. When all available threads are blocked, incoming requests queue up. The Thread Pool's hill-climbing algorithm injects threads at ~1–2/second — far too slow to keep pace with incoming load. The queue grows, latency explodes, and Kestrel's request timeout fires 503s.

**Diagnosis:** Use `dotnet-counters` to monitor `ThreadPool.ThreadCount`, `ThreadPool.PendingWorkItemCount`, and `ThreadPool.CompletedWorkItemCount`. PerfView or a production memory dump will show threads parked on `ManualResetEventSlim.Wait` inside `Task.Wait()`. The `Microsoft.Extensions.Logging` thread pool saturation warning event may also fire.

**Fix:** Replace blocking calls with proper `await` — `await _dbContext.SaveChangesAsync()` instead of `_dbContext.SaveChanges()`, async all the way up the call chain. As a **temporary band-aid**, increase `ThreadPool.SetMinThreads` to survive the burst, but each additional OS thread costs ~1 MB of stack memory. If the legacy library has no async API, isolate it behind a bounded channel or use `Task.Factory.StartNew` with `TaskCreationOptions.LongRunning` to get dedicated threads that don't starve the pool.

**Hint:** Look for specific diagnostic tools (`dotnet-counters`, PerfView, event counters) and understanding that `SetMinThreads` is a band-aid, not a fix. Follow-up: "What if the legacy library has no async API at all?" (Options: `LongRunning` threads, bounded channel/queue with dedicated workers, or wrap in a separate process.)

**🚩 Red Signal:** Suggests "just increase the thread count" without investigating root cause, or cannot explain why blocking on Thread Pool threads differs from blocking on a dedicated thread.

---

### 11. 🔴 A CSV parser in your pipeline uses `string.Split(',')` on each line, generating millions of small string allocations per file. How would you optimize this with `Span<T>` and related types?

`string.Split` allocates a new `string[]` plus individual `string` objects for every segment on every line — at millions of lines, this dominates GC pressure. The fix is to use `ReadOnlySpan<char>` to slice the original line without allocating new strings.

```csharp
// ✅ Zero-allocation field parsing with Span<T>
ReadOnlySpan<char> line = lineText.AsSpan();
while (!line.IsEmpty)
{
    int comma = line.IndexOf(',');
    ReadOnlySpan<char> field = comma == -1 ? line : line[..comma];

    // Parse directly from span — no string allocation
    int value = int.Parse(field);

    line = comma == -1 ? ReadOnlySpan<char>.Empty : line[(comma + 1)..];
}
```

Key techniques: `Span<T>` slicing creates views over existing memory (no copy, no allocation). `int.Parse(ReadOnlySpan<char>)` and similar overloads parse directly from spans. For fields you must store as strings, use `string.Create` or an interning strategy to minimize unique allocations. `MemoryExtensions.Split` (available in .NET 8+) provides a span-based tokenizer. For the overall read loop, use `PipeReader` from `System.IO.Pipelines` to process the file in pooled buffer chunks rather than reading entire lines into strings.

**Hint:** Strong answer mentions that `Span<T>` is stack-only (cannot be stored in heap objects, async methods, or closures) and knows the workaround: use `Memory<T>` or process synchronously. Follow-up: "Can you use `Span<T>` inside an `async` method?" (No — it's a `ref struct`. Use `Memory<T>` or extract the span-based work into a synchronous helper.)

**🚩 Red Signal:** Unfamiliar with `Span<T>`, suggests `StringBuilder` as the optimization, or doesn't understand why reducing allocations matters for throughput.

---

### 12. 🔴 A service using P/Invoke to call native libraries has growing heap fragmentation. Memory dumps show many pinned `byte[]` objects scattered across the heap. What's happening and how does the .NET 5+ Pinned Object Heap (POH) help?

When you pin a managed object (via `GCHandle.Alloc(obj, GCHandleType.Pinned)` or `fixed` blocks) for native interop, the GC cannot move that object during compaction. Pinned objects scattered throughout Gen 0/1/2 create "holes" — the GC compacts around them, leaving fragmented free space. With many pinned buffers (common in socket I/O, P/Invoke-heavy code), this fragments the entire heap, increasing GC pause times and memory usage.

.NET 5 introduced the **Pinned Object Heap (POH)** — a dedicated heap segment for objects that will be pinned for their lifetime. Allocate with `GC.AllocateArray<byte>(size, pinned: true)`. Objects on the POH are born pinned and live on a separate segment, so they never interfere with compaction of the regular generational heap. The GC knows these objects won't move and doesn't attempt to compact around them.

```csharp
// ❌ Pinning in the regular heap — causes fragmentation
byte[] buffer = new byte[4096];
GCHandle handle = GCHandle.Alloc(buffer, GCHandleType.Pinned);

// ✅ Allocate directly on the POH — no fragmentation of the regular heap
byte[] buffer = GC.AllocateArray<byte>(4096, pinned: true);
```

The trade-off: POH objects are never compacted (similar to the LOH), so you should still pool and reuse them. The POH solves fragmentation of the *regular* heap, not fragmentation within the POH itself. Combine with `ArrayPool<byte>` or a custom pool for reuse.

**Hint:** A strong answer distinguishes between the LOH (large objects), the POH (pinned objects), and the regular generational heap. Follow-up: "When would you still use `GCHandle` instead of the POH?" (Short-lived pins during a single P/Invoke call — `fixed` blocks are fine; POH is for long-lived pinned buffers.) "Does the POH exist on all GC modes?" (Yes — both Workstation and Server GC.)

**🚩 Red Signal:** Doesn't understand what pinning means, confuses the POH with the LOH, or is unaware that pinning affects GC compaction.
