# .NET Internals

This category covers the runtime internals that separate senior .NET engineers from those who only use the framework at surface level. Understanding the Thread Pool, async/await machinery, Garbage Collector generations, and memory management primitives is essential for diagnosing production performance issues, avoiding subtle deadlocks, and writing code that scales under real-world load. These questions probe whether a candidate can reason about what the runtime does behind the scenes — not just call the APIs.

## Sub-Topics

| File | Focus Area | Questions |
|------|-----------|-----------|
| [Thread Pool & Async/Await](thread-pool-async.md) | Thread Pool management, async/await state machine, ConfigureAwait, ValueTask, thread starvation | 5 |
| [Memory & Garbage Collection](memory-gc.md) | GC generations, Large Object Heap, ArrayPool, Span/Memory, pinning & POH | 5 |
