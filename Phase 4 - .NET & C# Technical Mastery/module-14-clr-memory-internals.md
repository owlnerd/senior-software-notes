# Module 14 — CLR & Memory Internals
*Phase 4: .NET & C# Technical Mastery · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **the CLR does not free you from memory management — it moves the decision from "when do I free this?" to "how much work am I handing the collector, and when will it choose to do that work?", and the answer to the second question shows up in your p99 latency, your container memory limit, and your cloud bill.**

That reframing matters because the naive picture of .NET memory is *"the GC handles it."* The real picture is that the GC handles it **on a schedule you do not control, with a cost proportional to what survives, in pauses that land wherever they land.** Every type you declare, every buffer you allocate, every cache you add, and every `async` method you write is an input to that system. A mid-level candidate can define "stack vs heap." A senior candidate can tell you why their service's memory doubled when it moved from a 4-core to a 32-core node, why a 300 ms pause appeared only under load, and why the fix was a `struct` in one case and an `ArrayPool` in another — and when the right answer was to change nothing.

This module opens **Phase 4**, and it is the foundation the rest of the phase stands on. Module 15 (async/await and concurrency) needs the state-machine allocation model from Part E and the suspension model from Part C. Module 17 (performance engineering) is essentially Parts E and F applied with a measurement discipline. Module 18 (ASP.NET Core internals) inherits the DI lifetime traps that cause the leaks in Part F. Module 19 (EF Core) inherits change-tracker retention, which is a textbook mid-life-crisis problem. It also closes a loop from Phase 3: Module 13's cascading failures frequently *begin* as a GC problem — a heap under pressure raises latency, latency raises concurrency, concurrency raises allocation, and the loop closes.

This module has eight jobs:

1. **Replace folklore with a model.** "Value types go on the stack" is wrong, "structs are faster" is wrong, "the GC compacts everything" is wrong, and "`Dispose` frees memory" is wrong. Each of these is a common interview answer and each one marks a candidate as mid-level.
2. **Explain what the runtime actually does** between your IL and the machine code that runs: type loading, method dispatch, generics instantiation, tiered compilation, PGO, and the three deployment models (JIT, ReadyToRun, Native AOT).
3. **Make the GC mechanical, not magical.** Allocation contexts, roots, mark/plan/relocate/compact, generations and budgets, write barriers, card tables, regions, LOH, POH, Server vs Workstation, background GC, suspension.
4. **Cover the current platform state honestly.** Regions replaced segments in .NET 7. DATAS is the default for Server GC from .NET 9. .NET 10 significantly expanded escape analysis and stack allocation. .NET 11 (RC1 as of September 2026, GA scheduled 10 November 2026) extends both further and adds runtime async. Interviewers at the architect level increasingly ask what has changed recently.
5. **Teach lifetime properly** — finalization, `IDisposable`, `SafeHandle`, weak references, handles, pinning, and native memory — because the most expensive production memory bugs live here, not in the GC.
6. **Give you an allocation cost model** you can reason with out loud, plus the decision procedure for struct vs class, pooling vs not, and span-based rewriting vs leaving it alone.
7. **Make you diagnostically dangerous.** Four pathologies, a leak taxonomy, and a `dotnet-counters → dotnet-gcdump → dotnet-dump + SOS` workflow you can narrate end to end.
8. **Raise it to architecture.** GC pauses in a latency budget, instance sizing, Native AOT as a deployment decision, GC configuration per workload archetype, and memory as a cost lever.

Eight framings to carry through:

1. **You pay for survivors, not for garbage.** A gen0 collection's cost is roughly proportional to what lives, not to what died. Allocating a million short-lived objects can be cheaper than keeping ten thousand alive.
2. **Allocation is cheap; collection is not.** Allocation is usually a pointer bump. The bill arrives later, on a different thread, in a pause.
3. **Storage location follows the *container*, not the type.** A `struct` inside a `class` is on the heap. A captured local is on the heap. A `ref struct` is *forbidden* from being on the heap. "Value type = stack" is a coincidence of the common case.
4. **The heap is not one heap.** SOH (gen0/1/2), LOH, POH, plus loader heaps and native allocations. They have different policies, different costs, and different failure modes.
5. **Memory is a latency problem before it is a capacity problem.** Most teams discover their GC configuration through tail latency, not through OOM.
6. **The runtime adapts to the machine.** Heap count, budgets, and hard limits are derived from cores and cgroup limits. Change the machine and you change the memory profile, without changing a line of code.
7. **Most "memory leaks" in .NET are retention bugs.** Something is still reachable that you believe is dead. The GC is almost never wrong; your object graph is.
8. **Every optimization here has a measurement attached or it does not exist.** This is the module where guessing is most tempting and most punished.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | What "managed" means | GC + type safety + exact liveness information; the runtime must always be able to find every reference |
| 2 | IL → machine code | Assembly = metadata + IL; types load lazily; methods compile on first call |
| 3 | Object layout | Object header + MethodTable pointer + fields; 16 bytes overhead and a 24-byte minimum on x64 |
| 4 | Method dispatch | Direct, vtable slot, or virtual stub dispatch for interfaces; devirtualization removes all three |
| 5 | Generics instantiation | Reference types share one canonical body; each value type gets its own specialized code |
| 6 | Tiered compilation | Tier-0 for startup, tier-1 for throughput, OSR for loops, dynamic PGO for speculation |
| 7 | R2R and Native AOT | The startup/throughput/memory triangle; pick a corner deliberately |
| 8 | Loader heaps and ALCs | Type data is not GC'd; unloading requires a collectible ALC and zero escapes |
| 9 | The stack | Frames, ~1 MB default, fast, uncatchable overflow; `async` moves it to the heap |
| 10 | The managed heap | Reserve vs commit vs working set; SOH, LOH, POH — different policies |
| 11 | Value vs reference types | Copy semantics and identity, not storage location |
| 12 | Where things actually live | The container decides: fields, captures, boxes, and arrays all go to the heap |
| 13 | Struct layout | Padding and alignment are real; field order changes size; measure, don't assume |
| 14 | Boxing | A heap allocation + copy; triggered by `object`, interfaces, `params object[]`, non-generic APIs |
| 15 | Structs behind interfaces | Constrained callvirt avoids boxing only when the type is statically known; generics with constraints are the fix |
| 16 | `readonly struct` and `in` | Non-readonly structs passed by `in` get defensive copies — a silent pessimization |
| 17 | Equality | Default `ValueType.Equals` boxes and may use reflection; implement `IEquatable<T>` |
| 18 | Strings | Immutable, length-prefixed, interned literals; interning is a per-ALC lifetime commitment |
| 19 | Spans | `Span<T>` is a length + reference; `ref struct` rules exist to keep it stack-only |
| 20 | Ref safety | `ref` returns, `ref` fields, and `scoped` are compile-time escape analysis |
| 21 | The GC's premise | Generational hypothesis + tracing reachability; cost scales with survivors |
| 22 | Allocation | Per-thread allocation context; pointer bump fast path; slow path triggers a GC |
| 23 | Roots | Stack slots, registers, statics, handles, finalizer queue — computed from exact stack maps |
| 24 | GC phases | Mark → plan → relocate → compact (or sweep); compaction is what makes bump allocation possible |
| 25 | Generations | Gen0/1/2 with dynamic budgets; promotion; a gen2 is a full heap walk |
| 26 | Write barriers | Old→young references are recorded in the card table so gen0 GCs don't scan gen2 |
| 27 | Regions | .NET 7 replaced fixed segments with 4 MB regions; generations became sets of regions |
| 28 | LOH | ≥85,000 bytes, allocated in gen2, swept not compacted by default → fragmentation |
| 29 | POH | Pinned objects segregated since .NET 5; pinning elsewhere blocks compaction |
| 30 | Workstation vs Server GC | Server = one heap and one GC thread per core; throughput vs footprint and pause shape |
| 31 | Background GC | Concurrent gen2 marking with short pauses; gen0/1 still stop the world |
| 32 | Suspension | Threads must reach safe points; time-to-suspend is a real and measurable latency source |
| 33 | DATAS | Default in .NET 9+ for Server GC: heap count adapts to load, heap size tracks live data |
| 34 | Latency modes | `SustainedLowLatency`, `TryStartNoGCRegion` — narrow tools, easy to misuse |
| 35 | GC configuration | A short list actually matters: gcServer, DATAS, heap hard limit, heap count, conserve memory |
| 36 | Containers | cgroup limits feed the heap hard limit (75% default) and heap count; OOMKilled ≠ OutOfMemoryException |
| 37 | Reading the GC | Allocation rate, survival rate, pause time %, gen2 frequency, fragmentation, committed vs live |
| 38 | Finalization | Non-deterministic, single-threaded, costs an extra GC cycle; resurrection is possible |
| 39 | `IDisposable` | Deterministic release of *non-memory* resources; `Dispose` does not free managed memory |
| 40 | `IAsyncDisposable` | For resources whose release is I/O; `await using`; don't block in `Dispose` |
| 41 | `SafeHandle` | The correct way to own a native handle; solves the premature-collection race |
| 42 | Weak references | `WeakReference<T>`, `ConditionalWeakTable`, `DependentHandle` — caches and attached state |
| 43 | Pinning | `fixed`, `GCHandle.Pinned`, `Memory<T>.Pin()`; pins fragment the heap |
| 44 | Native memory | Invisible to the GC; `NativeMemory`, `Marshal`, and `GC.AddMemoryPressure` |
| 45 | Interop | `GC.KeepAlive`, marshalling copies, and callbacks that outlive their delegates |
| 46 | Allocation cost model | Fast path ~ a few ns; real cost = collection + cache pressure + promotion risk |
| 47 | Struct vs class | Small, immutable, short-lived, value-semantic, and measured → struct; otherwise class |
| 48 | Pooling | `ArrayPool`, `ObjectPool`, `RecyclableMemoryStream`; pooling moves objects to gen2 |
| 49 | Zero-allocation patterns | Spans, `stackalloc` with a fallback, `string.Create`, UTF-8 literals, `SearchValues` |
| 50 | Hidden allocations | Closures, LINQ chains, iterators, `params`, boxed enumerators, string concatenation |
| 51 | Async state machines | Heap-allocated only when the method actually suspends; runtime async changes the shape |
| 52 | Mid-life crisis | Objects that die just after promotion are the worst case; caches cause it |
| 53 | Deabstraction | .NET 9/10/11 escape analysis stack-allocates boxes, small arrays, delegates, enumerators |
| 54 | Native AOT memory | No JIT, no tiering, smaller footprint, fast startup, Workstation GC by default |
| 55 | When not to optimize | If allocation isn't in your top three costs, this work is negative value |
| 56 | Four pathologies | Leak, bloat, churn, fragmentation — different symptoms, different fixes |
| 57 | Leak taxonomy | Statics, events, closures, timers, DI lifetimes, unbounded caches, ALCs, native |
| 58 | Diagnostic workflow | Counters → gcdump → dump + SOS; two snapshots beat one |
| 59 | SOS | `dumpheap -stat`, `gcroot`, `eeheap`, `dumpobj`, `objsize` — symptom to root |
| 60 | Pause diagnosis | Separate "GC too often" from "GC too slow" from "suspension too slow" |
| 61 | Measuring | BenchmarkDotNet `MemoryDiagnoser`, allocation assertions in tests, production counters |
| 62 | Pauses and SLOs | Budget GC pause into p99 explicitly; pause count × pause length |
| 63 | Instance sizing | Heap count follows cores; hard limit follows the container; autoscaling on memory is a trap |
| 64 | Native AOT decision | Serverless, CLI, sidecars, dense hosting — where startup and footprint dominate |
| 65 | GC per workload | API, batch, streaming, desktop, function — each has a defensible default |
| 66 | Memory as cost | Footprint sets density; density sets node count; node count is the bill |
| 67 | The memory review | A checklist you can narrate in an architecture review |
| 68 | When not to care | Most services never need any of Parts E–G; know how to say so |

---

# Part A — The execution model

## Concept 1 — What "managed" actually means

"Managed code" is not a synonym for "garbage collected." It means the runtime has **complete, exact knowledge of your program's references at every point where it might need it.** Three properties follow:

- **Type safety.** You cannot fabricate a reference from an integer, read past the end of an array, or reinterpret a `string` as a `byte[]`. The verifier and the JIT enforce this, and `unsafe`/`Unsafe`/`Span` are the explicit, auditable escape hatches.
- **Exact liveness information.** For every machine instruction where a collection can occur, the JIT emits a **GC info map** describing which stack slots and registers currently hold object references. This is what lets the collector be *precise* rather than *conservative*: it knows exactly what is a reference, so it can safely **move** objects and update the references to them.
- **Cooperative suspension.** Threads running managed code periodically reach points where they agree to stop. The GC cannot run until every managed thread is at such a point (Concept 32).

The middle property is the load-bearing one for everything in this module. Because the CLR's GC is *precise*, it can **compact** — slide live objects together to eliminate holes. Because it can compact, allocation can be a pointer bump instead of a free-list search. Almost every performance property of .NET memory descends from that single design choice. It is also the reason pinning is expensive (Concept 43): a pinned object is one the collector promised not to move, which breaks the invariant that makes everything else fast.

**The interview-grade sentence:** *"Managed means the runtime always knows exactly which bits are references, which is what lets the GC move objects. Compaction is why allocation is a pointer bump, and pinning is expensive precisely because it takes that ability away."*

---

## Concept 2 — Assembly, metadata, IL, and the path to machine code

A .NET assembly is a PE file containing three things: **metadata** (a set of tables describing types, methods, fields, signatures, and references to other assemblies), **IL** (the stack-based intermediate language for method bodies), and optionally **precompiled native code** (ReadyToRun, Concept 7). There is no separate header/implementation split and no name mangling; metadata is the single source of truth, which is why reflection, serializers, DI containers, and mocking frameworks in .NET are as capable as they are — and why trimming and Native AOT are hard (Concept 54).

Execution proceeds lazily, and the laziness is worth internalizing:

1. **Assembly load.** Triggered by the first reference to a type in it, resolved through an `AssemblyLoadContext` (Concept 8).
2. **Type load.** The first use of a type builds its **MethodTable** and associated runtime structures in a loader heap. This includes laying out fields, building the vtable, resolving interface maps, and running the static constructor's prerequisites.
3. **Method compilation.** The first call to a method invokes the JIT through a **prestub**: the method's entry point initially points at a stub that compiles the method, patches the entry point to the real code, and jumps to it. Subsequent calls go straight to compiled code.
4. **Re-compilation.** With tiered compilation (Concept 6), a hot method is compiled *again*, with optimization, on a background thread, and its entry point is re-patched.

Two practical consequences that show up in interviews. First, **startup cost is dominated by type loading and JIT**, not by your code — which is why a service that serves its first request in 900 ms can serve the thousandth in 4 ms, and why cold-start-sensitive deployments reach for R2R or Native AOT. Second, **the first execution of a code path is not representative of anything**; any benchmark without warm-up is measuring the JIT.

---

## Concept 3 — Object layout: header, MethodTable pointer, and what an object costs

On 64-bit CoreCLR, every heap object has this shape:

```
 [ -8 ]  Object header  (8 bytes: sync block index + padding)
 [  0 ]  MethodTable*   (8 bytes)   <-- an object reference points HERE
 [  8 ]  fields...
         padding to an 8-byte boundary
```

- The **MethodTable pointer** is the type identity. It is what `GetType()`, casts, virtual dispatch, and the GC all consult. The MethodTable itself lives in a loader heap and holds the instance size, the GC layout (which field offsets contain references), the vtable, the interface map, and static field locations.
- The **object header** holds a *sync block index*, used lazily for `lock`, `Monitor.Wait`, the default identity hash code, and COM interop. The common case is a "thin lock" stored directly in the header; contention or additional uses inflate it into a real sync block in a side table.

The cost arithmetic every senior .NET developer should have memorised:

| Thing | Size on x64 |
|---|---|
| Object overhead (header + MT pointer) | 16 bytes |
| Minimum object size | 24 bytes (an empty class instance) |
| Array overhead (header + MT + length) | 24 bytes, then the elements |
| `string` | 22 + 2×length bytes, rounded up to 8 |
| Reference (`object`, `string`, any class) | 8 bytes per field/slot |
| Alignment | Objects are 8-byte aligned |

So `new object()` costs 24 bytes; a `List<int>` with 4 items costs the list object (~32 bytes) plus an `int[]` (24 + 4×4 = 40 → 40 bytes); a `Dictionary<string,int>` with 10 entries costs several hundred bytes across three or four objects. A cache of 1,000,000 small objects is not "a few MB" — do the arithmetic including the 16-byte tax per object and the container's own overhead, and it is usually 3–5× what people guess.

On 32-bit the overhead is 8 bytes and the minimum object is 12 bytes, which is one reason 32-bit measurements don't transfer.

**Where this earns points:** when asked "how much memory will this cache use?", a senior candidate does the layout arithmetic aloud — objects, references, container overhead, and the fact that reference fields cost 8 bytes each regardless of what they point at — rather than saying "it depends."

---

## Concept 4 — Method dispatch: direct, virtual, interface

Three mechanisms, with three costs:

- **Direct call.** Static and non-virtual instance methods on a known type are direct `call` instructions and are freely inlinable. Note that C# emits `callvirt` even for non-virtual instance methods on reference types (it gets the null check for free), but the JIT recognises the pattern and emits a direct call.
- **Virtual call.** A load from the object's MethodTable vtable slot, then an indirect call. Cheap, but it blocks inlining unless the JIT can prove the exact type.
- **Interface call.** Interfaces have no fixed vtable slot across implementing types, so CoreCLR uses **Virtual Stub Dispatch (VSD)**: the call site starts at a lookup stub, which resolves the target and rewrites the site to a *dispatch stub* that does a fast "is the MethodTable the one I cached? then jump" check, falling back to a *resolve stub* and a global cache on a miss. Monomorphic interface call sites are therefore nearly as fast as virtual calls; polymorphic ones are meaningfully slower.

**The optimization that matters is devirtualization**: if the JIT can prove the exact concrete type, all three collapse to a direct call, and then inlining becomes possible, and then *everything else* becomes possible — constant folding, escape analysis, stack allocation (Concept 53). The JIT proves it in three ways: sealed types and sealed/`sealed override` methods; exact type known from the allocation site in the same method; and **guarded devirtualization** driven by dynamic PGO (Concept 6), which emits `if (type == the one we saw 97% of the time) { inlined fast path } else { virtual call }`.

The architect-level consequence: `sealed` is not a style preference. Sealing classes and overrides gives the JIT static facts, which is why the BCL seals aggressively. It is a free, zero-risk optimization on types that were never designed for inheritance.

---

## Concept 5 — Generics: shared code for reference types, specialized code for value types

This is one of the highest-signal internals topics because it explains several otherwise-confusing behaviours.

- For **reference type** arguments, the runtime compiles **one** canonical method body (internally `__Canon`) shared by `List<string>`, `List<Customer>`, `List<Stream>`, and every other reference instantiation. All references are the same size and the same shape to the GC, so one body suffices; exact type information is passed alongside when the code needs it (for `typeof(T)`, `new T()`, or a static field of `T`).
- For **value type** arguments, the runtime compiles a **separate specialization per type**: `List<int>` and `List<double>` have different code. This is what makes generics zero-boxing and why `List<int>` stores actual `int`s rather than boxes.

Consequences:

1. **`List<int>` is fast and allocation-free per element; `ArrayList` boxes every element.** This is the classic comparison, and the reason is specialization, not "generics are faster."
2. **Value-type generics cost code size.** Every instantiation is more JIT work, more code, and more instruction-cache pressure. A generic method instantiated over 40 struct types produces 40 bodies. In Native AOT, where everything is compiled ahead of time, this becomes a binary-size and build-time issue, and unbounded generic virtual method instantiation can be un-compilable at all.
3. **A `struct` constrained to an interface avoids boxing** precisely because a specialized body exists in which the constrained call can be resolved statically (Concept 15).
4. **Static fields are per-instantiation.** `Cache<int>.Instance` and `Cache<string>.Instance` are different fields; `Cache<Customer>.Instance` and `Cache<Order>.Instance` are also different despite code sharing. This is a genuinely useful trick (per-type caches with zero lookup cost) and an occasional source of surprise leaks.

---

## Concept 6 — Tiered compilation, OSR, and dynamic PGO

Modern .NET compiles most methods **twice**:

- **Tier-0** — minimal optimization, compiled fast. Its job is startup. It also, when PGO is on, includes lightweight *instrumentation*: counters for call sites and class probes recording which concrete types actually show up.
- **Tier-1** — full optimization, compiled on a background thread after a method has been called enough times (the call-count threshold defaults to 30, with a startup delay so the burst of initial calls doesn't cause a compile storm). The entry point is then patched.

Two mechanisms complete the picture:

- **OSR (on-stack replacement).** A method with a long-running loop would be stuck in tier-0 forever, because it is only entered once. OSR compiles an optimized version and transfers execution into it *mid-loop*, transplanting the live state. This is what allowed quick-JIT-for-loops to become the default.
- **Dynamic PGO.** The tier-0 instrumentation data feeds tier-1: guarded devirtualization of the types actually observed, better inlining decisions, and hot/cold block layout. On by default since .NET 8. This is why idiomatic interface-heavy C# performs far better than its IL suggests it should.

Interview-relevant consequences:

- **Benchmarks must warm up.** BenchmarkDotNet does this for you; hand-rolled `Stopwatch` loops usually do not.
- **The first N requests are slow**, and that is *visible* in p99 immediately after every deployment. If you deploy 50 times a day behind a 99.9% latency SLO, this is a real line item — and the mitigation is R2R, or warm-up requests in the readiness probe, not disabling tiering.
- **Disabling tiered compilation** (`TieredCompilation=false`) buys steady-state consistency at a large startup cost. It is occasionally right for long-lived, latency-critical, never-restarted processes and almost always wrong otherwise.

---

## Concept 7 — ReadyToRun and Native AOT: the startup/throughput/memory triangle

Three deployment models, three trade-offs:

| | JIT (default) | ReadyToRun (R2R) | Native AOT |
|---|---|---|---|
| Code generation | At runtime | Ahead of time, version-resilient, re-JITted when hot | Ahead of time, final |
| Startup | Slowest | Much faster | Fastest (milliseconds) |
| Peak throughput | Best (PGO + full opts) | Best (tier-1 still kicks in) | Slightly lower (no PGO, no re-optimization) |
| Memory footprint | Highest (JIT + tiering data) | High | Lowest |
| Binary size | Smallest | Larger | Larger on disk, no runtime needed |
| Reflection / dynamic | Unrestricted | Unrestricted | Restricted; requires trimming-safe code |
| Plugin loading, `Reflection.Emit` | Yes | Yes | No |

**R2R** is precompiled code embedded in the assembly. It is *version-resilient*, meaning it stays valid when dependencies are serviced, which is achieved by giving up some optimization (indirections through fixups). The runtime uses R2R code as a starting tier and still promotes hot methods to fully optimized tier-1 code. It is nearly free to enable and the default answer for "our container start-up is slow."

**Native AOT** removes the JIT entirely. Startup becomes tens of milliseconds, footprint drops substantially, and the deployment is a single native binary. The cost is a closed world: no runtime code generation, no loading assemblies that weren't compiled in, and reflection only over what the compiler could see and root. Concept 54 and Module 17 develop this; Concept 64 makes it an architectural decision.

---

## Concept 8 — Loader heaps, AssemblyLoadContext, and why unloading is hard

Not everything in a .NET process is on the GC heap. **Type system data** — MethodTables, MethodDescs, vtables, interface maps, generic instantiations, JIT-compiled code, and the *storage for static fields of reference type is rooted from here* — lives in **loader heaps**, unmanaged memory owned by an `AssemblyLoadContext` (ALC). Loader heap memory is never garbage collected. It is freed only when its entire ALC is unloaded.

This produces a category of memory growth that a heap dump does not explain: a process whose GC heap is stable at 200 MB but whose RSS climbs to 3 GB may be leaking *code and types*, not objects. The usual causes:

- **Dynamic assembly generation** — expression trees compiled repeatedly, `Reflection.Emit`, serializers or mappers that emit per-shape assemblies, Regex with `RegexOptions.Compiled` created in a loop, dynamic proxies.
- **Repeated plugin loads** into non-collectible ALCs.
- **Runaway generic instantiation over value types** (Concept 5).

**Collectible ALCs** (`new AssemblyLoadContext(name, isCollectible: true)`) make unloading possible, and they are the correct mechanism for plugin hosts and hot-reload scenarios. In practice they are hard to use correctly because unloading requires that *nothing* escapes: no instance of a type from the ALC reachable from outside, no delegate pointing into it, no thread executing its code, no static field of an outside type holding one of its objects, no event subscription. One escaped reference pins the entire ALC and everything it loaded. Debugging this is a specialist activity (`!dumpheap -mt`, `!gcroot`, `!finalizequeue`), and "we build a plugin system on collectible ALCs" is a claim an interviewer may probe hard.

**The senior signal:** knowing that RSS = GC heap + loader heaps + native allocations + thread stacks + the runtime itself, and therefore that "the heap is only 200 MB but the container is at 2 GB" is a diagnosable statement, not a contradiction (Concept 36).

---

# Part B — Where things actually live

## Concept 9 — The stack: frames, size, and the overflow you cannot catch

Each thread gets a contiguous, pre-reserved block of virtual memory for its stack. A **stack frame** holds the method's parameters, locals that the JIT could not keep in registers, spill slots, saved registers, and the return address. Allocation is a single register adjustment; deallocation is the same in reverse. There is no bookkeeping and no collector involvement, which is why "on the stack" is shorthand for "free."

Facts worth having precise:

- **Default size is about 1 MB** for threads created by .NET on Windows; the main thread's size comes from the executable header, and on Linux from `ulimit`/pthread defaults (commonly 8 MB for the main thread). `new Thread(action, maxStackSize)` lets you set it; the thread pool's is not adjustable per-thread.
- **Stack space is reserved, not committed.** A 1 MB stack does not consume 1 MB of physical memory until touched. A million threads would still be fatal — at 1 MB of *address space and* kernel bookkeeping each — which is one of several reasons for `async` (Module 15).
- **`StackOverflowException` cannot be caught.** Since .NET Core the process is terminated immediately — no `catch`, no `finally`, no graceful shutdown. Deep or unbounded recursion is therefore an availability bug, not an error-handling problem. For genuinely recursive algorithms over untrusted input (parsers, tree walks, serializers), either convert to an explicit stack or guard depth (`RuntimeHelpers.EnsureSufficientExecutionStack`, or a simple depth counter).
- **`stackalloc`** allocates from the current frame. It is bounded by the remaining stack, unverifiable in size at compile time, and must never appear inside a loop. The safe idiom is a size check with an `ArrayPool` fallback (Concept 49).

The most important corrective: **`async` methods move their locals off the stack.** When a method suspends, everything that must survive the `await` is stored in a heap-allocated state machine (Concept 51). "Use async to save stack memory" is true for thread stacks; "async is free" is not.

---

## Concept 10 — The managed heap: reserve, commit, and the three heaps

Three different numbers are all called "memory," and conflating them is the most common cause of confusing memory conversations:

| Term | Meaning | Where you see it |
|---|---|---|
| **Reserved** | Address space claimed, no physical backing | Virtual size; usually large and uninteresting |
| **Committed** | Backed by physical memory or page file | What the GC "owns"; `GCMemoryInfo.TotalCommittedBytes` |
| **Live / heap size** | Bytes in reachable objects | `GC.GetTotalMemory`, gcdump totals |
| **Working set (RSS)** | Physical pages currently resident | What the container limit and the OOM killer measure |

A healthy service typically has live < committed < RSS, with gaps that are explainable: the gap between live and committed is GC headroom and fragmentation; the gap between committed heap and RSS is stacks, loader heaps, native allocations, and the runtime itself.

The GC manages **three logical heaps**, with different policies:

- **SOH (Small Object Heap)** — generations 0, 1, and 2. Compacting, bump-allocated.
- **LOH (Large Object Heap)** — objects ≥ **85,000 bytes**. Collected only with gen2, swept rather than compacted by default (Concept 28).
- **POH (Pinned Object Heap)** — introduced in .NET 5, an explicit home for objects you intend to pin, so that pinning doesn't fragment the SOH (Concept 29).

Since .NET 7, all of these are carved out of **regions** rather than fixed segments (Concept 27).

---

## Concept 11 — Value types vs reference types: the actual definition

The distinction is **not** "stack vs heap." It is three things:

1. **Copy semantics.** Assigning a value type copies the bits; assigning a reference type copies an 8-byte reference. This is the whole story and everything else follows from it.
2. **Identity.** Reference types have identity — two references can point to the same object, and mutating through one is visible through the other. Value types have no identity; equality is about content.
3. **Default value and nullability.** A value type always has a zero-initialized default. A reference type's default is `null`.

Everything else people cite is consequence or coincidence:

- "Structs are faster" — sometimes. Copying a 96-byte struct on every method call is slower than passing an 8-byte reference. The break-even is usually around 16–24 bytes, and *always* subject to measurement.
- "Structs don't allocate" — they don't allocate *separately*. A struct in a field allocates as part of its container; a boxed struct allocates a full object.
- "Value types go on the stack" — see Concept 12.

The useful framing for interviews: **a value type is a *value*; a reference type is a *thing*.** A `Money`, a `Point`, a `DateTime`, a `TimestampedReading` are values — two of them with the same content are interchangeable. A `Customer`, an `HttpClient`, an `Order` aggregate are things — identity matters, and you want exactly one of them.

---

## Concept 12 — Where things actually live: the container decides

Storage location is determined by **what contains the variable**, not by whether its type is a value type:

| Declaration | Where the data lives |
|---|---|
| Local `int x` in a non-async, non-capturing method | Stack (or a register) |
| `int` field of a `class` | Heap, inline in the object |
| `int` element of `int[]` | Heap, inline in the array |
| `struct` local captured by a lambda | Heap, in the closure object |
| `struct` local in an `async` method that lives across an `await` | Heap, in the state machine |
| `struct` boxed to `object` or an interface | Heap, as a boxed object |
| `struct` field of a `struct` field of a `class` | Heap, flattened inline |
| `class` local | Reference on the stack, object on the heap |
| Reference type static field | Object on the heap, rooted from the loader heap |
| `ref struct` (e.g. `Span<T>`) | Stack only — the compiler forbids everything else |

Two corollaries with real consequences.

**A struct inside a class is on the heap and is subject to all heap costs.** Making `Order.Id` a struct does not reduce GC pressure if `Order` is a class — the bytes were going to be there anyway. The struct saves you an *object* only when it replaces a separate object.

**Fields of a class that are reference types cost you GC work.** The GC must scan every reference field of every surviving object. A class with 30 reference fields is 30 pointers to trace on every collection that touches it. A class whose fields are all value types is *cheap to trace* — the GC layout says "no references here, skip." This matters for large, long-lived arrays: `MyStruct[100_000]` where `MyStruct` has no reference fields is a single object with no references to trace; `MyClass[100_000]` is 100,001 objects and 100,000 references traced on every gen2.

---

## Concept 13 — Struct layout, padding, and alignment

The CLR lays out struct fields with alignment requirements: an `int` on a 4-byte boundary, a `long`/reference on an 8-byte boundary. The runtime may reorder fields (`LayoutKind.Auto`, the default for structs in C#) to minimise padding, but you should not rely on it, and `LayoutKind.Sequential` (required for interop) preserves your declaration order — including the holes.

```csharp
[StructLayout(LayoutKind.Sequential)]
struct Bad  { byte a; long b; byte c; }   // 1 + 7 pad + 8 + 1 + 7 pad = 24 bytes
[StructLayout(LayoutKind.Sequential)]
struct Good { long b; byte a; byte c; }   // 8 + 1 + 1 + 6 pad       = 16 bytes
```

Three rules:

1. **Order fields largest-to-smallest** in interop or explicitly-laid-out structs. It costs nothing and can cut size by a third.
2. **A struct containing any reference field** must be traced by the GC and cannot be stack-allocated as freely, cannot be used with `Unsafe.SizeOf` assumptions in some interop paths, and prevents some optimizations. "Does this struct contain a reference?" is a question with real consequences.
3. **Measure, don't assume.** `Unsafe.SizeOf<T>()` gives the managed size; `Marshal.SizeOf<T>()` gives the marshalled (interop) size, and they legitimately differ (notably for `bool` and `char`). Interview trivia, but the kind that separates people who have done interop from people who haven't.

Alignment also explains cache behaviour: a hot 64-byte struct that straddles two cache lines costs twice the memory traffic. For contended counters, `[StructLayout(LayoutKind.Explicit, Size = 128)]` padding to avoid false sharing is a known technique (Module 15).

---

## Concept 14 — Boxing: mechanics and the triggers you don't see

**Boxing** allocates a heap object with the value type's MethodTable, copies the value into it, and returns a reference. **Unboxing** checks the type and copies back. The cost is an allocation (24+ bytes for anything small), a copy, and eventual collection — per box.

The obvious trigger is `object o = 42;`. The non-obvious ones are where the points are:

- **Assigning a struct to an interface**: `IComparable c = 42;` boxes. Calling an interface method on a struct through an interface-typed variable boxes.
- **Non-generic collections and APIs**: `ArrayList`, `Hashtable`, `DataRow` indexers, older interop signatures.
- **`params object[]`**: `string.Format("{0}", 42)` boxes the `42`. Interpolated strings in modern .NET use `DefaultInterpolatedStringHandler` and avoid this for common types — a real, measurable .NET 6+ improvement.
- **`Enum` operations**: `enum.HasFlag` boxed historically (fixed in .NET Core), `Enum.ToString()`, and using an enum as a dictionary key with the default comparer boxed on older runtimes.
- **`ValueType.Equals`/`GetHashCode` without overrides** (Concept 17).
- **Structs in `IEnumerable<T>` positions**: `foreach` over a `List<T>` uses the struct enumerator directly; `foreach` over the same list *typed as `IEnumerable<T>`* boxes the enumerator. This one is extremely common in real code and is exactly what .NET 9/10/11 escape analysis has been chasing (Concept 53).
- **Lambdas capturing structs** and `Nullable<T>` in some patterns.
- **`async` methods returning `Task<T>` where `T` is a struct** allocate the `Task` object regardless; `ValueTask<T>` exists for that case.

**How to find them:** the IL keyword is `box`. Practical tools: BenchmarkDotNet's `[MemoryDiagnoser]` allocation column, the `Microsoft.CodeAnalysis.PerformanceSensitiveAnalyzers` / `ClrHeapAllocationAnalyzer` Roslyn analyzers, sharplab.io to read the IL, and `dotnet-counters` allocation rate in aggregate.

---

## Concept 15 — Structs behind interfaces, constrained callvirt, and generic constraints

C# compiles a method call on a struct through a constraint into `constrained. callvirt`. When the JIT knows the exact struct type, it resolves the call directly with **no boxing**. When it doesn't — because the variable is typed as the interface — it must box.

```csharp
interface IShape { double Area(); }
struct Circle : IShape { public double R; public double Area() => Math.PI*R*R; }

double A(IShape s) => s.Area();          // boxes at every call site that passes a Circle
double B<T>(T s) where T : IShape => s.Area();   // no boxing: specialized for Circle
double C<T>(in T s) where T : IShape, allows ref struct => s.Area();  // and no copy
```

This is the single most useful applied consequence of Concept 5, and it is the mechanism behind the BCL's generic-with-constraint patterns (`IEqualityComparer<T>` passed as a struct type parameter, `SearchValues<T>`, the generic math interfaces, `Vector<T>` helpers).

The follow-up an interviewer may ask: *"so should all my interfaces be generic constraints?"* No. The pattern costs code size (one body per struct), loses the ability to store heterogeneous implementations in a collection, and gains you nothing if the implementations are classes. It belongs in hot paths where the struct type is statically known — the exact place the BCL uses it.

---

## Concept 16 — `readonly struct`, `in` parameters, and defensive copies

A non-trivially-sized struct passed by value is copied at every call. `in` (and `ref readonly`) pass by reference without allowing mutation — an apparent free win. There is a trap:

If the struct is **not** declared `readonly`, the compiler cannot prove that calling a member on it won't mutate it. Since `in` promises the caller's copy won't change, the compiler emits a **defensive copy** before every member access. You asked for pass-by-reference and got pass-by-reference *plus a copy per member access* — strictly worse than passing by value.

```csharp
struct Big { public decimal A, B, C, D; public decimal Sum() => A+B+C+D; }
readonly struct BigR { public readonly decimal A, B, C, D; public decimal Sum() => A+B+C+D; }

static decimal Slow(in Big b)  => b.Sum() + b.Sum();   // two defensive copies
static decimal Fast(in BigR b) => b.Sum() + b.Sum();   // none
```

Rules that follow, and they are cheap to apply:

- **Declare every struct `readonly` unless it genuinely needs mutation.** It is a correctness improvement (value types with mutable state are a classic bug source — mutating a struct in a `List<T>` mutates a copy) *and* an optimization enabler.
- If a struct must be mutable, mark individual non-mutating members `readonly` (`public readonly decimal Sum() => ...`), which restores the optimization member by member.
- Only reach for `in` on structs above roughly 16–24 bytes, and only after measuring. Below that, by-value in registers is faster.
- `ref readonly` parameters (C# 12) are the explicit, caller-visible version of `in`; the analyzer-friendly choice for APIs where the reference semantics matter.

---

## Concept 17 — Equality, hash codes, and the boxing trap in `ValueType.Equals`

If you do not override them, `ValueType.Equals(object)` and `GetHashCode()` have to work for arbitrary structs. The runtime has a fast path for "blittable" structs with no reference fields and no padding, comparing bits directly. For everything else it falls back to **reflection over the fields** — and the `object` parameter means the argument is **boxed**. A struct used as a dictionary key without overrides can therefore be dramatically slower than the equivalent class, which is a counter-intuitive result that makes a good interview question.

The correct pattern for any struct used in collections or comparisons:

```csharp
readonly struct Money : IEquatable<Money>
{
    public readonly decimal Amount;
    public readonly Currency Currency;
    public bool Equals(Money other) => Amount == other.Amount && Currency == other.Currency;
    public override bool Equals(object? o) => o is Money m && Equals(m);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public static bool operator ==(Money a, Money b) => a.Equals(b);
    public static bool operator !=(Money a, Money b) => !a.Equals(b);
}
```

`record struct` (and `readonly record struct`) generates all of this correctly, including a strongly-typed `IEquatable<T>.Equals`. For value-semantic data types, prefer it — it is the modern answer and it signals fluency with C# 9+ rather than a memorized 2008 pattern. The one thing to keep in mind: a `record struct` is mutable unless declared `readonly record struct`.

`Dictionary<TKey,TValue>` has an additional subtlety worth knowing: it special-cases `EqualityComparer<T>.Default` for types implementing `IEquatable<T>` so the comparison devirtualizes; providing a *class* comparer instead reintroduces an interface call per lookup.

---

## Concept 18 — Strings: layout, immutability, interning, and the literal lifetime

A `string` is a heap object with a length field followed by UTF-16 characters and a null terminator: **22 + 2n bytes**, rounded to 8. A 40-character string is ~104 bytes. Strings are usually the largest single category on a real service's heap; `dumpheap -stat` almost always shows `System.String` at the top, and the second question is always *"which ones, and who holds them?"*

**Immutability** buys thread safety, safe sharing, and safe use as dictionary keys — and costs a new allocation for every transformation. Hence:

- **Concatenation in a loop is O(n²) in allocation.** Use `StringBuilder`, or better, `string.Create`, or better still, write UTF-8 bytes directly and never materialize the string.
- **Interpolated strings** in .NET 6+ compile to `DefaultInterpolatedStringHandler`, which uses a pooled buffer and avoids boxing. `$"..."` is no longer the naive `string.Format` it once was — but it still produces a string.
- **Logging is the biggest hidden string cost in most services.** `logger.LogDebug($"Processing {order.Id}")` formats the string *even when Debug is disabled*. Use the structured overload with a message template and arguments (`LogDebug("Processing {OrderId}", order.Id)`) or, better, the `LoggerMessage` source generator, which avoids the boxing and the allocation entirely when the level is off.

**Interning.** All string literals in an assembly are interned in a per-ALC intern pool, so identical literals are the same object. `string.Intern(s)` adds a runtime string to that pool. The trap: **interned strings live as long as the ALC**, effectively forever. Interning user input, tenant IDs, or parsed tokens is an unbounded leak with a friendly name. If you want deduplication with a bounded lifetime, use a `ConcurrentDictionary<string,string>` you control, or `string.Intern`'s modern alternative — an `AlternateLookup`-based pool over spans — so you can evict.

---

## Concept 19 — `Span<T>`, `Memory<T>`, and the ref-struct rules

`Span<T>` is a `ref struct` holding a **managed reference plus a length**. It can point into a managed array, a `stackalloc` buffer, native memory, or the interior of an object, and it gives you array-like indexing with bounds checks the JIT can usually elide. Slicing is free — a new span pointing further along, no copy, no allocation.

The rules exist to keep the reference valid:

- A `ref struct` **cannot be boxed**, cannot be a field of a class or a non-ref struct, cannot be used as a generic type argument (until the `allows ref struct` anti-constraint in C# 13), cannot be captured in a lambda, cannot be used in `async` methods across an `await`, and cannot be used in iterators.
- The reason is uniform: all of those operations would put the reference on the heap, where it could outlive the stack frame or object it points into.

`Memory<T>` is the heap-safe counterpart — a normal struct that can be stored in fields and used across `await`. You obtain a `Span<T>` from it (`.Span`) at the point of use. The ownership rules are important and frequently misunderstood: **a `Memory<T>` handed to you is only valid while its owner says it is.** Passing a `Memory<T>` that wraps a pooled array to code that stores it for later is a use-after-free with a managed accent. The `Memory<T>` usage guidelines (linked in the resources) define the ownership/consumption/lifetime contract; if your API takes a `Memory<T>`, document whether you retain it past the call.

For an architect, the important part is *where these belong*: parsers, serializers, protocol handlers, and hot I/O paths. Spreading `Span<T>` through domain code buys nothing and costs readability and testability.

---

## Concept 20 — Ref returns, ref fields, and `scoped`: escape analysis at compile time

C# has a second, entirely compile-time memory-safety system layered on the GC: **ref safety**. `ref` returns, `ref` locals, and (since C# 11) `ref` fields inside `ref struct`s let you work with references to memory the GC does not own, while the compiler proves nothing outlives its storage.

```csharp
ref int First(int[] a) => ref a[0];          // fine: the array is on the heap
ref int Bad() { int x = 0; return ref x; }   // error: would outlive the frame
Span<T> AlsoBad() { Span<byte> s = stackalloc byte[64]; return s; } // error
```

The vocabulary an interviewer might use:

- **`scoped`** — an explicit promise that a `ref` or `ref struct` parameter does not escape the method, which *enables* the caller to pass stack-allocated data.
- **`ref` fields** — `Span<T>` is itself implemented with a `ref T` field since .NET 7; before that it required runtime magic (`ByReference<T>`).
- **`ref readonly` / `in`** — reference without mutation (Concept 16).
- **`UnscopedRef`** — the escape hatch that says "this reference may outlive the call", used on `ref` returns from struct members.

This is a compile-time analogue of the JIT's runtime escape analysis (Concept 53), and naming that parallel is a nice signal: *"C# does escape analysis at compile time for ref safety; the JIT does it at compile time for stack allocation; they're solving the same question — does this reference outlive the frame? — for different reasons."*

---

# Part C — The garbage collector

## Concept 21 — What the GC is for, and the two assumptions it is built on

The GC's job is to reclaim memory that is no longer **reachable**. Note the word: not "no longer used," not "no longer needed" — *no longer reachable from a root*. That distinction is the source of essentially every .NET memory leak (Concept 57).

Two design assumptions shape everything:

1. **The generational hypothesis: most objects die young.** Empirically, the overwhelming majority of allocations in typical programs become garbage almost immediately — request-scoped DTOs, intermediate strings, LINQ closures. If you can collect *just* those cheaply and rarely touch the old survivors, you get most of the benefit for a fraction of the work.
2. **Tracing, not counting.** The GC does not track references as they are created (no reference counting). It periodically walks the object graph from roots and marks what it finds. This handles cycles for free and costs nothing at assignment time except a write barrier (Concept 26).

Combine them and you get the most important cost statement in this module:

> **The cost of a collection is proportional to the number of *surviving* objects, not to the amount of garbage.**

Dead objects cost nothing to collect — they are simply not marked, and the space they occupied is reused. This is why "allocate a million short-lived objects" can be cheap and "keep a million objects alive in a cache" is expensive forever. It also explains the "mid-life crisis" pathology (Concept 52) and why caching is a *GC decision*, not just a hit-rate decision.

---

## Concept 22 — Allocation: the allocation context and the pointer bump

Each thread has its own **allocation context** — a small chunk of gen0 carved out for it. Allocation on the fast path is:

```
ptr = alloc_ptr
if (ptr + size > alloc_limit) goto slow_path
alloc_ptr = ptr + size
*(ptr) = MethodTable*
return ptr
```

Three or four instructions, no lock, no free list, no search. This is why allocation in .NET is often *faster* than `malloc`: there is no fragmentation to navigate because the compacting collector guarantees a contiguous free region.

The slow path (`JIT_New` and friends) asks the GC for a new allocation context. If gen0's budget is exhausted, this **triggers a collection**. Therefore:

- **Allocation is where GCs happen.** A thread that never allocates never triggers a GC, though it can still be suspended by one.
- **The allocation rate sets the GC frequency**, and gen0 budget sets the amount allocated between collections. "Allocations/sec" from `dotnet-counters` divided by the gen0 budget approximates your gen0 GC rate.
- **Large objects bypass this path**: allocations ≥ 85,000 bytes go straight to the LOH with different accounting (Concept 28).
- **Zeroing costs real time.** New memory is zeroed before you see it. For big arrays this is measurable, which is why `GC.AllocateUninitializedArray<T>(n)` exists — use it only when you will immediately overwrite the whole buffer.

---

## Concept 23 — Roots and reachability

A **root** is a reference the GC treats as definitionally live. The set:

- **Stack slots and registers** of every managed thread, identified exactly from the JIT's GC info for the current instruction pointer. This is the precision that enables compaction (Concept 1).
- **Static fields** — reference-type statics are rooted from handles associated with their loader heap / ALC.
- **GC handles** — `GCHandle` of type Normal or Pinned, plus internal handles used by interop and the runtime (Concept 43).
- **The finalization queue** — an object with a pending finalizer is kept alive by that queue (Concept 38).
- Thread-locals, and the arguments of in-flight interop calls.

Anything transitively reachable from a root survives. Anything else is garbage, cycles included.

A subtlety that catches people in interviews: **a local variable does not keep an object alive until the closing brace.** In optimized code, the JIT's liveness analysis ends the object's life at its last *use*. An object can be collected while one of its methods is still executing, if that method no longer touches `this`. This is exactly the bug `GC.KeepAlive` exists for (Concept 45), and the reason `SafeHandle` is better than a raw `IntPtr` field (Concept 41). In Debug builds this never reproduces, because locals are kept alive to the end of scope for the debugger — one of the classic "only fails in Release, only in production" bugs.

---

## Concept 24 — The phases: mark, plan, relocate, compact, sweep

A collection proceeds roughly as:

1. **Suspend** the managed threads (Concept 32).
2. **Mark.** Walk from the roots, marking reachable objects. For a gen0/gen1 collection, the walk is limited: objects in older generations are *not* traced unless the card table says they might point into the collected generation (Concept 26). This is what makes ephemeral collections cheap.
3. **Plan.** Decide whether to compact or sweep this time, based on fragmentation and budget heuristics, and compute new addresses.
4. **Relocate.** Update every reference — from roots, from surviving objects, from handles — to the new addresses.
5. **Compact** (slide survivors together, leaving one contiguous free area) **or sweep** (leave objects in place, thread the holes onto a free list).
6. **Resume** threads.

Two facts to keep straight, because they are commonly muddled:

- **The SOH compacts; the LOH sweeps by default.** Compaction makes bump allocation possible and eliminates fragmentation; it costs the memory copy and the reference fixups, so the GC does it selectively based on how fragmented things are.
- **Compaction requires movable objects.** Pinned objects (Concept 43) cannot move, so the collector must plan around them, often leaving holes. A handful of long-lived pins scattered through gen2 can defeat compaction entirely.

---

## Concept 25 — Generations, budgets, and promotion

Three generations on the SOH:

- **Gen0** — the nursery. Every small object is born here (except the special cases in Concept 28/29). Collected frequently and cheaply.
- **Gen1** — a buffer. Objects that survived one gen0 collection. The buffer exists so that objects that are "about to die but hadn't yet when gen0 ran" get a second chance to die cheaply before entering gen2.
- **Gen2** — everything older, plus the LOH and POH. Collecting gen2 means walking the whole heap. This is the expensive one.

Key mechanics:

- **Each generation has a budget** (how much can be allocated into it before a collection of that generation is triggered). These budgets are **dynamically tuned** by the GC based on survival rates, and differ enormously between Workstation, Server, and DATAS modes. Do not memorize numbers; memorize that they adapt.
- **A collection of generation N collects generations 0..N.** "Gen2 GC" means "full GC." There is no way to collect gen2 alone.
- **Survivors are promoted.** Survive gen0 → gen1. Survive gen1 → gen2. Once in gen2, an object is only reclaimed by a full collection.
- **Gen2 GCs are what you feel.** Their frequency and duration are the two numbers to watch (Concept 37).

The design intent is that gen0 collections are so cheap they are essentially free, and gen2 collections are rare. When a service is in trouble, one of those two statements has stopped being true, and the diagnosis in Concept 60 is precisely about determining which.

---

## Concept 26 — Write barriers and the card table

Problem: if a gen0 collection doesn't trace gen2, how does it know about a gen2 object holding the only reference to a gen0 object?

Solution: every reference-field write in managed code goes through a **write barrier** — a small piece of JIT-emitted code that records the fact that a region of the heap was modified. The record lives in the **card table**, a compact side structure where each entry covers a small span of heap (on the order of 256 bytes). During an ephemeral collection, the GC scans only the *dirty* cards in older generations, treating references found there as additional roots.

Consequences worth knowing:

- **Reference field writes are not free.** Assigning a reference costs the store plus the barrier. Assigning a value type field does not. This is a small but real argument for reference-free structs in hot loops and huge arrays.
- **`Array.Copy` over reference arrays** is meaningfully more expensive than over primitives, for the same reason plus covariance checks.
- **Writing to many scattered old objects dirties many cards**, making subsequent gen0 collections more expensive. A long-lived object graph that is constantly mutated to point at new short-lived objects is a known anti-pattern — it converts "cheap gen0" into "gen0 that scans a lot of gen2."
- .NET 10 improved the Arm64 write barrier implementation to handle GC regions more precisely, trading a little write throughput for **measured GC pause improvements of roughly 8% to over 20%** — a good concrete example to cite if asked what has changed recently.

There is also a **brick table** used to locate object starts within a card quickly; you rarely need it by name, but knowing it exists signals you have read the runtime's design docs.

---

## Concept 27 — Segments → regions (.NET 7 onwards)

Historically the GC reserved memory in large fixed-size **segments**, with an "ephemeral segment" holding gen0 and gen1 and additional segments for gen2 and LOH. Generations were *address ranges* within segments, which made it hard to give memory back to the OS and hard for a generation to shrink.

.NET 7 replaced this with **regions**: the heap is divided into small uniform units (4 MB), and each region is assigned to a generation. A generation is now a *set of regions* rather than an address range.

Why this matters, in terms an architect cares about:

- **Memory can be returned to the OS at region granularity**, so committed memory tracks live data far more closely. This is the enabling work for DATAS (Concept 33).
- **Generations can grow and shrink independently** without the "ephemeral segment is full but gen2 has space" awkwardness.
- **Fragmentation behaviour changed**, which is why memory-related regressions and improvements clustered around the .NET 7 upgrade, and why comparing memory numbers across .NET 6 → 7+ requires care.
- Regions are the default on 64-bit; segments remain as a fallback configuration (`GCName`/`DOTNET_GCgen0size`-era tuning advice from before .NET 7 often no longer applies, which is a good reason to distrust old blog posts).

---

## Concept 28 — The Large Object Heap

Objects of **85,000 bytes or more** are allocated on the LOH. That is 85,000 bytes exactly, not 85 KB — so a `byte[85000]` is on the LOH but a `byte[84000]` is not, and the array header counts. In practice: arrays of about 10,600 `long`s, 21,250 `int`s, or strings of about 42,500 characters.

Properties:

- **LOH objects are gen2 from birth.** They are only collected during a full (gen2) collection. Allocating large arrays frequently therefore *forces gen2 GCs*, which is the expensive kind.
- **The LOH is swept, not compacted, by default.** Freed space becomes a free list. Because objects are large and sizes vary, this fragments: you can have 500 MB of free LOH space and still fail to allocate a 40 MB array.
- **You can force compaction**: `GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;` before an induced `GC.Collect()`. It is a *very* expensive stop-the-world operation. Legitimate uses exist (after a batch phase transition, during a maintenance window); putting it on a timer is not one.
- **The threshold is configurable** (`System.GC.LOHThreshold` / `DOTNET_GCLOHThreshold`). Raising it moves medium-large arrays back to the SOH where they get compacted; this is occasionally the right fix for a service that allocates lots of 90 KB buffers, but it increases gen0/1 copying costs. Measure both ways.

**The real-world pattern:** `MemoryStream` growth doubles its internal array, and a 2 MB response body produces a chain of LOH allocations. This is exactly what `Microsoft.IO.RecyclableMemoryStream` was built to fix (Concept 48). Large `byte[]` buffers for I/O, large `List<T>` growth, and `string.Join` over big collections are the other three usual suspects.

---

## Concept 29 — The Pinned Object Heap and the cost of pinning

**Pinning** tells the GC "this object must not move." It is required whenever a native pointer to managed memory escapes to code the runtime doesn't control: `fixed` blocks, `GCHandle.Alloc(obj, GCHandleType.Pinned)`, most async socket and file I/O, and interop marshalling of arrays.

Pinning conflicts directly with compaction. A pinned object in the middle of gen0 forces the collector to work around it, leaving a hole and potentially promoting neighbouring objects it would rather have left alone. Long-lived pins in gen2 prevent the heap from being compacted at all in those regions. The classic symptom is a heap where `!dumpheap -stat` shows modest live bytes but committed memory is several times larger, and `!gchandles` shows many pinned handles.

.NET 5 introduced the **Pinned Object Heap (POH)** to fix the structural version of this problem: allocate objects you *intend* to pin directly into a heap where nothing moves, so they never fragment the SOH.

```csharp
byte[] buffer = GC.AllocateArray<byte>(length: 8192, pinned: true);   // lives on the POH
```

Guidance:

- **Long-lived pinned buffers** (I/O buffers, interop scratch space, ring buffers for a native library) belong on the POH.
- **Short-lived pins** (`fixed` over a small array inside a method) are fine as-is; the window is short.
- **Never pin inside a loop over many small objects.** That is the pathological case.
- Pooled I/O buffers plus POH allocation is the standard high-performance pattern: allocate once, pin permanently, reuse forever.

---

## Concept 30 — Workstation vs Server GC

This is the highest-yield GC interview question after "explain generations," and most candidates give half the answer.

| | Workstation GC | Server GC |
|---|---|---|
| Heaps | One | One **per logical core** the process may use |
| GC threads | Collection happens on the allocating thread | A dedicated, high-priority GC thread per heap |
| Goal | Low latency on a shared machine; small footprint | Maximum throughput; process assumed to own the machine |
| Ephemeral pauses | Shorter individually, more frequent | Longer per GC but heavily parallel; larger budgets → far fewer GCs |
| Memory footprint | Lower | Higher (n heaps, larger budgets) — unless DATAS is on |
| Default | Console/desktop apps | ASP.NET Core apps (the web SDK sets it) |
| Setting | `<ServerGarbageCollection>false</ServerGarbageCollection>` | `true`, or `DOTNET_gcServer=1` |

The mechanism people miss: **Server GC allocates one heap per core, and each heap has its own allocation contexts and its own budget.** That is why a Server GC process's memory grows with core count even though the workload is identical — and why the same container image used 400 MB on a 4-core node and 2.5 GB on a 32-core node before DATAS. It is also why Server GC without a CPU limit inside a container that *has* a CPU limit used to be a common misconfiguration (the runtime now reads cgroup CPU limits, but check it when you see this pattern).

The decision, stated the way a senior candidate would:

> *"Server GC for throughput-oriented server workloads that have multiple cores and can afford the footprint — which is the ASP.NET Core default and usually correct. Workstation GC for memory-constrained containers, for processes running many-to-a-host, for desktop apps, and for small sidecars. Since .NET 9, DATAS is on by default with Server GC, which removes most of the old footprint argument; if I'm on .NET 8 or earlier in a small container, I'd check Workstation GC explicitly."*

---

## Concept 31 — Background GC: what is actually concurrent

**Background GC (BGC)** applies to **gen2 only**, and it is on by default in both Workstation and Server modes.

What happens: instead of one long stop-the-world full collection, the GC does the bulk of gen2 marking *concurrently* with your threads running. It takes two short pauses — one at the start to set up, one near the end to finish — and during the concurrent phase, allocation continues and gen0/gen1 collections can still occur (these are called *ephemeral GCs during BGC*, and they do stop the world briefly).

What this buys and costs:

- **Buys:** gen2 pauses go from potentially hundreds of milliseconds to two short pauses, at the cost of some throughput and a bit more memory (the heap grows during the concurrent phase).
- **Costs:** BGC is *not* compacting for the concurrent part. A truly compacting gen2 (a "blocking gen2") still pauses fully. So your worst-case pause is not the BGC pause; it is the blocking gen2 the GC decides it needs, or an induced `GC.Collect()`.
- **Gen0 and gen1 are always fully blocking.** They are just short. This is why "we use background GC so we don't have pauses" is wrong and a good thing to correct in an interview.

`GCSettings.LatencyMode` interacts with this (Concept 34), and `System.GC.Concurrent = false` disables it (occasionally correct for batch workloads that want maximum throughput and don't care about pauses).

---

## Concept 32 — Suspension: safe points, hijacking, and time-to-suspend

Before a GC can start, **every managed thread must reach a safe point** — a place where the GC info accurately describes which registers and stack slots hold references. Two mechanisms get them there:

- **GC polls.** The JIT inserts checks at method calls and loop back-edges in some code shapes; a thread notices the GC is pending and parks itself.
- **Thread hijacking.** For threads in fully-optimized code without polls, the runtime rewrites the return address on the thread's stack so that when the current function returns, it lands in runtime code that parks it.

Threads in native code (P/Invoke, blocking syscalls) are in "preemptive mode" and don't need to be stopped — the GC just requires that they don't return into managed code mid-collection.

This is a genuine latency source with a name: **time-to-suspend**. Symptoms of it going wrong:

- A **long-running tight loop with no calls and no allocations** may have no poll points, delaying suspension until the loop exits. This is rarer with modern JIT loop handling but still appears in numeric kernels.
- **Thread pool starvation or oversubscription** means many threads to stop, each needing a context switch.
- **A thread blocked in a long native call that returns** at an awkward moment.

You can see this directly: GC ETW/EventPipe events expose suspension duration separately from GC duration. If pause time is high but "GC duration" is low, suspend time is the story — and the fix is in your code's shape or your thread count, not in GC configuration. This distinction is a strong senior signal in a diagnosis question.

---

## Concept 33 — DATAS: Dynamic Adaptation To Application Sizes

**DATAS** was introduced as opt-in in .NET 8 and is **enabled by default from .NET 9** for Server GC. It is the most important recent change in .NET memory behaviour and a very likely "what's new" interview question.

The problem it solves: classic Server GC does not adapt to the application. Its heap count equals the core count, and its budgets aim at throughput, so heap size varies wildly with the machine rather than with the workload, and it does not shrink aggressively when load drops. In containers and in autoscaled fleets this is precisely backwards.

What DATAS does:

- **Starts with one heap** and adds heaps only when the workload demonstrates it needs them, using throughput cost percentage as the signal (default target ~2%, tunable).
- **Sets the allocation budget from the long-lived data size**, so the heap is roughly proportional to *the live set* rather than to available RAM.
- **Shrinks back down** when load falls, returning memory to the OS (which regions made possible, Concept 27).

The practical result is that the same app on a 12-core and a 28-core machine has similar heap sizes, and that memory usage after a traffic spike returns to baseline instead of staying inflated.

When to turn it off (`System.GC.DynamicAdaptationMode = 0` / `DOTNET_GCDynamicAdaptationMode=0`):

- Sustained high-throughput workloads where you measured a throughput regression and have the memory to spare.
- Workloads with very spiky arrival patterns where the ramp-up in heap count costs you at the start of each burst — DATAS needs a few GCs to add heaps, so allocating threads can wait during that transition.

Both of those are *measured* conclusions. "We disabled DATAS" without a benchmark is a red flag; so is "we upgraded to .NET 9 and memory behaviour changed" without knowing DATAS is why.

---

## Concept 34 — Latency modes and `TryStartNoGCRegion`

`GCSettings.LatencyMode` lets you ask the GC to bias its choices:

| Mode | Meaning |
|---|---|
| `Batch` | Throughput above all; disables concurrent GC |
| `Interactive` | The default for concurrent configurations |
| `SustainedLowLatency` | Avoid blocking gen2 collections where possible; the heap may grow |
| `LowLatency` | Workstation-only, short-term, avoids gen2 almost entirely |
| `NoGCRegion` | Set by `TryStartNoGCRegion`, not assignable directly |

`GC.TryStartNoGCRegion(totalSize)` asks the GC to pre-commit enough memory that no collection will occur until you call `EndNoGCRegion()` or exceed the budget. It is the right tool for a genuinely bounded critical section — a market-open burst, a frame in a game loop, a latency-critical trading window — and it is a foot-gun everywhere else: exceed the budget and you get a GC anyway, forget to end it and you accumulate garbage indefinitely, and the call itself may trigger a blocking collection to free up the space it needs.

Related APIs, with honest guidance:

- **`GC.Collect()`** — almost always wrong in server code. Legitimate uses: after a distinct phase change in a batch process, before taking a memory measurement in a diagnostic, and in tests. It forces a blocking gen2 and defeats the GC's tuning.
- **`GC.AddMemoryPressure` / `RemoveMemoryPressure`** — tells the GC about native memory owned by managed objects so it collects more eagerly (Concept 44). Legitimately useful and underused.
- **`GCConserveMemory` (0–9)** — trades throughput for a smaller footprint, including more aggressive compaction. Worth trying in memory-constrained containers before more invasive changes.

---

## Concept 35 — The GC configuration surface that actually matters

There are dozens of knobs. These are the ones that come up in real work, set in `runtimeconfig.json` (or as `DOTNET_`-prefixed environment variables, which is what you usually use in containers):

| Setting | Env var | What it does | When you'd touch it |
|---|---|---|---|
| `System.GC.Server` | `DOTNET_gcServer` | Server vs Workstation | Small containers → 0; throughput services → 1 |
| `System.GC.Concurrent` | `DOTNET_gcConcurrent` | Background gen2 GC | Off for pure batch throughput |
| `System.GC.DynamicAdaptationMode` | `DOTNET_GCDynamicAdaptationMode` | DATAS on/off | Off only with a measured regression |
| `System.GC.HeapHardLimit` / `...Percent` | `DOTNET_GCHeapHardLimit(Percent)` | Absolute cap on the GC heap | Containers where you must leave room for native memory |
| `System.GC.HeapCount` | `DOTNET_GCHeapCount` | Fix Server GC heap count | Pinning behaviour across heterogeneous node sizes |
| `System.GC.ConserveMemory` | `DOTNET_GCConserveMemory` | 0–9, trade throughput for footprint | Memory-pressured services |
| `System.GC.RetainVM` | `DOTNET_GCRetainVM` | Keep freed segments/regions reserved | Churny workloads where re-reserving costs |
| `System.GC.LOHThreshold` | `DOTNET_GCLOHThreshold` | Large-object cut-off | Lots of just-over-85K arrays |
| `System.GC.NoAffinitize` | `DOTNET_GCNoAffinitize` | Don't pin GC threads to cores | Multi-tenant hosts, CPU-limited containers |

Two rules for using this table in an interview. First, **name the setting and the number you'd expect it to change** — "I'd set `GCConserveMemory=5` and expect committed bytes to fall and throughput to drop a few percent; if it doesn't, my hypothesis was wrong." Second, **the default is usually right**; reaching for configuration before measuring allocation behaviour is the classic mid-level move. The order is: measure → reduce allocation/retention → then configure.

---

## Concept 36 — Containers: cgroups, hard limits, heap count, and OOMKilled

The runtime reads cgroup limits and adapts. The parts you must know:

- **Memory limit → heap hard limit.** When a container memory limit is present, the GC sets a heap hard limit of **75% of that limit** by default (with a special case for very small limits). The remaining 25% is for everything that is not the GC heap: thread stacks, loader heaps, native allocations, the runtime itself, and your P/Invoke'd libraries. When the GC heap approaches the hard limit, the GC becomes aggressive; exceeding it throws `OutOfMemoryException`.
- **CPU limit → heap count.** Server GC sizes its heap count from the CPUs available to the process, honouring cgroup CPU quota. A container limited to 2 CPUs on a 64-core node should get 2 heaps, not 64 — verify this if you inherited an older runtime or an unusual orchestrator configuration.
- **`GC.RefreshMemoryLimit()`** (from .NET 8) lets a process pick up a changed cgroup limit at runtime, for vertical-scaling scenarios.

**The distinction that wins points: `OOMKilled` is not `OutOfMemoryException`.**

| | `OutOfMemoryException` | Container `OOMKilled` (exit 137) |
|---|---|---|
| Who decides | The .NET GC, against its hard limit | The kernel, against the cgroup limit |
| What you see | A managed exception, possibly caught | The process vanishes; no stack, no logs |
| Usual cause | GC heap genuinely full, or LOH fragmentation | Total RSS exceeded the limit — often native memory, stacks, or the 25% margin being too small |

So a container that is OOMKilled while `dotnet-counters` shows a 400 MB GC heap is telling you the problem is *not* the GC heap. Candidates for the rest: native libraries (image processing, compression, ML runtimes, database drivers), thread stacks from an unbounded thread count, loader heap growth from dynamic code (Concept 8), memory-mapped files, or a heap hard limit set too close to the container limit.

---

## Concept 37 — Reading the GC: the metrics that actually mean something

From `dotnet-counters monitor --counters System.Runtime`, or the same counters exported via OpenTelemetry (Module 28):

| Metric | What it tells you | Rough "look into it" threshold |
|---|---|---|
| **Allocation Rate (B/sec)** | How hard you're driving the GC | Hundreds of MB/s sustained deserves a look |
| **% Time in GC** | Fraction of CPU spent collecting | > 10% sustained is worth investigating; > 20% is a problem |
| **Gen 0/1/2 GC Count** | Frequency by generation | Gen2 count climbing steadily with load = promotion problem |
| **GC Heap Size** | Live-ish managed bytes | Compare against committed and RSS |
| **GC Committed Bytes** | What the GC holds from the OS | Large gap vs heap size = fragmentation or headroom |
| **Working Set** | RSS | The number your container limit compares against |
| **GC Fragmentation (%)** | Free space inside the heap | High + LOH-heavy = classic LOH fragmentation |
| **Pause time** (`GCMemoryInfo.PauseDurations`, `PauseTimePercentage`) | Actual stop-the-world cost | Compare against your latency budget (Concept 62) |
| **ThreadPool Queue Length** | Not GC, but the best co-indicator | Rising queue + rising pause time = the Module 13 cascade |

`GC.GetGCMemoryInfo()` gives you all of this programmatically, including per-generation sizes, fragmentation, the pause durations of the last collections, whether the last GC was compacting or concurrent, and `MemoryLoadBytes`/`TotalAvailableMemoryBytes` (the machine's view). Exposing a few of these as application metrics is cheap and pays for itself the first time you have an incident.

**The three ratios to reason with:**

1. **Survival rate** (promoted bytes ÷ allocated bytes). High survival means your objects are outliving gen0 — the expensive case.
2. **Gen2 frequency**. Gen2s should be rare and driven by real growth, not by LOH allocations or induced collections.
3. **Committed ÷ live**. Persistently > 2–3× means fragmentation, pinning, or a heap that grew during a spike and hasn't been given back (and if you're pre-.NET 9 Server GC without DATAS, it may never be).

---

# Part D — Lifetime, finalization, and native memory

## Concept 38 — Finalization: the queues, the thread, and resurrection

A type with a finalizer (`~MyType()`, which compiles to an override of `Object.Finalize`) gets special treatment:

1. **At allocation**, a reference to the object is added to the **finalization queue**. This means the object is registered before it is ever used.
2. **When the GC finds it unreachable**, it does *not* collect it. It moves the reference to the **f-reachable queue**, which is a root — so the object is resurrected to live for at least one more collection cycle, along with everything it references.
3. **The finalizer thread** — a single, dedicated thread — dequeues and runs finalizers.
4. **The next collection** finally reclaims the object.

The costs are all structural, and stating them precisely is a strong senior signal:

- **Finalizable objects survive at least one extra GC and get promoted**, dragging their whole object graph to an older generation with them.
- **Finalization is non-deterministic** in both timing and order. You cannot assume a finalizer runs before process exit — since .NET Core, finalizers are **not** guaranteed to run on shutdown at all.
- **There is one finalizer thread.** A single slow or blocking finalizer stalls every other object's finalization, and memory climbs behind it. `!finalizequeue` in SOS showing a large backlog is a specific, diagnosable pathology.
- **Resurrection is possible** — a finalizer can store `this` somewhere reachable. It is legal, it is almost always a bug, and it is a fun interview question.

**Rule:** do not write finalizers. The only legitimate reason is a class that directly owns an unmanaged resource *and* can't use `SafeHandle` — and if you're writing one, you also implement `IDisposable` and call `GC.SuppressFinalize(this)` in `Dispose` so the common path avoids all of the above.

---

## Concept 39 — `IDisposable` and what `Dispose` does not do

`IDisposable` is about **deterministic release of things that are not memory**: file handles, sockets, database connections, locks, native buffers, unmanaged library contexts, cancellation registrations, event subscriptions.

The single most common misconception, and a cheap thing to correct in an interview: **`Dispose` does not free managed memory and does not make the object collectible.** It runs your cleanup method. The object is reclaimed when it becomes unreachable, exactly like any other object.

The full pattern, for a class that owns both managed and unmanaged resources:

```csharp
public class Resource : IDisposable
{
    private SafeFileHandle? _handle;      // managed wrapper over a native resource
    private bool _disposed;

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);        // no finalizer needed now
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            _handle?.Dispose();           // other managed IDisposables
        }
        // free directly-held unmanaged resources here (rare if you use SafeHandle)
        _disposed = true;
    }
}
```

Practical guidance:

- **Sealed classes can skip the virtual `Dispose(bool)` ceremony** — a simple `public void Dispose()` is fine and clearer.
- **If you use `SafeHandle` for every native resource, you never need a finalizer**, and the pattern collapses to the simple form.
- **`using` and `using var`** are the delivery mechanism; `await using` for `IAsyncDisposable`.
- **Double-dispose must be safe.** Idempotence is part of the contract.
- **Don't implement `IDisposable` on a type just because it has `IDisposable` fields it didn't create.** Ownership is the question: who created it decides who disposes it. DI containers dispose what they create, which is where the lifetime bugs in Module 18 come from.

---

## Concept 40 — `IAsyncDisposable`

Introduced because some cleanup is *I/O*: flushing a buffered stream, sending a close frame, committing a transaction, draining a channel. Doing that synchronously in `Dispose` means blocking a thread — the exact sin that causes thread pool starvation (Module 15).

```csharp
public sealed class Publisher : IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        await _channel.FlushAsync();
        await _connection.CloseAsync();
    }
}

await using var publisher = new Publisher();
```

Rules:

- **Implement both** when a type might be used by synchronous callers; make the synchronous `Dispose` do the non-blocking subset, or accept the block and document it. Never `.GetAwaiter().GetResult()` inside `Dispose` in library code.
- **`await using` respects `ConfigureAwait`** via `ConfigureAwait(false)` on the `DisposeAsync` result (`await using var x = y.ConfigureAwait(false)` needs the `ConfiguredAsyncDisposable` form).
- **DI containers understand `IAsyncDisposable`** — `ServiceProvider` must be disposed asynchronously (`await provider.DisposeAsync()`) or it throws when it holds async-disposable singletons. A real production trap.

---

## Concept 41 — `SafeHandle`: the right way to own a native resource

Storing a native handle in an `IntPtr` field has a race condition described in Concept 23: the GC can collect the owning object — and run its finalizer, closing the handle — while a method of that object is still executing, if the JIT determined `this` is no longer used. The result is a handle closed out from under an in-flight native call. Worse, handle values are recycled by the OS, so you can end up operating on *someone else's* file.

`SafeHandle` (and `CriticalHandle`) solve this:

- It is a **finalizable, reference-counted wrapper**. Marshalling a `SafeHandle` into a P/Invoke increments its ref count for the duration of the call, making premature release impossible.
- It has a **critical finalizer** that runs in a constrained region, so it still releases even in aggressive shutdown scenarios.
- It **cannot be recycled into an invalid state** — release happens exactly once.

The BCL uses this throughout: `SafeFileHandle`, `SafeWaitHandle`, `SafeSocketHandle`, `SafeMemoryMappedViewHandle`. For your own interop, derive from `SafeHandleZeroOrMinusOneIsInvalid` and override `ReleaseHandle()`.

```csharp
sealed class SafeWidgetHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    public SafeWidgetHandle() : base(ownsHandle: true) { }
    protected override bool ReleaseHandle() => NativeMethods.WidgetDestroy(handle);
}
```

**The interview-grade sentence:** *"I'd never hold a raw `IntPtr` for a native resource. `SafeHandle` fixes the premature-collection race, is ref-counted across P/Invoke, and has a critical finalizer — which also means I don't have to write a finalizer myself."*

---

## Concept 42 — Weak references, `ConditionalWeakTable`, and `DependentHandle`

A **weak reference** lets you reference an object without keeping it alive.

```csharp
var weak = new WeakReference<Bitmap>(bitmap);
if (weak.TryGetTarget(out var b)) { /* still alive */ }
```

Two flavours: *short* weak references are cleared as soon as the object is unreachable; *long* ones (`trackResurrection: true`) survive until after finalization, which matters only if you care about resurrection.

Where weak references belong:

- **Caches where the cached item is expensive but reconstructible** and the consumer holds strong references while in use. This is a narrower case than people think, because the GC clears weak references on any collection, so a weak cache has an unpredictable hit rate. Prefer a bounded cache with an explicit eviction policy (`MemoryCache`, `HybridCache` from Module 10) for most workloads.
- **Breaking event-handler leaks** (the weak event pattern), though the cleaner fix is usually explicit unsubscription.
- **Observer registries** where the subject shouldn't own the observers' lifetimes.

`ConditionalWeakTable<TKey, TValue>` is the specialist tool: it attaches data to an object without affecting its lifetime, and crucially the *value* does not keep the *key* alive even if the value references the key. It is implemented with **ephemerons**, which the GC handles specially. Use it for attached state (the runtime itself uses it for things like dynamic object metadata). `DependentHandle` is the public primitive underneath (public since .NET 5).

---

## Concept 43 — `GCHandle`, `fixed`, and `Memory<T>.Pin()`

Four handle types, and it is worth knowing all four by name:

| `GCHandleType` | Effect |
|---|---|
| `Normal` | A strong root — keeps the object alive, allows it to move |
| `Weak` | Doesn't keep alive; cleared when unreachable |
| `WeakTrackResurrection` | Doesn't keep alive; cleared after finalization |
| `Pinned` | Strong root **and** the object cannot move |

`GCHandle` is what you use to hand an object's identity to native code (as an opaque token, via `GCHandle.ToIntPtr`) and to pin a buffer for the lifetime of a native operation. **Every `GCHandle.Alloc` needs a matching `Free`** — an unfreed handle is a permanent root, i.e. a leak that `!gcroot` will attribute to "handle."

`fixed` is the scoped, compiler-managed version for short pins:

```csharp
fixed (byte* p = buffer) { NativeMethods.Process(p, buffer.Length); }
```

`Memory<T>.Pin()` returns a `MemoryHandle` that works uniformly for managed arrays, native memory, and custom `MemoryManager<T>` implementations — the right abstraction when you're writing library code that shouldn't care where the memory came from.

**The architectural rule:** pins should be either *very short* (a `fixed` block around a single call) or *permanent and few* (POH-allocated I/O buffers). The dangerous middle is many medium-lived pins scattered through the heap — for example, pinning each request's buffer for the duration of an async socket operation, at high concurrency, without pooling. That is a fragmentation engine.

---

## Concept 44 — Native memory and `GC.AddMemoryPressure`

Memory allocated outside the GC heap is invisible to the GC:

- `NativeMemory.Alloc/Free` (.NET 6+, the modern, allocation-aligned API)
- `Marshal.AllocHGlobal` / `AllocCoTaskMem`
- Memory-mapped files (`MemoryMappedFile`)
- Anything allocated inside a native library you P/Invoke into

The problem this creates: a small managed object (say, 40 bytes) that owns a 200 MB native buffer looks like 40 bytes to the GC. The GC sees no pressure, doesn't collect, the finalizer/`Dispose` never runs, and you exhaust the container's memory while the GC heap reports "everything is fine."

The fix is to tell it:

```csharp
_native = NativeMemory.Alloc(size);
GC.AddMemoryPressure(size);       // "this managed object is worth `size` bytes"
// ... on release:
NativeMemory.Free(_native);
GC.RemoveMemoryPressure(size);
```

This is how `Bitmap`, some crypto wrappers, and several native-backed BCL types behave, and it is genuinely underused in application code that wraps native libraries. A related API, `HandleCollector`, does the same thing for limited handle counts.

For interviews: **"our container is OOMKilled but the GC heap is small"** should immediately produce "native memory, thread stacks, loader heaps, or fragmentation — and the first thing I'd check is whether we wrap a native library without memory pressure." (Concept 36.)

---

## Concept 45 — Interop and the GC: `GC.KeepAlive`, marshalling, and callbacks

Three interop-and-GC interactions that appear in real bugs:

**1. Premature collection.** As in Concept 23 — the JIT can consider an object dead before your method finishes. If you passed a field of it (e.g. a raw handle) to native code, the object may be finalized mid-call.

```csharp
var stream = new FileStream(...);
NativeMethods.Use(stream.SafeFileHandle.DangerousGetHandle());
GC.KeepAlive(stream);    // forces liveness to this point
```

`GC.KeepAlive` is a method that does nothing; its only effect is to extend the JIT-computed lifetime of its argument. `SafeHandle` removes the need for it in most cases (Concept 41).

**2. Marshalling copies.** Marshalling a `string` to `char*`/`byte*` allocates and copies. Marshalling a blittable array can pin instead of copying. Marshalling a struct with `LayoutKind.Auto` won't work at all. Knowing which path your P/Invoke takes is the difference between a fast interop layer and a very slow one; `[LibraryImport]` (the source-generated replacement for `[DllImport]`, and the only option under Native AOT) makes the marshalling explicit and inspectable.

**3. Delegates passed to native code.** A managed delegate handed to native code as a function pointer must be kept alive by *you*; the GC has no idea the native side is holding it. The classic crash is "works in debug, crashes after a few minutes in production" — the delegate was collected. Store it in a field or a `GCHandle` for as long as the native side may call it. `[UnmanagedCallersOnly]` static methods avoid the issue entirely and are the modern approach.

---

# Part E — Allocation-conscious design

## Concept 46 — What an allocation actually costs

Break it into four parts, because different workloads pay different parts:

1. **The allocation itself** — a pointer bump, a few nanoseconds. Genuinely cheap.
2. **Zeroing** — proportional to size; noticeable for large buffers.
3. **The collection** — amortized. Dying in gen0 is nearly free; surviving is expensive. **This is where the real cost lives.**
4. **Cache and memory bandwidth effects** — this is the one people forget. Allocating constantly evicts your working set from L1/L2. A "zero-allocation" rewrite sometimes wins mostly on cache locality, not on GC time.

So the useful mental model is not "allocations are bad" but:

> **Short-lived allocations that die in gen0 are cheap. Allocations that survive are expensive. Allocations ≥ 85,000 bytes are expensive immediately. Allocations that are pinned are expensive structurally.**

This is why the highest-value optimizations in real services are usually (a) eliminating LOH allocations, (b) eliminating retention, and (c) eliminating allocation in the hottest 1% of code — in that order — and why micro-optimizing allocation in a method that runs 50 times per request is a waste of an afternoon.

---

## Concept 47 — Struct vs class: a decision procedure

Microsoft's own guidance is a starting point (logically represents a single value, instance size under 16 bytes, immutable, not frequently boxed). Here is the version to say out loud in an interview, because it shows the trade-off reasoning rather than the rule:

**Choose `struct` when all of these hold:**
- It represents a **single value** with no identity (`Money`, `Point`, `Timestamp`, `OrderId`).
- It is **immutable** (declare it `readonly struct`).
- It is **small** — roughly ≤ 16–24 bytes, or larger only if you'll always pass it by `in`/`ref`.
- It **won't be boxed frequently** — not stored as `object`, not used through non-generic interfaces.
- You have a reason: dense arrays, a hot path with measured allocation pressure, avoiding an allocation per element, or interop layout.

**Choose `class` when:**
- Identity matters, or it's mutable, or it's polymorphic, or it's large, or it has a lifetime you manage (`IDisposable`), or you just don't have evidence either way.

**The killer applications for structs** are the ones to name: large arrays of records (`Candle[]`, `Point3D[]`, `Reading[]`) where a struct array is one object and a class array is n+1 objects with n references to trace; keys and IDs in hot dictionaries; and value objects in DDD (Module 22) where `readonly record struct` gives you correct equality for free.

**The traps to name:** mutable structs in collections (you mutate a copy), structs with reference fields (they still create GC work), oversized structs copied on every call, and structs behind interfaces (Concept 15).

---

## Concept 48 — Pooling: `ArrayPool`, `ObjectPool`, `RecyclableMemoryStream`

Pooling trades GC work for manual lifetime management. Sometimes that's an excellent trade; often it is not.

**`ArrayPool<T>.Shared`** — the default answer for temporary buffers.

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(size);   // may be LARGER than requested
try { /* use buffer.AsSpan(0, size) */ }
finally { ArrayPool<byte>.Shared.Return(buffer, clearArray: containsSensitiveData); }
```

Three rules that catch everyone at least once: the returned array is **at least** the requested size (never use `.Length`), **returning is mandatory** (use `try/finally`), and **never touch a buffer after returning it** — that's a use-after-free, and it produces data corruption that looks like a concurrency bug. The shared pool has a maximum array size (1 MB by default); larger rents allocate normally.

**`ObjectPool<T>`** (`Microsoft.Extensions.ObjectPool`) for expensive-to-construct objects — `StringBuilder`, parser state, large reusable graphs. ASP.NET Core uses it internally.

**`RecyclableMemoryStream`** (`Microsoft.IO.RecyclableMemoryStream`) replaces `MemoryStream` with a pooled, chunked implementation that avoids the LOH doubling problem in Concept 28. If your service buffers response bodies, serializes large payloads, or proxies content, this is frequently the single highest-value change available.

**When pooling is a mistake:**
- The objects are small and short-lived — you're replacing a cheap gen0 death with permanent gen2 residency plus synchronization.
- The pool is unbounded — now it's a memory leak with good intentions.
- The lifetime is unclear — pooling turns a class of bug from "slow" into "corrupt."
- You haven't measured. Pooling adds real complexity; it must pay for it.

Note the GC consequence: **pooled objects live forever, which means they live in gen2.** If a pooled object references short-lived objects, you're writing old→young references and dirtying cards (Concept 26). Pools should hold *buffers*, not object graphs.

---

## Concept 49 — Zero-allocation patterns worth knowing

A short catalogue, with the shape of each:

**Conditional `stackalloc` with a pooled fallback** — the canonical high-performance buffer idiom:

```csharp
const int StackLimit = 256;
byte[]? rented = null;
Span<byte> buffer = length <= StackLimit
    ? stackalloc byte[StackLimit]
    : (rented = ArrayPool<byte>.Shared.Rent(length));
try { /* work on buffer[..length] */ }
finally { if (rented is not null) ArrayPool<byte>.Shared.Return(rented); }
```

**`string.Create`** — build a string with exactly one allocation and no intermediate:

```csharp
string s = string.Create(length, state, static (span, st) => { /* fill span */ });
```

**UTF-8 everywhere it's possible.** `"literal"u8` gives a `ReadOnlySpan<byte>` with no allocation at all; `Utf8.TryWrite`, `IUtf8SpanFormattable`, and `System.Text.Json`'s UTF-8 path let you go from data to bytes without ever creating a `string`. For a JSON API, this is often the biggest single allocation win available.

**`SearchValues<T>`** (.NET 8+) — precomputed, vectorized multi-value search; replaces `IndexOfAny(char[])` in hot parsing paths.

**Slicing instead of substringing.** `ReadOnlySpan<char>` slices are free; `Substring` allocates. Parsers should take spans.

**`TryFormat`/`TryParse` over `ToString`/`Parse`** — write into a caller-provided span.

**`CollectionsMarshal.GetValueRefOrAddDefault`** — one dictionary lookup instead of two for the get-or-add pattern; `CollectionsMarshal.AsSpan(list)` to iterate a `List<T>` without the enumerator.

**`ValueTask` / `IValueTaskSource`** for hot async paths that usually complete synchronously (Module 15).

**`ArrayPool` + POH** for long-lived I/O buffers (Concept 29).

Use these where they belong — parsers, serializers, protocol layers, hot loops — and nowhere else. The readability cost is real.

---

## Concept 50 — The allocations you don't see

Worth being able to list quickly, because "what allocates in this method?" is a common live-coding prompt:

| Source | What allocates |
|---|---|
| **Closures** | A display class per capturing lambda scope; a `Func`/`Action` per delegate creation |
| **Non-capturing lambdas** | Cached by the compiler — free after the first use (use `static` lambdas to guarantee it) |
| **LINQ** | An enumerator and a closure per operator in the chain, plus boxing of struct enumerators |
| **`foreach` over `IEnumerable<T>`** | Boxes the struct enumerator (the target of .NET 9–11 escape analysis) |
| **Iterators (`yield return`)** | One state machine object per enumeration |
| **`params object[]`** | An array plus a box per value-type argument |
| **String concatenation / interpolation** | The result string, plus intermediates in loops |
| **`async`** | A state machine box and a `Task` — when and only when the method suspends (Concept 51) |
| **`Task.WhenAll`/`WhenAny`** | The array and the combining task |
| **Exceptions** | The exception object plus stack trace capture — expensive; never use for control flow |
| **`ToList()`/`ToArray()`** | A full copy; and `ToArray()` on a large collection may hit the LOH |
| **Boxing** | Everything in Concept 14 |
| **Logging** | Message formatting, boxed arguments, scope objects — even when the level is disabled |

Two of these deserve emphasis for server code. **Logging** is routinely the top allocator in an ASP.NET Core service, and the `LoggerMessage` source generator fixes it at essentially zero readability cost. **LINQ in a per-request hot path** is usually fine at 1,000 rps and usually not fine at 100,000 rps — the honest answer is "measure, and if it's hot, rewrite that method, not the codebase."

---

## Concept 51 — Async state machines: where the "stack" goes

The compiler rewrites an `async` method into a state machine struct implementing `IAsyncStateMachine`, holding every local that lives across an `await` plus the awaiter and a builder.

The crucial detail most candidates miss: **the state machine starts on the stack.** It is only boxed onto the heap when the method **actually suspends** — that is, when an `await` hits an incomplete operation. An `async` method whose awaits all complete synchronously (a cache hit, a buffered read) allocates nothing for the state machine, and `ValueTask` lets it allocate nothing for the result either.

So the allocation profile of an async method is:

| Scenario | Allocations |
|---|---|
| All awaits complete synchronously, returns `ValueTask` | Zero |
| All awaits complete synchronously, returns `Task` | `Task` object (or the cached `Task.CompletedTask`/cached `Task<bool>` values) |
| Suspends at least once | Boxed state machine + `Task` + the awaiter's continuation machinery |

Implications:
- **Don't `async`-ify methods that never await.** You pay the machinery for nothing.
- **`ValueTask<T>` is for hot paths that usually complete synchronously** — not a general replacement for `Task<T>`, and it has real consumption rules (await once, don't store, don't `WhenAll` directly).
- **Fewer, coarser awaits** allocate less than many fine-grained ones, though readability usually wins.
- **Deep async call chains** each allocate their own state machine when they suspend; a 10-frame async stack that suspends allocates 10 state machines plus 10 tasks.

**What's changing:** .NET 11 introduces **runtime async**, where the JIT and runtime perform the state-machine transformation instead of the C# compiler. It is opt-in behind a feature switch in .NET 11 and expected to become the default in .NET 12, with the goal of removing much of the per-suspension allocation and indirection. Knowing this exists — and that it is not yet the default — is a good "what's coming" answer. Module 15 goes deep.

---

## Concept 52 — Caching and the mid-life crisis

The generational GC's worst case is an object that **survives gen0 and gen1 and then dies shortly after being promoted to gen2**. You paid the copying cost of two promotions, you added it to the expensive generation, and then it becomes garbage that only a full collection can reclaim.

The name for this is **mid-life crisis**, and the most common cause is caching with a medium-length lifetime: session state, per-request caches with a short sliding expiration, object pools sized wrong, and connection or buffer caches that churn.

How to reason about it:

- **Either die young or live long.** The two cheap lifetimes are "dies in gen0" and "lives for the process." The expensive one is everything in between.
- **Bounded caches with stable contents** are fine: they reach gen2 and stay there, and gen2 stops caring about them.
- **Caches with high turnover** are the problem: every eviction is gen2 garbage, and gen2 garbage accumulates until a full GC.
- **The fix is usually a policy change**, not a GC setting: make the cache smaller and stabler, or make the cached objects cheap enough to not cache at all, or move the cache out of process (Redis, per Module 10 — which also removes the memory from your pod's budget entirely).

This connects directly to Module 10's caching material: a local in-process cache has a *GC cost* that a distributed cache does not, and that cost belongs in the trade-off next to hit rate and consistency.

---

## Concept 53 — Deabstraction: what the JIT now does for you (.NET 9 → 11)

Worth knowing precisely because it is current, it is frequently asked as "what's new," and it changes the advice.

**.NET 9** introduced limited escape analysis, including stack allocation of boxes that don't escape.

**.NET 10** extended this substantially:
- **Small, fixed-size arrays of value types** that don't outlive the method are stack-allocated (`int[] n = {1,2,3};` in a local loop — no heap allocation).
- **Small arrays of reference types** likewise.
- **Escape analysis through local struct fields** — an object referenced by a field of a non-escaping struct no longer counts as escaping.
- **Delegates** — if a `Func` doesn't escape, the delegate object is stack-allocated (the closure display class is still heap-allocated; the team has said closures are next).
- **Array interface method devirtualization** and **array enumeration deabstraction**, so `foreach` over an array typed as `IEnumerable<T>` can devirtualize, inline, and stack-allocate the enumerator.
- Plus supporting work: better inlining when a callee becomes devirtualizable, inlining of some `try/finally` methods, improved code layout, and improved Arm64 write barriers.

**.NET 11** (RC1 as of September 2026; GA 10 November 2026) continues: **conditional escape analysis** extended so that `foreach` over an `IEnumerable` held in an *instance field* now also stack-allocates its enumerator — one published measurement goes from ~13.9 ns with a 32-byte allocation on .NET 10 to ~2.7 ns with zero allocation. Plus devirtualization of non-shared generic virtual methods, better induction-variable analysis, and runtime async.

**What this means for advice you give:**

1. **The gap between idiomatic and hand-tuned C# keeps narrowing.** "Don't use LINQ/interfaces/`foreach` because of allocations" was decent advice in 2016 and is increasingly wrong. Measure on your target runtime.
2. **Upgrading the runtime is a performance strategy.** Moving a service from .NET 8 to .NET 10 typically buys measurable allocation and latency improvements with zero code change — a legitimate architectural argument with a cost (testing, LTS timing) attached.
3. **Small methods help the JIT.** Escape analysis works within a method after inlining, so keeping methods inlinable and avoiding gratuitous virtual dispatch gives the optimizer material to work with.
4. **Don't assume.** These optimizations are conditional. Verify with BenchmarkDotNet's allocation column, or by reading the disassembly (`[DisassemblyDiagnoser]`, or sharplab.io).

---

## Concept 54 — Native AOT's memory profile

Native AOT changes the memory picture in ways worth knowing concretely:

- **No JIT, no tiering, no PGO data** — the memory the JIT would use for compilation and instrumentation simply isn't there. Footprint drops substantially, often by tens of megabytes for a small service.
- **No loader heaps growing at runtime** — types are fixed at build time.
- **Startup is milliseconds**, because nothing needs to be compiled or type-loaded from metadata in the usual way.
- **Workstation GC is the default** for Native AOT; Server GC is available via the usual configuration.
- **The same GC** otherwise — generations, regions, LOH, everything in Part C applies.

The costs are the closed-world restrictions: no `Reflection.Emit`, no runtime assembly loading, reflection only over what the trimmer could see and root, and third-party libraries that use reflection dynamically may not work. The ecosystem has moved a long way — System.Text.Json source generation, `LibraryImport`, `LoggerMessage`, Minimal APIs, gRPC, and the `Microsoft.Extensions.*` stack are all AOT-friendly — but EF Core and some reflection-heavy libraries remain constraints.

Concept 64 turns this into an architectural decision.

---

## Concept 55 — When not to optimize allocations

The discipline that separates a senior engineer from an enthusiast:

- **If GC isn't in your top three costs, this work is negative value.** Check `% Time in GC` first. At 2%, nothing in Part E will move your latency; your problem is I/O, serialization, the database, or a lock.
- **Optimize the hot 1%.** Amdahl's law applies. A 10× improvement in a method consuming 3% of CPU buys you 2.7%.
- **Every span-based rewrite costs readability, testability, and bug risk.** Buffer lifetime bugs are worse than the allocations they replace.
- **The runtime is getting faster than your optimizations are aging.** Code written to dodge a 2018 JIT limitation is now slower and uglier than the naive version (Concept 53).
- **Measure before and after, in an environment that resembles production.** BenchmarkDotNet for micro, load test for macro, production counters for truth.
- **Say the trade-off out loud in an interview.** "I'd profile first; if allocation is genuinely the bottleneck I'd start with the LOH and retention, and only then reach for spans in the specific hot path" is a better answer than any list of techniques.

---

# Part F — Diagnosing memory problems

## Concept 56 — The four pathologies

Every .NET memory problem is one of four things, and naming which one you're looking at is the first move in any diagnosis.

| Pathology | Symptom | Cause | Fix |
|---|---|---|---|
| **Leak (retention)** | Memory grows monotonically, never returns to baseline after load drops | Something still reachable that should be dead | Find the root; break the reference |
| **Bloat** | High but *stable* memory | The working set genuinely is that large: caches, big collections, inefficient layouts | Reduce what you keep, or accept and size for it |
| **Churn** | High allocation rate, high `% Time in GC`, frequent gen0/gen1, memory stable | Allocating too much too fast | Reduce allocations in the hot path |
| **Fragmentation** | Committed ≫ live; allocation failures despite free space | LOH sweeping, pinning, or a spike that never compacted | Pool large buffers, POH, LOH compaction, reduce pins |

The distinguishing test is **shape over time**, not level. Put memory on a graph with load on the same axis:

- Grows with load, returns to baseline after → **healthy**.
- Grows with load, never returns → **leak**.
- Flat and high → **bloat**.
- Flat memory but high GC CPU → **churn**.
- Memory high, live set low → **fragmentation** (check LOH and pinned handles).

Naming the shape before naming a tool is exactly the reasoning an interviewer is listening for.

---

## Concept 57 — The .NET leak taxonomy

Since the GC is essentially never wrong, every "leak" is a retention bug. The complete practical list:

1. **Static collections.** A `static Dictionary<string, Thing>` that only ever gets added to. The most common leak in .NET, by a wide margin. Also: static caches keyed by tenant/user/correlation-ID.
2. **Event handlers.** `publisher.Event += subscriber.Handler` makes the *publisher* hold the *subscriber*. A long-lived publisher (a singleton, a static, a UI control) plus short-lived subscribers is an unbounded leak. Unsubscribe in `Dispose`, or use weak events.
3. **Closures capturing more than you think.** A lambda captures the entire display class for its scope, not just the variables it uses — so capturing a small `int` in the same scope as a big object can retain the big object. Registering that lambda somewhere long-lived retains everything.
4. **Timers and background tasks.** `System.Threading.Timer` and `System.Timers.Timer` hold their callback target; an un-disposed timer keeps its owner alive forever. Same for un-cancelled `Task.Run` loops and `CancellationTokenRegistration`s that are never disposed — registering a callback on a long-lived `CancellationToken` retains it until the token dies.
5. **DI lifetime mistakes.** A singleton capturing a scoped service ("captive dependency") keeps that scope's whole graph — including a `DbContext` and its change tracker — alive for the process lifetime. `IServiceScope`s created manually and not disposed do the same. (Module 18.)
6. **Unbounded caches.** `MemoryCache` without a size limit, a `ConcurrentDictionary` used as a cache, or a dictionary keyed by something unbounded (request ID, URL).
7. **`string.Intern` on runtime data** (Concept 18).
8. **Un-freed `GCHandle`s** and pinned handles (Concept 43).
9. **Collectible ALC escapes and dynamic code generation** (Concept 8) — visible as RSS growth without heap growth.
10. **Native memory** owned by managed wrappers without pressure accounting (Concept 44).
11. **`HttpClient` misuse** — the opposite problem (socket exhaustion from creating many) plus the DNS problem from a permanent singleton; use `IHttpClientFactory` and `PooledConnectionLifetime` (Module 13, Concept 12).
12. **`ThreadLocal<T>` / `AsyncLocal<T>`** holding large objects on pooled threads that never die.

For an interview, having six or seven of these ready — and being able to say *why* each one retains — is worth far more than knowing the tools.

---

## Concept 58 — The diagnostic workflow

A workflow you can narrate end to end, using only free, cross-platform, in-box tools:

**Step 1 — Confirm the shape (production, low overhead).**
```
dotnet-counters monitor -p <pid> --counters System.Runtime
```
Read: working set, GC heap size, allocation rate, gen0/1/2 counts, `% Time in GC`, fragmentation. Classify with Concept 56.

**Step 2 — Find *what* is on the heap.**
```
dotnet-gcdump collect -p <pid>          # low-overhead, no process dump, opens in VS / PerfView
```
Take **two** dumps, ten minutes apart, under load. The diff is the answer — absolute counts tell you much less than "what grew."

**Step 3 — Find *who is holding it*.**
```
dotnet-dump collect -p <pid>
dotnet-dump analyze core_dump
> dumpheap -stat                        # types by count and size
> dumpheap -mt <MethodTable> -min 1000  # instances of the suspicious type
> gcroot <address>                      # THE command: the reference chain from a root
> objsize <address>                     # retained size of an object graph
> eeheap -gc                            # generation/region sizes, committed vs reserved
> finalizequeue                         # finalizer backlog
> gchandles -stat                        # handle counts by type (pinned handles!)
```

**Step 4 — If the story is *rate* or *pauses*, trace instead.**
```
dotnet-trace collect -p <pid> --profile gc-verbose
```
Analyse in PerfView (GCStats gives pause durations, suspension time, promoted bytes, allocation by type and by stack).

**Step 5 — For a reproducible case, measure it properly.** BenchmarkDotNet with `[MemoryDiagnoser]` (Concept 61).

Two rules that make this workflow work in practice: **capture under load, not at idle**, and **always diff**. A single snapshot tells you what is there; two snapshots tell you what is growing, and only the second question has an answer you can act on.

---

## Concept 59 — From symptom to root with SOS

The one command worth being fluent in is `gcroot`. A worked example of the reasoning, which is what an interviewer wants to hear:

> "`dumpheap -stat` shows 4.1 million `System.String` and 900,000 `OrderSnapshot`, total 2.3 GB. `OrderSnapshot` shouldn't exist outside a request. I pick a few instances with `dumpheap -mt`, run `gcroot` on each, and they all resolve to the same chain: a static field on `TelemetryBuffer` → a `ConcurrentQueue` → nodes → snapshots. So it's leak type 1 — a static collection. The queue is drained by a background flusher that started failing silently three days ago when the exporter endpoint changed. The fix is a bounded queue that drops on overflow plus an alert on drop count; the leak was the *absence of a bound*, and the outage was the absence of the alert."

That narrative has the structure interviewers score: measurement → classification → root → fix → the systemic lesson.

Other high-value SOS patterns:

- **Duplicate strings dominating the heap** → deduplicate at the boundary (parse once, share instances), or stop materializing them (spans, UTF-8).
- **`byte[]` on the LOH with high fragmentation** → pooled buffers, `RecyclableMemoryStream`.
- **Huge `finalizequeue`** → a blocking finalizer, or a type that should be `IDisposable` and isn't being disposed.
- **Many pinned handles** → pinning in a hot path; move to POH or pool.
- **`Task`/state-machine objects in the millions** → an async leak: tasks that never complete, or an unbounded fan-out.

---

## Concept 60 — Diagnosing pauses: too often, too slow, or too slow to stop

When latency is the symptom, split the problem three ways before touching anything:

| Observation | Meaning | Fix direction |
|---|---|---|
| Many gen0/gen1 GCs, each short | **Too often** — allocation rate is high | Reduce allocations in the hot path; check gen0 budget/DATAS |
| Few GCs, each long | **Too slow** — large live set surviving each collection | Reduce retention; check cache sizes and promotion rate |
| Frequent gen2 | LOH allocations, induced `GC.Collect`, or heavy promotion | Pool large buffers; remove `GC.Collect`; fix mid-life crisis |
| Pause ≫ GC duration | **Suspension** is the cost, not collection | Long loops without safe points; too many threads; blocking in native code |
| Pause time fine, latency bad | Not GC at all | Go look at I/O, locks, thread pool starvation (Module 15) |

The measurements that separate these: `GCMemoryInfo.PauseDurations` and `PauseTimePercentage` for the pause itself, PerfView's GCStats for suspension time vs GC time and promoted bytes per collection, and `dotnet-counters` for rate.

The connection back to Phase 3: a GC pause is an **omission/timing failure** from the caller's perspective (Module 13, Concept 3). Under load, longer pauses raise concurrency, higher concurrency raises allocation, and the loop can go metastable — which is why "our latency degraded and never recovered until we shed load" incidents so often turn out to have a GC component. A senior answer names that loop.

---

## Concept 61 — Measuring memory properly

**BenchmarkDotNet** is the standard, and `[MemoryDiagnoser]` is the reason it matters here:

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
[SimpleJob(RuntimeMoniker.Net100, baseline: true)]
public class ParseBench
{
    [Benchmark] public int Current() => Parse(_input);
    [Benchmark] public int SpanBased() => ParseSpan(_input);
}
```

It reports **Allocated** (bytes per operation) and **Gen0/Gen1/Gen2 collections per 1,000 operations**, which is exactly the vocabulary of Part C. Running the same benchmark against two runtime monikers is also the cleanest way to demonstrate Concept 53 to yourself and to a team.

Things to get right:
- **Warm up** (BenchmarkDotNet does; hand-rolled loops don't) — otherwise you measure tier-0 code.
- **Don't benchmark in Debug**, and don't trust a benchmark whose result the JIT can elide — return the value or use `Consumer`.
- **Micro-benchmarks don't prove macro results.** A 40% allocation reduction in a method that isn't hot changes nothing. Pair with a load test.

**Allocation assertions in tests** are underrated and cheap: `GC.GetAllocatedBytesForCurrentThread()` before and after a hot-path operation, asserting a ceiling, catches allocation regressions in CI without a benchmark suite.

**In production**, the counters from Concept 37 exported through OpenTelemetry give you the same vocabulary continuously, which means the next incident starts with data instead of a repro attempt.

---

# Part G — Architecture-level consequences

## Concept 62 — GC pauses and the latency budget

Make it arithmetic, not vibes. If your p99 budget for an endpoint is 200 ms and the work itself takes 120 ms, you have 80 ms of slack for queuing, scheduling, and GC. Then:

- **Pause cost ≈ pause frequency × pause duration × probability of overlapping a request.** A 5 ms pause every 200 ms affects roughly 2.5% of request-time — which lands squarely on p99.
- **Gen0 pauses** are typically sub-millisecond to a few milliseconds. Usually invisible.
- **Gen2 pauses** with background GC are two short pauses; a *blocking* gen2 on a large heap can be tens to hundreds of milliseconds. This is what shows up in the p99.9.
- **Bigger heaps mean longer full collections.** A 32 GB heap is not free; it is a latency decision as much as a capacity one.

The architectural moves, in order of leverage:
1. **Reduce retention** (smaller live set → shorter gen2).
2. **Eliminate LOH churn** (fewer forced gen2s).
3. **Remove induced `GC.Collect()`** (people are still shipping this).
4. **Keep background GC on**; don't set `Concurrent=false` on a latency-sensitive service.
5. **Consider more, smaller instances** rather than one large-heap instance — a genuinely architectural answer that connects to Module 6's scale-out material.
6. Only then, configuration (Concept 35).

And the honest framing for an interview: *"GC pause belongs in the latency budget explicitly, alongside network and dependency time. Most services have enough slack that it never matters; for the ones that don't, the fix is almost always a smaller live set rather than a GC setting."*

---

## Concept 63 — Sizing instances: heap, cores, DATAS, and autoscaling

Four things to get right when sizing a .NET workload:

1. **Heap count follows CPU.** With Server GC (pre-DATAS), heap count = available cores, so memory scales with the instance size independently of the workload. With DATAS (.NET 9+), heap count adapts to load instead — which makes vertical scaling far more predictable.
2. **The heap hard limit follows the container limit** (75% by default). Leave headroom for native memory, thread stacks, and loader heaps, and *increase* the headroom if you use native libraries. If you're OOMKilled with a healthy heap, lower the percent rather than raising the container limit blindly.
3. **Requests and limits should be close for .NET.** A Kubernetes memory request far below the limit invites the scheduler to overcommit a node, and a .NET process that sizes itself against the *limit* will happily grow into memory the node doesn't have.
4. **Autoscaling on memory is usually wrong for .NET.** The GC grows the heap to fill available memory when it is cheap to do so; memory utilization is therefore a lagging, sticky, and partly self-inflicted signal. HPA on memory can trigger scale-out from GC headroom rather than from load, and — worse — never scales back in. Scale on CPU, RPS, or queue depth. DATAS makes memory a *better* signal than it used to be (heap now tracks live data), but CPU and concurrency remain the honest ones.

A defensible default for a typical ASP.NET Core service: **Server GC + DATAS (the .NET 9+ default), 2–4 vCPU, 1–2 GB limit with request ≈ limit, scale on CPU/RPS, alert on `% Time in GC` and working set trend.**

---

## Concept 64 — Native AOT vs JIT as an architectural decision

Where Native AOT wins decisively:

- **Serverless / scale-to-zero** (Azure Functions, Container Apps scaling from zero) — cold start goes from hundreds of milliseconds to tens.
- **CLI tools and short-lived processes** — where the process lifetime is dominated by startup.
- **Sidecars, agents, and dense multi-tenant hosting** — where per-process footprint × process count is the cost.
- **Edge and constrained environments** — small images, no runtime to install.

Where the JIT wins:

- **Long-running services** — tier-1 plus dynamic PGO beats AOT code quality, and startup amortizes to nothing.
- **Anything with heavy reflection, dynamic loading, plugins, or runtime code generation** — EF Core, some serializers, some DI and mapping libraries.
- **Teams without the appetite** for trimming warnings, AOT-compatible dependency auditing, and a new class of build-time failures.

The middle option people forget: **ReadyToRun**. It buys most of the startup improvement with none of the restrictions, at the cost of a larger assembly. For a container that restarts on deploy but runs for hours, R2R is very often the right answer and Native AOT is over-engineering.

**The architect framing:** *"This is a startup-and-footprint decision, not a speed decision. If cold start or density is in our SLO or our bill, AOT is worth the constraints; if not, R2R gives us most of the startup win for free, and JIT gives us the best steady-state throughput. I'd also check our dependency graph for AOT compatibility before promising anything."*

---

## Concept 65 — A defensible GC configuration per workload archetype

| Workload | Configuration | Reasoning |
|---|---|---|
| **ASP.NET Core API, ≥2 cores, ≥1 GB** | Server GC + DATAS (defaults on .NET 9+) | Throughput with adaptive footprint; the default is right |
| **Small container (≤1 core, ≤512 MB)** | Workstation GC, consider `ConserveMemory` | Server GC's per-core heaps and budgets don't fit |
| **Batch / ETL, throughput only** | Server GC, `Concurrent=false`, possibly DATAS off | Pauses don't matter; maximize throughput |
| **Low-latency (trading, real-time bidding)** | Server GC + background GC, aggressive allocation reduction, POH-pooled buffers, possibly `SustainedLowLatency` in a window | Pause budget is the constraint; the real work is reducing the live set |
| **Streaming / high-throughput ingestion** | Server GC + DATAS, pooled buffers, `RecyclableMemoryStream`, span-based parsing | LOH and churn are the two risks |
| **Desktop / client** | Workstation GC (default) + background GC | Responsiveness on a shared machine |
| **Serverless function** | Native AOT or R2R, Workstation GC | Startup and footprint dominate |
| **Dense multi-tenant host** | Workstation GC or Server GC with fixed low `HeapCount`, hard limits per process | Fairness and density over per-process throughput |

The meta-point to state: **start from the default, change one thing, measure, and write down why.** A configuration you can't justify with a number is a liability for the next person — which is Module 31's ADR argument applied to runtime tuning.

---

## Concept 66 — Memory as a cost and capacity lever

The chain an architect should be able to walk in a cost conversation:

**Footprint per instance → instances per node → nodes → bill.**

- A service whose footprint drops from 1.8 GB to 900 MB can run twice as densely, which halves the node count for a memory-bound fleet. At 60 pods, that is real money and it's a conversation executives follow (Module 33).
- Conversely, **memory is often the cheapest resource to buy.** Two engineer-weeks of span-based rewriting to save 400 MB across 10 pods is a bad trade unless it also buys latency. Say this out loud; it is exactly the judgment being scored.
- **A runtime upgrade is frequently the best memory optimization available** — .NET 7's regions, .NET 9's DATAS, and .NET 10's escape analysis each moved real numbers for real services with no code change.
- **Moving state out of process** (Redis, per Module 10) converts a memory problem into a network problem, with its own consistency and latency costs. Sometimes right, and the trade-off should be explicit.

---

## Concept 67 — The memory review: a checklist you can narrate

For a service under review, in order:

1. **What is the live set, and why is it that size?** Name the largest three categories. If nobody knows, that's the first finding.
2. **Working set vs GC heap.** If the gap is large, account for it: stacks, native, loader heaps, fragmentation.
3. **Allocation rate and `% Time in GC` under peak load.** Two numbers; if nobody has them, that's the second finding.
4. **Gen2 frequency and cause.** LOH? Promotion? Induced collections? (`grep -r "GC.Collect"` is a legitimate five-second review step.)
5. **LOH allocations.** Large buffers, `MemoryStream`, `ToArray()` on big collections, serialization buffers.
6. **Pinning.** Handle counts, POH usage, interop-heavy paths.
7. **Caches.** Bounded? Sized? Evicting? What's the turnover rate — mid-life crisis risk?
8. **Lifetimes.** DI registrations reviewed for captive dependencies; `IDisposable` honoured; event subscriptions balanced; timers disposed.
9. **Native dependencies.** Any managed wrapper over native memory? Is memory pressure reported?
10. **Configuration.** Server vs Workstation, DATAS, hard limit vs container limit, and *why* each non-default setting exists.
11. **Container sizing.** Request vs limit, CPU limit vs heap count, autoscaling signal.
12. **Deployment model.** JIT, R2R, or AOT — and whether cold start is in anyone's SLO.
13. **Observability.** Are GC counters exported and alerted on? Is there a way to take a gcdump in production?
14. **Runtime version.** On a supported LTS? What would upgrading buy?

---

## Concept 68 — When not to care

The final calibration, and it matters as much as everything above:

- **Most services never need Parts E–G.** An API serving 200 rps with a 500 ms budget on a 2 GB pod will never notice the GC. Saying so confidently is a senior signal; optimizing it anyway is not.
- **The defaults are excellent and improving every year.** The team tuning them has more data than you do.
- **Reach for this material when you have a symptom**: a latency tail you can attribute, a container being killed, a memory graph that only goes up, a density target you're missing, a cold-start budget you're blowing.
- **The knowledge is still worth having even when the tuning isn't.** Understanding boxing, lifetimes, retention, and LOH changes how you *write* code by default — which is cheaper than optimizing later — and it is what lets you diagnose the one incident a year where it matters.
- **In an interview, the meta-answer wins:** *"I'd want to know what problem we're solving before touching any of this. If we don't have a measurement showing GC in the top three, the right move is to leave it alone and go look at I/O."*

---

# Putting it together

## Worked example 1 — "Our p99 latency spikes every few seconds. Diagnose it."

Narrate in this order:

1. **Confirm it's GC before assuming it.** `dotnet-counters`: `% Time in GC`, gen0/1/2 counts, allocation rate, working set. If `% Time in GC` is 3% and pauses are sub-millisecond, this is not a GC problem — go look at thread pool queue length, dependency latency, and lock contention (Concepts 55, 60).
2. **If it is GC, classify.** Frequent short GCs = churn. Rare long GCs = large live set. Frequent gen2 = LOH or promotion. Pause ≫ GC duration = suspension (Concept 60).
3. **Suppose it's rare-and-long gen2, every ~4 seconds.** Check for LOH allocations first: `dotnet-gcdump`, look at LOH contents. Find `byte[]` of 200 KB–2 MB from response buffering (Concept 28).
4. **Trace the allocation source.** `dotnet-trace --profile gc-verbose`, PerfView "GC Heap Alloc Stacks" → `MemoryStream.EnsureCapacity` under the JSON serialization path.
5. **Fix in order of leverage.** Replace the buffering with `RecyclableMemoryStream` (or stream the response directly with `PipeWriter` / `Utf8JsonWriter`), which removes the LOH allocations and therefore the forced gen2s (Concepts 48, 49).
6. **Verify with the same metrics.** Gen2 count per minute, p99, allocation rate. State the expected direction *before* measuring.
7. **Prevent regression.** Add gen2 count and `% Time in GC` to the dashboard; add an allocation-ceiling assertion on the serialization path in CI (Concept 61).
8. **Name the systemic lesson.** The bug wasn't the buffer; it was that no one had a number for allocation rate on the hot path.

## Worked example 2 — "Design the memory strategy for a high-throughput ingestion service."

100k events/sec, ~2 KB each, parse → validate → batch → write to storage.

1. **Do the arithmetic first.** 200 MB/s of raw payload. Naively materializing each event as objects is 200 MB/s of allocation with a high survival rate through the batching window — this is a GC design problem, not an afterthought (Concept 46).
2. **Never materialize what you don't need.** Parse from `ReadOnlySequence<byte>` with `System.IO.Pipelines`; extract only the fields used for routing; keep the payload as bytes (Concept 49).
3. **Pool everything per-event.** `ArrayPool<byte>` for buffers, with strict `try/finally` return discipline; POH-allocated buffers for the network layer (Concepts 43, 48).
4. **Make the batch the only thing that survives.** Batches live for the flush interval — a mid-life-crisis risk. Keep the batch window short and the batch representation flat (`struct` arrays, not object graphs), so the surviving set is a few large objects rather than millions of small ones (Concepts 47, 52).
5. **Watch the LOH.** A 4 MB batch buffer is a gen2 allocation; pool the batch buffers rather than allocating per flush (Concept 28).
6. **GC configuration.** Server GC with DATAS; measure with DATAS off too, because this is exactly the sustained-high-throughput case where it can cost (Concept 33).
7. **Backpressure is a memory control.** Bounded channels (Module 11) cap in-flight memory; without a bound, a downstream slowdown becomes an OOM. This is the Module 13 queue-backlog lesson expressed in bytes.
8. **Size the container.** Heap hard limit below the container limit with headroom for pooled buffers and native TLS; request ≈ limit; scale on queue depth, not memory (Concepts 36, 63).
9. **Instrument.** Allocation rate, gen2 count, pool rent/return balance, batch flush latency, pause time.
10. **State what you're not doing.** No custom GC configuration until measured; no span-based rewriting outside the parser; no `GC.Collect` between batches.

## Worked example 3 — "Struct or class? Walk me through this type."

Given `class PriceTick { string Symbol; decimal Bid; decimal Ask; DateTime Ts; }`, stored 50 million at a time in memory for backtesting:

1. **Count the objects.** As a class: 50M objects × (16 header + 8 `string` ref + 16 + 16 + 8) ≈ 64 bytes each → ~3.2 GB, plus 50M references in the array, plus 50M references to trace on every gen2. This is a GC problem before it's a memory problem (Concepts 3, 12).
2. **Make it a `readonly record struct`.** Now the array is *one* object. 50M × (8 ref + 16 + 16 + 8 = 48 bytes) = 2.4 GB in a single allocation — which lands on the LOH, fine, it's one object and never moves (Concepts 28, 47).
3. **Kill the reference field.** `string Symbol` means the array still contains 50M references to trace. Replace with a `short SymbolId` into a lookup table → the struct becomes reference-free, the array becomes *invisible* to the tracer, and it shrinks to 40 bytes → 2.0 GB with zero tracing cost (Concepts 12, 26).
4. **Order the fields** largest first to avoid padding; verify with `Unsafe.SizeOf<PriceTick>()` (Concept 13).
5. **Implement equality properly** — `readonly record struct` does it, avoiding `ValueType.Equals` reflection and boxing (Concept 17).
6. **Pass it by `in`** where consumed in hot loops, which is safe because it's `readonly` (Concept 16).
7. **State the trade-offs honestly.** No identity, no polymorphism, copy-on-assign, and the struct is now 40 bytes — too big for casual by-value passing. If the access pattern were "a few at a time, polymorphic, mutable," the class would win.
8. **Mention the next step if it still doesn't fit:** structure-of-arrays (`decimal[] bids; decimal[] asks;`) for vectorization and cache locality, or memory-mapped files to move it out of the heap entirely.

## Worked example 4 — "Our pod is OOMKilled at 2 GB but `dotnet-counters` shows a 600 MB heap."

1. **Name the distinction immediately.** OOMKilled is the kernel measuring RSS against the cgroup limit; the GC heap is only one contributor (Concept 36).
2. **Account for the difference systematically:** GC heap (600 MB) + committed-but-free heap (fragmentation/headroom — check `GC Committed Bytes` vs heap size) + thread stacks (count threads × ~1 MB) + loader heaps (dynamic code?) + native allocations (which libraries?) + the runtime itself.
3. **Check the heap hard limit.** If it's the 75% default of 2 GB = 1.5 GB, the GC is allowed to grow to 1.5 GB before it panics, leaving only 500 MB for everything else. If native usage is significant, lower `GCHeapHardLimitPercent` (Concept 36).
4. **Look for native memory.** Image processing, compression, ML inference, native DB drivers, `MemoryMappedFile`. Check whether wrappers report `GC.AddMemoryPressure` (Concept 44).
5. **Check thread count.** A blocked-thread-pool service with 400 threads is 400 MB of stacks before anything else (Module 15).
6. **Check for dynamic code.** RSS growth with a flat heap, plus a growing count of `DynamicMethod`/generated assemblies → loader heaps (Concept 8).
7. **Fix, then verify with the right metric** — working set, not heap size — under sustained load, not at startup.

---

## Common questions and what a strong answer contains

**"Stack vs heap?"** Don't answer with the storage taxonomy. Answer: value vs reference semantics is the type-level distinction; *storage location is decided by the container* — fields, captures, boxes, and arrays put value types on the heap; `ref struct` is the one type that's stack-only by construction (Concepts 11–12).

**"When would you use a struct?"** Small, immutable, value-semantic, not boxed, with a measured reason — and the killer application is large arrays where a struct array is one traceable object instead of n+1 (Concepts 47, 12).

**"What is boxing and when does it happen?"** Allocation + copy; then list the non-obvious triggers — interfaces on structs, `params object[]`, `foreach` over `IEnumerable<T>`, default `ValueType.Equals` — and mention that .NET 9/10 stack-allocate some boxes now (Concepts 14, 53).

**"How does the GC work?"** Tracing from roots, generational, compacting; allocation is a pointer bump; cost is proportional to survivors; gen0/1/2 with dynamic budgets; write barriers and the card table make ephemeral collections cheap; LOH is separate and swept (Concepts 21–28).

**"Why are there generations?"** The generational hypothesis plus the cost model: collecting the nursery touches only recent objects, so it's cheap, and the card table lets you skip the old generation safely (Concepts 25–26).

**"Server vs Workstation GC?"** One heap and GC thread per core vs one heap on the allocating thread; throughput vs footprint; ASP.NET Core defaults to Server; and since .NET 9 DATAS changes the footprint argument (Concepts 30, 33).

**"What's the Large Object Heap?"** ≥85,000 bytes, born in gen2, swept not compacted, causes fragmentation and forces full GCs; fix with pooling and `RecyclableMemoryStream` (Concept 28).

**"What's new in .NET memory recently?"** Regions replaced segments (.NET 7); DATAS default (.NET 9); escape analysis and stack allocation of small arrays, delegates, and enumerators plus Arm64 write barrier improvements (.NET 10); conditional escape analysis extensions and runtime async (.NET 11) (Concepts 27, 33, 53).

**"Does `Dispose` free memory?"** No. It releases non-memory resources deterministically. Memory is reclaimed when the object is unreachable and the GC gets to it (Concept 39).

**"Finalizer vs `IDisposable`?"** Finalizers are non-deterministic, single-threaded, cost an extra GC cycle and a promotion, may not run at all on shutdown; `IDisposable` is deterministic. Write `IDisposable`; use `SafeHandle` instead of a finalizer (Concepts 38, 39, 41).

**"Can you have a memory leak in .NET?"** Yes — unreachable is the criterion, not unused. Then list four or five from the taxonomy with the retention mechanism for each (Concept 57).

**"How would you debug growing memory in production?"** Counters to classify the shape, two gcdumps under load to find what's growing, a dump plus `gcroot` to find who's holding it — and diff, always (Concepts 56, 58, 59).

**"What causes GC pauses and how do you reduce them?"** Separate too-often from too-slow from too-slow-to-suspend; then reduce the live set, eliminate LOH churn, remove induced collections, keep background GC on, and consider smaller heaps across more instances (Concepts 60, 62).

**"`Span<T>` — what is it and what can't it do?"** A length plus a managed reference; a `ref struct`, therefore stack-only: no boxing, no class fields, no `await`, no lambdas, no iterators — because all of those would put the reference on the heap (Concept 19).

**"What does `readonly struct` buy?"** Correctness plus the elimination of defensive copies when passed by `in` — and note that a non-readonly struct with `in` is *worse* than by value (Concept 16).

**"Native AOT — when?"** Startup and footprint decision: serverless, CLI, sidecars, density. Not for reflection-heavy or plugin-based apps. R2R is the middle option people forget (Concepts 54, 64).

**"Why did memory go up when we moved to bigger nodes?"** Server GC heap count follows core count; budgets follow available memory. DATAS (.NET 9+) largely fixes it; before that, pin `GCHeapCount` or use Workstation GC (Concepts 30, 33, 63).

**"Container OOMKilled with a small heap — what's going on?"** RSS ≠ GC heap. Enumerate: committed headroom, stacks, loader heaps, native memory, and the 75% hard-limit margin (Concepts 36, 44).

**"Should we autoscale on memory?"** Usually no for .NET — the GC grows into available memory, so memory is a sticky, partly self-inflicted signal. Scale on CPU, RPS, or queue depth (Concept 63).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "Value types go on the stack" | Explains that the container decides; names fields, captures, boxes, arrays |
| "Structs are faster" | Gives the size threshold, copy costs, and the array-of-structs argument, then says "measure" |
| "The GC handles memory, so there are no leaks" | Defines a leak as unwanted reachability and lists the retention mechanisms |
| "`Dispose` frees memory" | Distinguishes resource release from memory reclamation |
| Writes finalizers for managed resources | Uses `IDisposable` + `SafeHandle`, and can explain the f-reachable queue cost |
| "We call `GC.Collect()` to keep memory low" | Explains that it forces a blocking gen2 and defeats tuning; removes it |
| Recites "gen0, gen1, gen2" with no cost model | Says "you pay for survivors, not garbage" and explains the card table |
| Unaware of the LOH | Knows 85,000 bytes, gen2-at-birth, sweep-not-compact, and the pooling fix |
| Thinks background GC means no pauses | Knows gen0/1 are always blocking and BGC takes two short pauses |
| Treats pause time as GC time | Separates suspension from collection and knows how to measure both |
| "Server GC is better" | Frames it as throughput vs footprint, mentions heap-per-core and DATAS |
| Unaware of DATAS | Knows it's the .NET 9+ default, what it adapts, and when to turn it off |
| Still quotes pre-.NET 7 segment tuning advice | Knows regions replaced segments and that old tuning posts may mislead |
| "Avoid LINQ, it allocates" | Knows what .NET 9/10/11 escape analysis now removes, and says "measure on our runtime" |
| Optimizes allocation everywhere | Checks `% Time in GC` first and optimizes the hot 1% |
| Adds pooling reflexively | Knows pooling moves objects to gen2 and creates use-after-free risk |
| Uses `in` on every struct parameter | Knows about defensive copies on non-readonly structs |
| Uses structs as dictionary keys without `IEquatable<T>` | Knows `ValueType.Equals` boxes and may use reflection |
| `string.Intern` for deduplication | Knows interned strings live for the ALC's lifetime |
| Conflates OOMKilled with `OutOfMemoryException` | Accounts for RSS = heap + stacks + loader heaps + native + headroom |
| Autoscales on memory | Scales on CPU/RPS/queue depth and explains why memory is a poor signal |
| Doesn't know the diagnostic tools | Narrates counters → gcdump → dump + `gcroot`, and insists on diffs under load |
| Guesses at cache memory cost | Does the object-layout arithmetic aloud, including the 16-byte tax |
| Ignores the runtime version | Treats a runtime upgrade as a legitimate performance strategy with evidence |
| Optimizes without a symptom | Says clearly when the right answer is "leave it alone" |

---

## Practice exercises

**Exercise 1 — Prove the boxing (30 min).** Write a benchmark with `[MemoryDiagnoser]` comparing: `foreach` over `List<int>`, over the same list typed as `IEnumerable<int>`, and over an `int[]` typed as `IEnumerable<int>`. Run it against .NET 8 and .NET 10 with `[SimpleJob]` for both. Explain every difference using Concepts 14, 15, and 53. **This is the highest-value exercise in the module** — it turns "the JIT got better" into numbers you measured.

**Exercise 2 — Object layout arithmetic.** Compute by hand the memory used by `Dictionary<string, Customer>` with 100,000 entries, where `Customer` has 6 reference fields and 4 value-type fields. Then measure it with `dotnet-gcdump` and `objsize`. Reconcile the two numbers. Most people are off by 2–3×.

**Exercise 3 — Build a leak, then find it.** Write an ASP.NET Core app with a static `ConcurrentDictionary` keyed by request ID that never evicts. Load it. Take two `dotnet-gcdump` snapshots, diff them, then take a `dotnet-dump` and walk `dumpheap -stat` → `dumpheap -mt` → `gcroot` to the static. Repeat for an event-handler leak and a captive-dependency leak. Write down the *signature* of each in the tooling.

**Exercise 4 — LOH fragmentation.** Allocate a repeating pattern of 100 KB and 1 MB arrays, releasing the large ones, until an allocation fails or committed memory far exceeds live. Observe with `eeheap -gc` and the fragmentation counter. Then fix it with `ArrayPool` and with `LargeObjectHeapCompactionMode.CompactOnce`, and compare the cost of each.

**Exercise 5 — Server vs Workstation vs DATAS.** Run the same load test against one service with four configurations: Workstation, Server with DATAS off, Server with DATAS on, and Server with `GCHeapCount=2`. Record p50/p99, RSS, allocation rate, gen2 count, and throughput. Do it on a 4-core and a 16-core machine. Write a one-page recommendation with numbers. This is close to a real architect task.

**Exercise 6 — Defensive copies.** Benchmark a 48-byte struct passed by value, by `in` (non-readonly), and by `in` (readonly), each calling a member twice. Explain the ranking. Then look at the disassembly with `[DisassemblyDiagnoser]` and find the copies.

**Exercise 7 — The premature-collection bug.** Write a class holding an `IntPtr` from `NativeMemory.Alloc` with a finalizer that frees it, and a method that uses the pointer without touching `this` afterward. Run in Release with aggressive GC and reproduce the use-after-free. Fix it twice: once with `GC.KeepAlive`, once with `SafeHandle`. This bug is much more convincing once you've caused it.

**Exercise 8 — Async allocation profile.** Benchmark an async method whose awaits always complete synchronously vs one that always suspends, returning `Task<T>` and `ValueTask<T>`. Four combinations, allocation column on. Explain each number with Concept 51.

**Exercise 9 — The container budget.** Run a service in a container with a 512 MB limit. Find the heap hard limit the runtime chose (`GCMemoryInfo.TotalAvailableMemoryBytes` / `HighMemoryLoadThresholdBytes`). Drive it to `OutOfMemoryException`. Then add a native allocation of 200 MB and drive it to OOMKilled instead. Document the difference in what you observe from inside and outside the process.

**Exercise 10 — The memory review write-up (one page).** For a real service you know: live set and its top three categories, working-set-to-heap gap and its explanation, allocation rate and `% Time in GC` at peak, gen2 frequency and cause, LOH sources, cache bounds and turnover, DI lifetime audit results, non-default GC settings with justifications, container sizing, deployment model, and the three changes with the best ratio of memory saved to effort. This is very close to a real architect take-home.

---
## Free resources

### Primary sources — the runtime's own design documentation

| Resource | What it covers | Why read it |
|---|---|---|
| [Book of the Runtime (BOTR)](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/README.md) | The CLR's internal design, written by the people who built it | **The single best free resource in this module.** Start here |
| [BOTR — Garbage Collection Design](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/garbage-collection.md) | Allocation, generations, card tables, plan/relocate/compact, concurrency | The authoritative version of Part C |
| [BOTR — Type Loader](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/type-loader.md) | MethodTables, layout, generics instantiation | Concepts 3 and 5 from the source |
| [BOTR — Method Descriptor](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/method-descriptor.md) | MethodDesc, prestubs, how a call becomes machine code | Concept 2 |
| [BOTR — Virtual Stub Dispatch](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/virtual-stub-dispatch.md) | How interface dispatch actually works | Concept 4 |
| [BOTR — RyuJIT Overview](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/ryujit-overview.md) | The JIT's phases and IR | Background for Concepts 6 and 53 |
| [dotnet/runtime — design/features](https://github.com/dotnet/runtime/tree/main/docs/design/features) | Design documents for regions, DATAS, and other GC features | Where new GC work is documented before it ships |
| [The GC source itself (`gc.cpp`)](https://github.com/dotnet/runtime/blob/main/src/coreclr/gc/gc.cpp) | ~50k lines of the actual collector | Not light reading, but searchable and definitive |
| [ECMA-335 — Common Language Infrastructure](https://www.ecma-international.org/publications-and-standards/standards/ecma-335/) | The IL, metadata, and type-system specification | For the "what is guaranteed vs what is an implementation detail" question |

### Microsoft Learn — GC and memory

| Resource | What it covers |
|---|---|
| [Garbage collection in .NET](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/) · [Fundamentals](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals) | The canonical overview: generations, phases, heaps |
| [Workstation and server GC](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/workstation-server-gc) | Concept 30, from the source |
| [Background garbage collection](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/background-gc) | What is and isn't concurrent (Concept 31) |
| [Dynamic adaptation to application sizes (DATAS)](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/datas) | **Read this one** — Concept 33, including the tuning knobs |
| [The large object heap](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/large-object-heap) | The 85,000-byte threshold, sweeping, compaction (Concept 28) |
| [Latency modes](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/latency) · [Induced collections](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/induced) | Concept 34, and why `GC.Collect` is usually wrong |
| [Garbage collector config settings](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/garbage-collector) | **The complete knob list** for Concepts 35–36 |
| [Cleaning up unmanaged resources](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged) · [Implementing Dispose](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose) · [Implementing DisposeAsync](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-disposeasync) | Concepts 38–40 |
| [Weak references](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/weak-references) | Concept 42 |
| [Memory and garbage collection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/memory) | The server-workload version, with a demo app that shows each pathology |
| [`GCSettings`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.gcsettings) · [`GC.GetGCMemoryInfo`](https://learn.microsoft.com/en-us/dotnet/api/system.gc.getgcmemoryinfo) · [`GCMemoryInfo`](https://learn.microsoft.com/en-us/dotnet/api/system.gcmemoryinfo) | The programmatic surface for Concept 37 |
| [`GCHandle`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.gchandle) · [`SafeHandle`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.safehandle) · [`ConditionalWeakTable`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.conditionalweaktable-2) | Concepts 41–43 |

### Microsoft Learn — runtime, JIT, and deployment

| Resource | What it covers |
|---|---|
| [What's new in the .NET 10 runtime](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10/runtime) | **Current-state reading for Concept 53** — stack allocation, escape analysis, write barriers, inlining |
| [What's new in the .NET 9 runtime](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-9/runtime) | Where DATAS-by-default and the first escape analysis landed |
| [Tiered compilation settings](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/compilation) | Concept 6's configuration surface |
| [Native AOT deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/) · [ReadyToRun](https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run) | Concepts 7, 54, 64 |
| [Trimming options and warnings](https://learn.microsoft.com/en-us/dotnet/core/deploying/trimming/trimming-options) | The practical cost of going AOT |
| [`System.Runtime.InteropServices` source generation (`LibraryImport`)](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke-source-generation) | Concept 45's modern interop path |

### Microsoft Learn — diagnostics

| Resource | What it covers |
|---|---|
| [.NET diagnostic tools overview](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/) | The whole toolbox |
| [Tutorial: Debug a memory leak](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/debug-memory-leak) | **Do this one hands-on** — it is Concept 58 as a guided exercise |
| [`dotnet-counters`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters) · [`dotnet-gcdump`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-gcdump) · [`dotnet-dump`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dump) · [`dotnet-trace`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) | The four tools of the workflow |
| [SOS debugging extension](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/sos-debugging-extension) · [`dotnet-sos`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-sos) | The command reference for Concept 59 |
| [`dotnet-monitor`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-monitor) | Collecting dumps and traces from containers without shelling in |
| [Well-known EventCounters](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/available-counters) | Exactly which counters exist and what they mean (Concept 37) |
| [Performance code-analysis rules](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/performance-warnings) | The CA rules that catch several Part E mistakes at build time |

### Language and type-system references

| Resource | What it covers |
|---|---|
| [Write safe and efficient C# code](https://learn.microsoft.com/en-us/dotnet/csharp/write-safe-efficient-code) | **The official version of Concepts 16 and 19** — `readonly struct`, `in`, `ref` returns, spans |
| [Choosing between class and struct](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/choosing-between-class-and-struct) | The framework design guidelines behind Concept 47 |
| [`ref struct` types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct) · [`ref` fields and `scoped`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/declarations) | Concepts 19–20 |
| [Memory and spans](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/) · [`Memory<T>` usage guidelines](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines) | **The ownership/lifetime contract** most people skip |
| [`ArrayPool<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1) · [`MemoryPool<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.memorypool-1) | Concept 48 |
| [Boxing and unboxing](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing) | The language-level version of Concept 14 |
| [High-performance logging with `LoggerMessage`](https://learn.microsoft.com/en-us/dotnet/core/extensions/logger-message-generator) | The single highest-value allocation fix in most services (Concept 50) |

### Deep dives, blogs, and articles

| Resource | What it covers |
|---|---|
| [.NET Memory Performance Analysis](https://github.com/Maoni0/mem-doc/blob/master/doc/.NETMemoryPerformanceAnalysis.md) — Maoni Stephens | **The best single document on diagnosing .NET memory**, written by the GC architect. Long; worth every page |
| [Maoni0/mem-doc repository](https://github.com/Maoni0/mem-doc) | Supporting material, talks, and slides |
| [Preparing for the .NET 10 GC (DATAS)](https://devblogs.microsoft.com/dotnet/preparing-for-dotnet-10-gc/) — .NET Blog | Why Server GC and DATAS differ, with benchmark data (Concept 33) |
| [Performance Improvements in .NET 10](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/) — Stephen Toub | **The deabstraction and escape-analysis material in Concept 53**, with disassembly |
| [Performance Improvements in .NET 11](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-11/) — Stephen Toub | The current edition: conditional escape analysis, runtime async, delegate layout |
| [Performance Improvements in .NET 8](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/) — Stephen Toub | Dynamic PGO by default, and the best explanation of guarded devirtualization |
| [.NET Blog — Maoni Stephens' posts](https://devblogs.microsoft.com/dotnet/author/maoni) | GC design decisions explained by the person who makes them |
| [tooslowexception.com](https://tooslowexception.com/) — Konrad Kokosa | Deep .NET memory articles; the companion blog to *Pro .NET Memory Management* |
| [prodotnetmemory.com](https://prodotnetmemory.com/) — Konrad Kokosa | Includes the free **.NET memory management poster** — print it and put it on the wall |
| [adamsitnik.com](https://adamsitnik.com/) — Adam Sitnik | Excellent posts on [`ArrayPool`](https://adamsitnik.com/Array-Pool/), [`Span<T>`](https://adamsitnik.com/Span/), and [value vs reference types](https://adamsitnik.com/Value-Types-vs-Reference-Types/) |
| [mattwarren.org](https://mattwarren.org/) — Matt Warren | "How the CLR works" series; object layout, JIT internals, and GC internals explained accessibly |
| [Dotnetos](https://dotnetos.org/) | .NET performance courses, articles, and conference talks; much of it free |
| [Microsoft.IO.RecyclableMemoryStream](https://github.com/microsoft/Microsoft.IO.RecyclableMemoryStream) | The README is a good short essay on LOH fragmentation (Concept 48) |
| [dotnet/runtime — Performance guidelines for contributors](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/) | How the BCL team itself reasons about allocation |

### Tools

| Tool | What it's for |
|---|---|
| [BenchmarkDotNet](https://benchmarkdotnet.org/) · [repo](https://github.com/dotnet/BenchmarkDotNet) | `[MemoryDiagnoser]`, `[DisassemblyDiagnoser]`, multi-runtime jobs (Concept 61) |
| [PerfView](https://github.com/microsoft/perfview) | GCStats, allocation stacks, suspension vs collection time. Windows-first but reads Linux traces |
| [dotnet/diagnostics](https://github.com/dotnet/diagnostics) | Source and docs for the `dotnet-*` CLI tools and SOS |
| [sharplab.io](https://sharplab.io/) | See the IL and JIT assembly for a snippet instantly — the fastest way to prove a boxing claim |
| [Speedscope](https://www.speedscope.app/) | Flame graphs from `dotnet-trace` output |
| [roslyn-analyzers](https://github.com/dotnet/roslyn-analyzers) | Includes the performance-sensitive analyzers that flag hidden allocations |
| [`GC.GetAllocatedBytesForCurrentThread`](https://learn.microsoft.com/en-us/dotnet/api/system.gc.getallocatedbytesforcurrentthread) | Allocation assertions in unit tests, no tooling required |

### Videos and talks

| Resource | What it covers |
|---|---|
| [The .NET YouTube channel](https://www.youtube.com/@dotnet) | The **Deep .NET** series with Stephen Toub covers spans, strings, LINQ internals, and async allocation in depth |
| [.NET Conf archives](https://www.dotnetconf.net/) | Annual performance sessions; the .NET 9/10/11 performance talks are the video version of Concept 53 |
| [Channel 9 / Learn: On .NET GC episodes](https://learn.microsoft.com/en-us/shows/on-dotnet/) | Interviews with the GC team |
| [Dotnetos conference talks on YouTube](https://www.youtube.com/@Dotnetos) | Konrad Kokosa and others on memory internals |

### Books worth owning (not free, listed for completeness)

- **Pro .NET Memory Management, 2nd edition** — Konrad Kokosa, Christophe Nasarre, Kevin Gosse (Apress, 2024). The definitive treatment; everything in Part C and D at ten times the depth, with troubleshooting scenarios.
- **Writing High-Performance .NET Code, 2nd edition** — Ben Watson. More operational, less internals; strong on measurement discipline.
- **Pro .NET Benchmarking** — Andrey Akinshin (the author of BenchmarkDotNet). The measurement half of Concept 61 done properly.
- **CLR via C#** — Jeffrey Richter. Dated in specifics (it predates .NET Core), still excellent on the type system and lifetime model.

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| Managed execution | Precise reference tracking → the GC can move objects → bump allocation; pinning takes that away |
| Object cost | 16 bytes overhead, 24-byte minimum on x64; arrays 24 + elements; strings 22 + 2n |
| Method dispatch | Direct / vtable / virtual stub dispatch; devirtualization unlocks inlining unlocks everything |
| Generics | One shared body for reference types; one specialized body per value type |
| Tiered compilation | Tier-0 for startup, tier-1 for throughput, OSR for loops, dynamic PGO by default since .NET 8 |
| Deployment models | JIT = best throughput; R2R = fast start, no restrictions; AOT = fastest start, closed world |
| Stack | ~1 MB, reserved not committed, overflow kills the process, `async` moves locals to the heap |
| Value vs reference | Copy semantics and identity — not storage location |
| Where things live | The container decides: fields, captures, boxes, arrays are all heap |
| Boxing | Allocation + copy; interfaces on structs, `params object[]`, `IEnumerable<T>` enumerators, `ValueType.Equals` |
| `readonly struct` / `in` | Non-readonly struct + `in` = a defensive copy per member access — worse than by value |
| Struct equality | Implement `IEquatable<T>` or use `readonly record struct`; the default boxes |
| Spans | Length + managed reference; `ref struct` → no heap, no `await`, no lambdas, no generics (pre-`allows ref struct`) |
| GC premise | Generational hypothesis + tracing; **you pay for survivors, not garbage** |
| Allocation | Per-thread allocation context, pointer bump; the slow path is where GCs are triggered |
| Roots | Stack/registers (exact maps), statics, handles, finalizer queue — locals die at last *use*, not last brace |
| Generations | 0 = nursery, 1 = buffer, 2 = everything old + LOH + POH; gen N collects 0..N |
| Write barrier | Records old→young writes in the card table (~256-byte granularity) so gen0 skips gen2 |
| Regions | .NET 7 replaced segments with 4 MB regions; enables returning memory and DATAS |
| LOH | ≥85,000 bytes, gen2 at birth, swept not compacted; forces full GCs and fragments |
| POH | `GC.AllocateArray<T>(n, pinned: true)`; the right home for long-lived pinned I/O buffers |
| Server vs Workstation | One heap + GC thread per core vs one heap on the allocating thread; throughput vs footprint |
| Background GC | Gen2 only, two short pauses; gen0/gen1 always stop the world |
| Suspension | Safe points + hijacking; pause = suspend time + collect time, and they're measured separately |
| DATAS | Default since .NET 9 for Server GC; heap count adapts to load, heap size tracks live data |
| Latency tools | `SustainedLowLatency`, `TryStartNoGCRegion`, `ConserveMemory` — narrow, measured, documented |
| Containers | Heap hard limit = 75% of the cgroup limit; heap count from CPU quota; `GC.RefreshMemoryLimit` |
| OOMKilled vs OOM exception | Kernel vs GC; RSS = heap + committed headroom + stacks + loader heaps + native |
| Key metrics | Allocation rate, `% Time in GC`, gen2 count, committed vs live, pause durations, fragmentation |
| Finalizers | Extra GC cycle, promotion, one thread, may never run; use `IDisposable` + `SafeHandle` |
| `Dispose` | Releases non-memory resources deterministically; does **not** free memory |
| `SafeHandle` | Ref-counted across P/Invoke, critical finalizer; fixes the premature-collection race |
| Weak refs | `WeakReference<T>`, `ConditionalWeakTable` (ephemerons), `DependentHandle` |
| Pinning | Very short (`fixed`) or permanent and few (POH); the middle is a fragmentation engine |
| Native memory | Invisible to the GC — report it with `GC.AddMemoryPressure` |
| Allocation cost | Cheap to allocate, expensive to survive; ≥85K expensive immediately; pinned expensive structurally |
| Struct vs class | Small, immutable, value-semantic, unboxed, measured — and arrays of structs are one traced object |
| Pooling | `ArrayPool` rents are ≥ requested, must be returned, never touched after return; pools live in gen2 |
| Zero-alloc toolkit | `stackalloc`+pool fallback, `string.Create`, `u8` literals, `SearchValues`, `TryFormat`, `CollectionsMarshal` |
| Hidden allocations | Closures, LINQ, iterators, `params`, boxed enumerators, logging, exceptions, `ToList/ToArray` |
| Async allocations | State machine boxes only on actual suspension; `ValueTask` for usually-synchronous paths |
| Mid-life crisis | Die young or live long; medium-lifetime caches are the expensive case |
| Deabstraction | .NET 9 boxes → .NET 10 small arrays, struct fields, delegates, array enumerators → .NET 11 field enumerators |
| Four pathologies | Leak (grows, never returns), bloat (flat high), churn (high GC CPU), fragmentation (committed ≫ live) |
| Leak taxonomy | Statics, events, closures, timers, captive dependencies, unbounded caches, interning, handles, ALCs, native |
| Workflow | `dotnet-counters` → two `dotnet-gcdump`s under load → `dotnet-dump` + `gcroot`; always diff |
| SOS | `dumpheap -stat`, `dumpheap -mt`, `gcroot`, `objsize`, `eeheap -gc`, `finalizequeue`, `gchandles` |
| Pause diagnosis | Too often (allocation) vs too slow (live set) vs too slow to suspend (safe points/threads) |
| Latency budget | Pause frequency × duration × overlap probability; gen2 is what lands on p99.9 |
| Sizing | Heap count follows CPU, hard limit follows the container; request ≈ limit; don't autoscale on memory |
| Native AOT decision | Startup and density, not throughput; R2R is the middle option people forget |
| Runtime currency | .NET 10 is LTS (Nov 2025); .NET 11 RC1 Sept 2026, GA 10 Nov 2026; upgrading is a performance strategy |
| When not to care | Check `% Time in GC` first; if it's not top-three, the right answer is to leave it alone |

---

## Progress

Module 14 complete — **Phase 4 (.NET & C# Technical Mastery) is now open.** Phase 3 gave you the distributed-systems vocabulary; this module starts the descent into the runtime that actually executes your designs.

This module closes several loops from earlier phases:

- **Module 6's Little's Law** now has a memory expression: concurrency × per-request allocation = the heap you must size for, and a queue with no bound is an OOM with extra steps.
- **Module 10's caching trade-offs** now include a GC cost — an in-process cache is a promotion engine, which is a real argument for moving state to Redis that has nothing to do with hit rate.
- **Module 13's cascading failures** now have a GC mechanism: pauses raise latency, latency raises concurrency, concurrency raises allocation, and the loop can go metastable. "It was a GC death spiral" is a specific, diagnosable claim.
- **Module 12's storage arithmetic** and this module's object-layout arithmetic are the same skill applied at different layers.

Threads left open on purpose:

- **Async internals** — `Task` machinery, `SynchronizationContext`, ThreadPool starvation, `ValueTask` consumption rules, `Channel<T>`, and runtime async in depth are **Module 15**. Concept 51 is the memory-shaped summary.
- **The .NET memory model** — `volatile`, `Interlocked`, barriers, and what the JIT is allowed to reorder — belongs with concurrency in **Module 15**.
- **Measurement discipline** — BenchmarkDotNet methodology, statistical rigour, `Span<T>`-based rewriting, Native AOT in practice, and JIT-level tuning are **Module 17**. Concepts 46–55 and 61 are the foundation it builds on.
- **DI lifetimes and the captive-dependency problem** named in Concept 57 are diagnosed properly in **Module 18**.
- **EF Core change tracking as a retention mechanism** — the single most common cause of bloat in .NET line-of-business services — is **Module 19**.
- **`readonly record struct` as a DDD value object** is **Module 22**.
- **Observability for these counters** — exporting GC metrics through OpenTelemetry and alerting on them — is **Module 28**.

Next in the curriculum: **Module 15 — Async/await & concurrency** (`Task` internals, `SynchronizationContext`, ThreadPool starvation, `Channel<T>`, `async void` and the deadlock traps), which takes the state machines from Concept 51 and the suspension model from Concept 32 and builds the concurrency half of .NET mastery on top of them.
