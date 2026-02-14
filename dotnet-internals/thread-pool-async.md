# Thread Pool & Async/Await

---

### 1. 🟡 Your API handles ~200 requests/second normally, but during a burst to 2,000 req/s you see sudden latency spikes — p99 jumps from 50 ms to 5 seconds, then slowly recovers over 30+ seconds. CPU is not saturated. What is happening inside the Thread Pool, and how do you fix the slow ramp-up?

The .NET Thread Pool maintains a pool of pre-created worker threads and uses a hill-climbing algorithm to adjust the count: it adds a thread, measures whether throughput improved, and continues or reverses. This algorithm intentionally injects threads slowly (roughly 1–2 per second) to find the optimal count, which means a sudden burst can exhaust the current pool before new threads are created. During the gap, incoming work items queue up, causing the latency spike.

The fix depends on the workload. For predictable bursts, call `ThreadPool.SetMinThreads` to pre-warm a higher baseline (e.g., match your expected peak concurrency). This avoids the slow hill-climbing ramp entirely for that range. Long-term, audit the code for blocking calls on Thread Pool threads — each blocked thread is one fewer thread available for new requests, amplifying the starvation. Creating a new OS thread per request is not the answer either, since each thread costs ~1 MB of stack space.

**Hint:** A strong answer explains the hill-climbing algorithm and why the Thread Pool cannot instantly scale. Look for awareness of `ThreadPool.SetMinThreads` as a tuning lever for burst workloads, and that this is a band-aid — the real fix is ensuring threads aren't blocked. Bonus: candidate mentions monitoring `ThreadPool.ThreadCount` and `ThreadPool.PendingWorkItemCount` via `dotnet-counters`.

**🚩 Red Signal:** Cannot explain why blocking Thread Pool threads (e.g., `Task.Result` inside an async pipeline) leads to thread starvation and deadlocks, or suggests spawning `new Thread()` per request.

---

### 2. 🟡 A junior developer on your team refactored a synchronous service method to be "async" by wrapping the entire body in `Task.Run(() => { ... })` and adding `async`/`await`. The method does no I/O — it just computes a result. They claim it's now async and won't block. What's wrong with this approach, and what actually happens when the compiler encounters `await`?

Wrapping synchronous CPU-bound code in `Task.Run` does not make it truly asynchronous — it simply moves the same blocking work from the caller's thread to a Thread Pool thread. In a server scenario (e.g., ASP.NET Core), the request was already running on the Thread Pool, so this just adds overhead: a thread context switch, a state machine allocation, and no net gain in throughput. The original thread is released but a different Thread Pool thread is consumed.

When the compiler encounters `await`, it transforms the async method into a state machine implementing `IAsyncStateMachine`. Each `await` is a suspension point: if the awaited task is not yet complete, the method records its current state, yields control to the caller, and registers a continuation. When the task completes, the continuation resumes execution — on the captured `SynchronizationContext` if one exists, or on the Thread Pool otherwise. The value of `async`/`await` comes from truly asynchronous I/O (e.g., `HttpClient.GetAsync`, database calls), where no thread is held during the wait.

```csharp
// ❌ Fake async — just offloads to another thread pool thread
public async Task<Result> ComputeAsync(Input input)
{
    return await Task.Run(() => Compute(input));
}

// ✅ Keep synchronous code synchronous on the server
public Result Compute(Input input)
{
    // CPU-bound work — no benefit from async
    return HeavyCalculation(input);
}
```

**Hint:** Look for understanding that `async` does not mean "runs on a background thread" — it means the method can yield and resume. A strong answer distinguishes I/O-bound async (truly non-blocking) from CPU-bound work (which still consumes a thread). Ask the candidate: "What about `async void` — why is that dangerous?" (Unobserved exceptions crash the process.)

**🚩 Red Signal:** Believes `async` automatically runs code on a background thread, or cannot explain the state machine transformation that the compiler performs.

---

### 3. 🟡 You have a shared NuGet library used by both an ASP.NET Core API and a Blazor Server app. The API works fine, but the Blazor Server app deadlocks intermittently on calls to your library. You notice your library methods do not use `ConfigureAwait(false)`. Why does this cause a deadlock in Blazor Server but not in the API, and how do you fix it?

Blazor Server has a `SynchronizationContext` that enforces single-threaded access to the component's circuit (similar to a UI thread). When your library method `await`s without `ConfigureAwait(false)`, the continuation is posted back to that `SynchronizationContext`. If the calling code blocks synchronously (e.g., `.Result` or `.Wait()`) while holding that context, the continuation can never execute because the context is occupied — resulting in deadlock. ASP.NET Core, by contrast, has **no** `SynchronizationContext`, so continuations run on any available Thread Pool thread and the deadlock pattern does not manifest.

The fix is to add `ConfigureAwait(false)` to every `await` in your library code. This tells the awaiter not to capture and resume on the original context, allowing the continuation to run on any Thread Pool thread. This is a best practice for all shared/library code that does not need to touch UI or context-specific state. Even though it has less impact in ASP.NET Core (no context to capture), it protects consumers that do have a `SynchronizationContext` — Blazor Server, WPF, WinForms, and legacy ASP.NET.

```csharp
// ❌ Library code without ConfigureAwait — risky for Blazor/UI callers
public async Task<Data> GetDataAsync()
{
    var response = await _httpClient.GetAsync("/api/data");
    return await response.Content.ReadFromJsonAsync<Data>();
}

// ✅ Library code with ConfigureAwait(false)
public async Task<Data> GetDataAsync()
{
    var response = await _httpClient.GetAsync("/api/data").ConfigureAwait(false);
    return await response.Content.ReadFromJsonAsync<Data>().ConfigureAwait(false);
}
```

**Hint:** A strong answer explains that `ConfigureAwait(false)` tells the awaiter to skip capturing the `SynchronizationContext`, and correctly identifies which frameworks have one (Blazor Server, WPF, WinForms, legacy ASP.NET) vs. which don't (ASP.NET Core). Follow up: "If ASP.NET Core has no SynchronizationContext, is ConfigureAwait(false) pointless there?" (Not entirely — it's still a best practice in libraries for portability, and it avoids a minor overhead of checking for the context.)

**🚩 Red Signal:** Never heard of `ConfigureAwait`, dismisses it as irrelevant, or cannot explain why the same code deadlocks in one host but not another.

---

### 4. 🔴 You are profiling a high-throughput caching service and notice that `GetFromCacheAsync<T>()` returns the cached value synchronously ~95% of the time, yet allocates a `Task<T>` on every call. The allocation rate is millions per second and driving significant Gen 0 GC pressure. How do you eliminate these allocations, and what constraints does the solution introduce?

Replace `Task<T>` with `ValueTask<T>` as the return type. `ValueTask<T>` is a struct that can wrap either a `T` result directly (for synchronous completion) or an underlying `Task<T>` (for the rare asynchronous path). When the cache hits — the 95% case — `ValueTask<T>` returns the result with zero heap allocation, since the struct lives on the stack. Only on a cache miss does it fall back to a real `Task<T>` allocation.

```csharp
// ❌ Allocates Task<T> even on cache hits
public async Task<Item> GetFromCacheAsync(string key)
{
    if (_cache.TryGetValue(key, out var item))
        return item;
    return await LoadFromDatabaseAsync(key);
}

// ✅ Zero allocation on the hot path
public ValueTask<Item> GetFromCacheAsync(string key)
{
    if (_cache.TryGetValue(key, out var item))
        return new ValueTask<Item>(item);
    return new ValueTask<Item>(LoadFromDatabaseAsync(key));
}
```

The constraints are important: a `ValueTask<T>` can only be awaited **once** — you cannot `.Result` it multiple times, store it for later, or await it concurrently. If a caller violates this, the behavior is undefined. If you need to do any of those things, call `.AsTask()` to convert it to a regular `Task<T>`. For even more advanced scenarios, `IValueTaskSource<T>` allows pooling the underlying async state machine objects, further reducing allocations on the async path.

**Hint:** A strong answer covers the restrictions of `ValueTask<T>` (single-await, no concurrent consumption) and knows when not to use it — if the method almost always completes asynchronously, `Task<T>` is simpler and has no restrictions. Ask: "What is `IValueTaskSource<T>` and when would you implement it?" (For extreme perf: it enables pooling the objects backing the async path, avoiding even the fallback `Task<T>` allocation.)

**🚩 Red Signal:** Cannot articulate any trade-off between `Task<T>` and `ValueTask<T>`, believes `ValueTask<T>` is always better, or is unaware that `ValueTask<T>` can only be consumed once.

---

### 5. 🔴 Your production API starts returning HTTP 503s under moderate load. Monitoring shows Thread Pool queue length spiking above 500, thread count climbing slowly (1–2 threads/second), and most threads are blocked in `Task.Wait()` or `Task.Result` calls inside a legacy data access layer. Walk through what is happening, how you would diagnose this, and what the fix looks like.

This is classic Thread Pool starvation. The legacy data access layer is making synchronous blocking calls (`Task.Wait()`, `Task.Result`) on Thread Pool threads, consuming them while waiting for I/O. When all available threads are blocked, new incoming requests queue up. The Thread Pool's hill-climbing algorithm tries to inject new threads, but it does so conservatively — roughly 1–2 per second — which cannot keep pace with incoming load. The queue grows, latency explodes, and eventually Kestrel's request timeout fires 503s.

**Diagnosis:** Use `dotnet-counters` to monitor `ThreadPool.ThreadCount`, `ThreadPool.PendingWorkItemCount`, and `ThreadPool.CompletedWorkItemCount` in real time. PerfView or a production memory dump can show the stacks of blocked threads — you will see them parked on `ManualResetEventSlim.Wait` inside `Task.Wait()`. The `Microsoft.Extensions.Logging` thread pool saturation event can also fire warnings.

**Fix:** The correct fix is to replace the blocking calls with proper `await` calls — `await _dbContext.SaveChangesAsync()` instead of `_dbContext.SaveChanges()`, and so on, all the way up the call chain. As a **temporary band-aid**, you can increase `ThreadPool.SetMinThreads` to a higher value to survive the burst, but this does not fix the root cause and each additional OS thread costs ~1 MB of stack memory. Do not simply suggest "add more threads" — at thousands of blocked threads, you are trading memory and context-switching overhead for no real throughput gain.

**Hint:** A strong answer includes specific diagnostic tools (`dotnet-counters`, PerfView, thread pool event counters) and knows the difference between a band-aid (`SetMinThreads`) and the real fix (async all the way). Follow up: "What if the legacy library has no async API?" (Options: use a dedicated non-Thread Pool thread via `Task.Factory.StartNew` with `TaskCreationOptions.LongRunning`, or isolate the blocking work behind a bounded channel/queue with its own thread pool.)

**🚩 Red Signal:** Suggests "just increase the thread count" without understanding the root cause, or cannot explain why blocking calls on Thread Pool threads is fundamentally different from blocking on a dedicated thread.
