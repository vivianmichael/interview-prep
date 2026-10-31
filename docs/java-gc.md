# Java Garbage Collection

## What is garbage collection in Java?

Garbage collection (GC) automatically reclaims heap memory occupied by objects that are no longer reachable from the running application. Developers do not explicitly free individual Java objects, but they should still avoid retaining references longer than needed.

**Mental model:** An object is eligible for collection when no live reference path from a GC root reaches it. GC roots include active thread stacks, static references, and certain JVM-managed references.

## How does generational garbage collection work?

Most Java objects become unreachable soon after they are created. Generational collectors use this observation by organizing heap memory so that young objects can be collected frequently, while longer-lived objects are collected less often.

Objects that remain reachable through collections may be promoted to an older generation. The exact heap layout and collection behavior depend on the JVM and selected garbage collector.

## What are the common garbage collectors?

- **Serial GC:** Uses a single thread for collection; can suit small heaps or simple workloads.
- **Parallel GC:** Uses multiple threads to collect and aims for application throughput.
- **G1 GC:** Divides the heap into regions and balances throughput with pause-time goals.
- **ZGC and Shenandoah:** Designed to keep pauses very short, including for large heaps, with trade-offs in resource use and workload behavior.

Collector availability and defaults depend on the JDK version and runtime configuration. Choose based on measured application behavior rather than assuming one collector is best for every workload.

## What causes memory leaks in a garbage-collected application?

A memory leak occurs when an application unintentionally keeps references to objects it no longer needs, preventing GC from reclaiming them. Common causes include unbounded caches, listeners that are never unregistered, static collections, and `ThreadLocal` values left behind on reused threads.

**Production investigation:** Monitor heap usage and GC pause metrics. If memory continues to grow after collections, capture and compare heap dumps to identify objects retaining unexpectedly large object graphs. Fix the lifetime or ownership of those references rather than trying to force GC as a workaround.

## Does calling `System.gc()` guarantee collection?

No. `System.gc()` only requests that the JVM consider performing garbage collection; the JVM may ignore the request. Application correctness should not depend on it.

## What is the difference between a memory leak and high memory usage?

High memory usage is not automatically a leak. An application may legitimately need a large working set or may temporarily allocate many objects. A leak is indicated by retained objects that continue accumulating beyond their intended lifetime, often causing increasingly frequent collections or eventual `OutOfMemoryError`.

Use heap profiles and dumps to distinguish retained live data from temporary allocation pressure. Tune heap size or GC settings only after understanding the allocation and retention patterns.
