# Memory & Garbage Collection

---

### 1. 🟢 Your long-running background service processes messages from a queue. After a few hours, you notice GC pause times increasing and `dotnet-counters` shows Gen 2 collections spiking from 1–2/minute to 20+/minute. Gen 0 and Gen 1 counts look normal. What is likely happening, and how would you investigate?

Frequent Gen 2 collections indicate that many objects are surviving long enough to be promoted to Gen 2 and then dying there, or that Gen 2 is growing large enough to trigger compacting collections. The .NET GC is generational, based on the hypothesis that most objects die young: **Gen 0** collects short-lived objects most frequently (typically < 1 ms), **Gen 1** acts as a buffer between short- and long-lived, and **Gen 2** holds long-lived objects and is expensive to collect because it may involve a full compacting GC. When Gen 2 collections spike, it usually means objects are being allocated, promoted (surviving Gen 0 → Gen 1 → Gen 2), and then discarded — defeating the generational hypothesis.

Common causes include large caches with aggressive eviction, holding references just long enough for promotion (e.g., storing objects in a `List<T>` that gets cleared periodically), or excessive use of finalizers (which force objects to survive an extra collection cycle). Investigate with `dotnet-gcdump` to see which object types dominate Gen 2. Also check `GCSettings.IsServerGC` — server GC mode allocates a separate heap per logical core and is better for throughput workloads, while workstation GC minimizes pauses. Tuning `GCSettings.LatencyMode` to `SustainedLowLatency` can help latency-sensitive services at the cost of allowing Gen 2 to grow larger before collecting.

**Hint:** A strong answer includes the generational hypothesis (most objects die young), distinguishes workstation vs. server GC modes, and mentions `GCSettings.LatencyMode` as a tuning lever. Follow up: "What's the difference between a blocking Gen 2 collection and a background Gen 2 collection?" (Background GC runs concurrently with your code for most of the work, but a blocking Gen 2 pauses all threads.)

**🚩 Red Signal:** Confuses value types on the stack with reference types on the heap, believes the GC runs on a fixed timer rather than on allocation pressure, or cannot name the three generations.

---

### 2. 🟡 Your API's process memory grows steadily from 500 MB to 2 GB over 24 hours, then stabilizes. There are no obvious leaks — no growing collections, no event handler subscriptions. A memory dump shows the managed heap has plenty of free space, but it is heavily fragmented, and most of the free blocks are on the Large Object Heap. What is causing this, and how do you fix it?

Objects 85,000 bytes or larger are allocated on the Large Object Heap (LOH). Unlike the Small Object Heap, the LOH is only collected during Gen 2 collections and is **not compacted by default**. When large objects are allocated and freed repeatedly, the LOH accumulates free gaps between surviving objects — this is fragmentation. The total committed memory grows because the runtime cannot reuse those gaps efficiently, even though the objects themselves have been collected.

The fix has two parts. First, avoid LOH allocations where possible: use `ArrayPool<T>.Shared` to rent and return large arrays instead of allocating `new byte[100_000]` per request. This keeps the buffers alive and reused, eliminating the allocate/free churn that causes fragmentation. Second, for cases where LOH allocations are unavoidable, you can trigger LOH compaction on demand via `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce` followed by `GC.Collect()`. This is a one-shot setting (resets after the next Gen 2 GC) and is expensive, so use it sparingly — for example, during a maintenance window or after a large cache rebuild.

**Hint:** A strong answer explains the 85,000-byte threshold, why the LOH is not compacted by default (moving large objects is expensive), and names `ArrayPool<T>` as the primary mitigation. Follow up: "How would you confirm LOH fragmentation in a dump?" (Use `!dumpheap -stat` in WinDbg/dotnet-dump to see LOH segment utilization, or look at the `GC Heap` section in PerfView's GCStats.)

**🚩 Red Signal:** Unaware that the LOH exists, thinks all heap allocations are treated equally by the GC, or suggests calling `GC.Collect()` in a loop as a fix.

---

### 3. 🟡 Your high-throughput file upload API processes thousands of files per minute. Each request handler allocates `new byte[1_000_000]` for a read buffer, processes the file, and lets the buffer fall out of scope. Under load, you see high GC pressure — Gen 0 collections are frequent and you notice LOH allocations climbing. How do you redesign the buffer management to fix this?

Replace the per-request `new byte[1_000_000]` allocation with `ArrayPool<byte>.Shared.Rent(1_000_000)`. `ArrayPool<T>` maintains a pool of pre-allocated arrays; `Rent` retrieves one (which may be slightly larger than requested) and `Return` gives it back for reuse. This eliminates the repeated allocation and GC of million-byte arrays, and since the arrays stay alive in the pool, they are never collected and never re-allocated on the LOH.

```csharp
// ❌ Allocates on every request — LOH allocation, high GC pressure
public async Task ProcessUploadAsync(Stream input)
{
    var buffer = new byte[1_000_000];
    int bytesRead;
    while ((bytesRead = await input.ReadAsync(buffer)) > 0)
        Process(buffer.AsSpan(0, bytesRead));
}

// ✅ Rent from pool — zero allocation on the hot path
public async Task ProcessUploadAsync(Stream input)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(1_000_000);
    try
    {
        int bytesRead;
        while ((bytesRead = await input.ReadAsync(buffer)) > 0)
            Process(buffer.AsSpan(0, bytesRead));
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

Critical details: rented arrays may be larger than requested (the pool rounds up to bucket sizes), so you must track the actual data length and not assume `buffer.Length == 1_000_000`. Always return arrays in a `finally` block to avoid leaking them from the pool. For scenarios where you want `IDisposable` semantics, use `MemoryPool<T>` which wraps `ArrayPool<T>` and returns `IMemoryOwner<T>` — calling `Dispose()` returns the buffer automatically. You can also create a custom pool via `ArrayPool<T>.Create(maxArrayLength, maxArraysPerBucket)` if `Shared` defaults don't fit your workload.

**Hint:** A strong answer mentions that `Shared` returns arrays that may be larger than requested, the importance of always returning buffers, and `MemoryPool<T>` as the `IDisposable` wrapper. Follow up: "What happens if you forget to return the array?" (The pool doesn't crash — it just allocates a new array next time, and the orphaned array eventually becomes garbage, defeating the purpose.)

**🚩 Red Signal:** Unaware that `new byte[1_000_000]` exceeds the 85,000-byte LOH threshold, or suggests reducing the buffer size to 80 KB to dodge the LOH rather than addressing the allocation pattern.

---

### 4. 🔴 You are reviewing a CSV parsing library that reads multi-GB files. The current implementation uses `string.Split(',')` on each line, then `Substring` to extract fields, generating hundreds of thousands of short-lived `string` allocations per second. Gen 0 GC collections are dominating CPU time. How would you rewrite this parser for zero-allocation (or near-zero) field extraction?

Use `ReadOnlySpan<char>` and `MemoryExtensions` to slice the input without allocating new strings. Instead of `string.Split` (which allocates an array of new strings), iterate over the line with `span.IndexOf(',')` and use slicing to extract `ReadOnlySpan<char>` views into the original buffer. No new strings are allocated — each field is a pointer+length window into the existing memory.

```csharp
// ❌ Allocates strings for every field on every line
foreach (string line in File.ReadLines(path))
{
    string[] fields = line.Split(',');
    var name = fields[0];
    var value = int.Parse(fields[1]);
}

// ✅ Zero-allocation parsing with Span<T>
foreach (string line in File.ReadLines(path))
{
    ReadOnlySpan<char> remaining = line.AsSpan();
    int commaIndex = remaining.IndexOf(',');
    ReadOnlySpan<char> name = remaining[..commaIndex];
    ReadOnlySpan<char> value = remaining[(commaIndex + 1)..];
    var parsed = int.Parse(value);
}
```

`Span<T>` is a `ref struct` (stack-only), which means it cannot be stored in fields, boxed, or used across `await` boundaries. For async scenarios (e.g., reading chunks from a network stream), use `Memory<T>` / `ReadOnlyMemory<T>` instead — they are heap-friendly and can be passed across async continuations. You can always get a `Span<T>` from a `Memory<T>` via `.Span` when you need the fast, stack-only slicing within a synchronous block. For the file-reading layer itself, use `PipeReader` from `System.IO.Pipelines`, which provides `ReadOnlySequence<byte>` — a linked list of `Memory<byte>` buffers that avoids large contiguous allocations and integrates with `ArrayPool` internally.

**Hint:** A strong answer explains why `Span<T>` cannot cross async boundaries (ref struct lives on the stack, and the stack is unwound at each `await`), names `ReadOnlySpan<char>` as the replacement for `string.Substring`, and mentions `Memory<T>` as the async-safe counterpart. Follow up: "How does `System.IO.Pipelines` improve on manual buffer management?" (It handles buffer pooling, backpressure, and partial-read scenarios without manual `ArrayPool` juggling.)

**🚩 Red Signal:** Cannot explain why `Span<T>` is restricted to the stack, conflates it with `Array<T>`, or is unaware that `string.Substring` allocates a new string.

---

### 5. 🔴 Your application uses P/Invoke extensively to call a native imaging library, passing `byte[]` buffers that the native code reads from. After running under load, you see Gen 0 and Gen 1 heap fragmentation in diagnostic dumps — the GC is struggling to compact because objects are pinned during the native calls. How does pinning cause this, and what does .NET 5+ offer to fix it?

When you pin a managed object (via `fixed` statement or `GCHandle.Alloc` with `GCHandleType.Pinned`), you tell the GC it cannot move that object during compaction. The GC must work around pinned objects, leaving gaps in the heap that cannot be filled by compaction. If many short-lived objects are pinned across Gen 0/Gen 1 segments, the heap becomes a swiss cheese of pinned objects and free gaps, increasing the effective memory footprint and slowing collections as the GC navigates the fragmentation.

.NET 5 introduced the **Pinned Object Heap (POH)** — a dedicated heap specifically for objects that will be pinned. You allocate directly onto the POH using `GC.AllocateArray<T>(length, pinned: true)`. Objects on the POH are never moved (they don't need to be — the heap is designed for pinned data), so they don't interfere with Gen 0/Gen 1 compaction at all. This cleanly separates pinned and non-pinned allocations, eliminating the fragmentation problem.

```csharp
// ❌ Pinning a regular array — fragments Gen 0/Gen 1 during compaction
byte[] buffer = new byte[65536];
var handle = GCHandle.Alloc(buffer, GCHandleType.Pinned);
try
{
    IntPtr ptr = handle.AddrOfPinnedObject();
    NativeImaging.ProcessBuffer(ptr, buffer.Length);
}
finally
{
    handle.Free(); // Must always free — otherwise memory leak
}

// ✅ .NET 5+ — allocate on the Pinned Object Heap
byte[] buffer = GC.AllocateArray<byte>(65536, pinned: true);
// buffer is already pinned on the POH — no GCHandle needed
unsafe
{
    fixed (byte* ptr = buffer)
    {
        NativeImaging.ProcessBuffer((IntPtr)ptr, buffer.Length);
    }
}
```

For long-lived pinned buffers (e.g., reused I/O buffers), the POH is the ideal home. For short-lived scenarios, combine POH allocation with `ArrayPool<T>` — allocate pinned arrays into a pool at startup so they remain on the POH and are reused, getting the benefits of both pooling and pinned-heap isolation. Always ensure `GCHandle.Free()` is called when using the older pattern — forgetting to free is a common memory leak that will not be caught by the GC since the handle is a root.

**Hint:** A strong answer names the Pinned Object Heap (POH) and `GC.AllocateArray<T>(length, pinned: true)` as the .NET 5+ solution. Look for understanding that the `fixed` statement pins only for the duration of the block (short-lived pinning), while `GCHandle.Alloc` can pin indefinitely (long-lived pinning), and that both cause fragmentation on the regular heap. Follow up: "What happens if you forget `GCHandle.Free()`?" (The handle acts as a GC root — the object is never collected, causing a memory leak.)

**🚩 Red Signal:** Uses `GCHandle.Alloc` for pinning but is unaware that `Free()` must be called, or has never heard of the Pinned Object Heap despite working with .NET 5+.
