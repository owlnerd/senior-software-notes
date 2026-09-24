# Module 15 — Async/await & Concurrency
*Phase 4: .NET & C# Technical Mastery · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **`async` is not about doing more things at once — it is about not paying for a thread while nothing is happening, and every bug, deadlock, and outage in this module comes from someone accidentally paying for that thread anyway.**

That reframing matters because the naive picture of `async/await` is *"it makes things run in parallel"* or *"it makes things faster."* Neither is true. A single `await` on a single request, in isolation, is **slower** than the synchronous equivalent — you pay a state machine, a possible allocation, and a scheduling hop. What you buy is that during the 40 ms your database takes to answer, the thread that asked the question is serving other requests instead of sitting in a kernel wait. Async is a **throughput and density** optimization purchased with **latency and complexity**. A mid-level candidate can say "async frees up the thread." A senior candidate can tell you exactly which thread, which pool it came from, what happens when that pool runs out, why the fix for a 30-second p99 was `ThreadPoolMinThreads` in one case and deleting a `.Result` in another, and when the right answer was to write synchronous code on a dedicated thread.

This module is the second half of the .NET runtime foundation. Module 14 gave you the memory model of execution — where objects live and what the collector charges you. This module gives you the **execution model of time**: what a thread costs, what the runtime does while nothing is happening, and how work gets scheduled onto cores. The two meet in several places, all of them load-bearing: Concept 51 of Module 14 (async state machines as heap allocations) is Concept 13 here; Module 14's suspension model (safe points, time-to-suspend) is why a thread that never yields is a GC problem as well as a throughput problem; and the cascading-failure loop from Module 13 has a concurrency expression here that is at least as common as the GC one.

This module has nine jobs:

1. **Separate three ideas that English conflates.** Concurrency, parallelism, and asynchrony are orthogonal. You can have concurrency without parallelism (one core, many in-flight operations), parallelism without asynchrony (`Parallel.For` on eight cores, all blocking), and asynchrony without either (one request, one `await`). Almost every confused async answer in an interview is a failure to separate these.
2. **Make the machinery mechanical, not magical.** What `Task` actually is, what the compiler generates, where the state machine lives, when it allocates, what `await` compiles to, and what changes in .NET 11 when the runtime takes the state machine over.
3. **Explain scheduling honestly.** `SynchronizationContext`, `TaskScheduler`, `ExecutionContext`, the ThreadPool's two queues and its work-stealing, and the thread-injection heuristic that turns a small blocking mistake into a 30-second outage.
4. **Teach the failure modes with their mechanisms.** ThreadPool starvation, sync-over-async deadlock, `async void`, unobserved exceptions, the `ValueTask` misuse rules, unbounded fan-out, and the `CancellationTokenSource` leak.
5. **Cover the memory model properly.** `volatile`, `Interlocked`, acquire/release, tearing, false sharing, and why a bug that never reproduces on your x64 laptop reproduces immediately on an Arm64 node. Almost nobody prepares this, and it is the single highest-signal area in a deep technical round.
6. **Give you the primitive catalogue with a decision procedure** — `lock`, `System.Threading.Lock`, `SemaphoreSlim`, `ReaderWriterLockSlim`, `SpinLock`, the concurrent collections, `Channel<T>`, `Parallel.ForEachAsync` — organized by the question they answer rather than by namespace.
7. **Cover the current platform state honestly.** `System.Threading.Lock` is new in .NET 9. `Task.WhenEach` is .NET 9. `ConfigureAwaitOptions` is .NET 8. `Parallel.ForEachAsync` is .NET 6. Runtime async is the headline runtime feature of .NET 11 (RC1 as of September 2026, GA scheduled 10 November 2026) and is a preview feature you opt into, while the BCL itself already ships compiled with it. The green-threads experiment was run and **rejected**, and knowing why is a strong signal.
8. **Make you diagnostically dangerous.** A starvation signature, a deadlock signature, and the `dotnet-counters → dotnet-stack → dotnet-dump + dumpasync` workflow narrated end to end.
9. **Raise it to architecture.** Concurrency limits as a design artifact, backpressure end to end, the async boundary in library design, graceful shutdown, and the cases where the correct architectural answer is "don't make this async."

Nine framings to carry through:

1. **Async is about threads, not speed.** The unit of saving is a blocked thread. If nothing would have blocked, async buys you nothing and costs you something.
2. **There is one pool, and everything shares it.** Your request handlers, your `Task.Run` calls, your timer callbacks, your logging sinks, and your library's internal continuations all draw on the same ThreadPool. It is a shared, unpriced resource, and treating it as free is how services die.
3. **Blocking on async code is the original sin.** `.Result`, `.Wait()`, `GetAwaiter().GetResult()`, and `Task.Run(...).Wait()` convert a scalable design into a thread-per-request one — with worse constants than thread-per-request ever had.
4. **`await` is a suspension point, and a suspension point is a scheduling decision.** Who resumes you, on which thread, with what ambient state, is decided by `SynchronizationContext`, `TaskScheduler`, and `ExecutionContext` — three different things that people merge into "the context."
5. **Unbounded concurrency is unbounded failure.** `Task.WhenAll` over a list you did not bound is a load test against your own dependency. Every fan-out needs a number attached, and that number is an architectural decision.
6. **The compiler and the CPU are both allowed to reorder your code.** Single-threaded semantics are preserved; cross-thread visibility is not, unless you asked for it. On x64 you often get away with it. On Arm64 you do not.
7. **Thread-safety is a contract, not a property.** `HttpClient` is safe, `DbContext` is not, `Random.Shared` is, `StringBuilder` is not. A senior candidate reads the contract; a mid-level candidate guesses from the name.
8. **Concurrency bugs are latent, not absent.** A race that has not fired is still a race. Load, a faster machine, more cores, a different architecture, or a GC pause at the wrong moment turns "works fine" into a 3 a.m. page.
9. **Measurement discipline applies harder here than anywhere.** Concurrency changes interact with the scheduler, the pool, and the hardware. "It seemed faster" is not evidence, and a microbenchmark of a concurrent system is usually measuring the wrong thing entirely.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Concurrency / parallelism / asynchrony | Three orthogonal ideas; conflating them is the root of most bad async answers |
| 2 | The cost of a thread | ~1 MB reserved stack, kernel object, scheduler slot, ~1–10 µs context switch — threads are the scarce resource |
| 3 | I/O-bound vs CPU-bound | Async helps the first and does nothing for the second; parallelism is the inverse |
| 4 | What async I/O actually does | The OS completes the operation; no thread waits — IOCP on Windows, epoll on Linux |
| 5 | Little's Law for threads | concurrency = arrival rate × latency; this is your thread budget, and it is arithmetic, not opinion |
| 6 | Async contagion | `async` propagates up the call stack because suspension can't be hidden behind a synchronous signature |
| 7 | Why not green threads | .NET ran the experiment and rejected it; the interop and ecosystem costs exceeded the ergonomic win |
| 8 | `Task` as a promise | State flags + result slot + continuation object; `await` registers a continuation |
| 9 | `TaskCompletionSource<T>` | The universal adapter from any completion source to a `Task`; always `RunContinuationsAsynchronously` |
| 10 | The compiler transform | A struct state machine + a builder; `MoveNext` is a switch; `await` is `IsCompleted`/`OnCompleted`/`GetResult` |
| 11 | The awaitable pattern | Awaitable is a *shape*, not an interface — any type with a conforming `GetAwaiter()` works |
| 12 | The synchronous fast path | A completed awaitable never suspends, never allocates, and never hops threads |
| 13 | Where the allocations are | Zero on the fully-synchronous path; one box on first real suspension; the box is also the `Task` |
| 14 | `ValueTask<T>` | For usually-synchronous hot paths; await exactly once, never twice, never concurrently |
| 15 | `IValueTaskSource` | How the BCL achieves zero-allocation async I/O; you almost never implement it, but you should know it exists |
| 16 | Runtime async (.NET 11) | The runtime, not the compiler, owns suspension: cleaner live stacks, fewer allocations, preview in .NET 11 |
| 17 | Exceptions in async | Captured into the `Task`, rethrown at `await` with the stack preserved via `ExceptionDispatchInfo` |
| 18 | `async void` | No `Task` to observe, so exceptions go to the process; only legitimate for event handlers |
| 19 | Cancellation | Cooperative, not preemptive; a token is a request, and ignoring it is a design choice you must make consciously |
| 20 | Timeouts and linked tokens | `WaitAsync`, `CancelAfter`, `CreateLinkedTokenSource` — and the registration leak nobody disposes |
| 21 | `SynchronizationContext` | The "resume here" policy; UI frameworks install one, ASP.NET Core deliberately does not |
| 22 | `TaskScheduler` | The "run here" policy for `Task` bodies; `Default` is the ThreadPool |
| 23 | `ExecutionContext` / `AsyncLocal<T>` | Ambient state that flows across `await` regardless of `ConfigureAwait`; copy-on-write, and it leaks if you let it |
| 24 | `ConfigureAwait(false)` | Opts out of resuming on the captured context; it does **not** change threads, suppress exceptions, or affect `AsyncLocal` |
| 25 | `ConfigureAwaitOptions` | .NET 8: `SuppressThrowing` and `ForceYielding` are real tools, not trivia |
| 26 | The ThreadPool | Global FIFO queue + per-thread LIFO local queues + work stealing; one pool shared by everything |
| 27 | Thread injection | Free up to `MinThreads` (= core count), then roughly one thread per 500 ms; this is the outage multiplier |
| 28 | ThreadPool starvation | Queue grows, thread count climbs slowly, latency explodes, CPU stays low — the signature is unmistakable |
| 29 | I/O completion | Async I/O completions come back on pool threads; there is no separate "IO thread" pool to tune in .NET Core |
| 30 | Blocking on async | `.Result`/`.Wait()` burn a pool thread and remove the only benefit async provides |
| 31 | The classic deadlock | One-thread context + a blocked caller = permanent stall; ASP.NET Core has no context, which hides the bug, not the cost |
| 32 | `Task.Run` | For moving CPU work off the caller; never to "make a sync method async" in a library |
| 33 | Dedicated threads | Long-running, blocking, or affinity-bound work belongs on its own thread, not the pool |
| 34 | Why a memory model | The compiler, JIT, and CPU all reorder; single-thread semantics are preserved, cross-thread visibility is not |
| 35 | The .NET memory model | Stronger than ECMA-335; documented in the runtime repo; x64 is TSO, Arm64 is weak |
| 36 | Atomicity and tearing | Reference and aligned word-size reads/writes are atomic; anything wider can tear |
| 37 | `volatile` | Acquire on read, release on write; not allowed on `long`/`double`; `Volatile.Read/Write` is the general form |
| 38 | `Interlocked` | Compare-and-swap is the primitive; full fence; the basis of every lock-free algorithm |
| 39 | Double-checked locking | Correct only with the right barriers; use `Lazy<T>` and stop writing it by hand |
| 40 | False sharing | Two hot variables on one 64-byte cache line destroy scaling; pad or separate |
| 41 | When lock-free is worth it | Rarely; contention must be measured first and correctness is much harder to prove |
| 42 | `lock` / `Monitor` | Thin lock in the object header, inflates to a sync block under contention; reentrant and thread-affine |
| 43 | `System.Threading.Lock` | .NET 9's dedicated type: faster, clearer, `EnterScope()`, and analyzers that catch misuse |
| 44 | No `await` inside `lock` | Monitor is thread-affine and the continuation may resume elsewhere; the compiler forbids it |
| 45 | `SemaphoreSlim` | The async-capable gate: `WaitAsync`, non-reentrant, not thread-affine — the standard critical section for async code |
| 46 | `ReaderWriterLockSlim` | Only wins with long reads and rare writes; usually loses to plain `lock` or an immutable snapshot |
| 47 | Cross-process primitives | Named `Mutex`/`Semaphore` are OS objects; they are distributed locks with all the caveats from Module 9 |
| 48 | Events and barriers | `ManualResetEventSlim`, `AutoResetEvent`, `CountdownEvent`, `Barrier` — mostly legacy in async code |
| 49 | `SpinLock` / `SpinWait` | Only for very short, uncontended critical sections; never with anything that can block |
| 50 | Deadlock taxonomy | Lock ordering, lock convoys, sync-over-async, exhausted pools, and the async-only variants |
| 51 | Async coordination | Build it from `TaskCompletionSource` + `SemaphoreSlim` + `Channel<T>`; don't reinvent monitors |
| 52 | Thread-safety as a contract | `HttpClient` yes, `DbContext` no, `Random.Shared` yes, `StringBuilder` no — read, don't guess |
| 53 | `ConcurrentDictionary` | Lock-free reads, striped-lock writes; `GetOrAdd`'s factory can run more than once and runs outside the lock |
| 54 | Queue / Stack / Bag | Lock-free segment-based; `ConcurrentBag` is thread-affine and wrong for most producer/consumer uses |
| 55 | Immutable collections | Copy-on-write + atomic reference swap is a legitimate, easily-reasoned concurrency strategy |
| 56 | `BlockingCollection<T>` | Blocks threads by design; `Channel<T>` is the async replacement |
| 57 | `Channel<T>` | The in-process queue with real backpressure; bounded by default in any design you'd defend |
| 58 | `IAsyncEnumerable<T>` | Streaming with backpressure built in; `[EnumeratorCancellation]` and `WithCancellation` are not optional |
| 59 | `System.IO.Pipelines` | Buffer management and backpressure for byte streams; the substrate under Kestrel |
| 60 | TPL Dataflow | Still the best fit for in-process pipelines with per-block parallelism and bounded capacity |
| 61 | Data parallelism | `Parallel`/PLINQ for CPU-bound partitionable work; partitioning and chunk size decide whether it helps |
| 62 | `Parallel.ForEachAsync` | .NET 6+: the correct primitive for bounded asynchronous fan-out |
| 63 | `WhenAll` / `WhenAny` / `WhenEach` | Different exception semantics; `WhenAny` leaves tasks unobserved; `WhenEach` (.NET 9) streams completions |
| 64 | Bounded concurrency | Every fan-out carries a number; the number is derived from the dependency's capacity, not from taste |
| 65 | Backpressure | A full queue must slow the producer; dropping, rejecting, or blocking are the only honest options |
| 66 | Fire-and-forget | An unobserved `Task` is a lost error and a shutdown hazard; make it explicit or make it a channel |
| 67 | Background work | `BackgroundService` is a long-lived scope problem; resolve scoped services per unit of work |
| 68 | Graceful shutdown | Tokens flow, `StopAsync` has a deadline, and in-flight work must either finish or be safely abandoned |
| 69 | Time and testability | `PeriodicTimer`, `TimeProvider`, `FakeTimeProvider` — stop using `Task.Delay` in tests |
| 70 | Rate limiting | `System.Threading.RateLimiting` gives you the four classic algorithms as first-class types |
| 71 | Concurrency at the data layer | Optimistic concurrency, idempotency keys, and why in-process locks don't survive scale-out |
| 72 | Observability | Queue length, thread count, lock contention, and `dotnet.thread_pool.*` meters are your early warning |
| 73 | The diagnostic workflow | counters → `dotnet-stack` → dump + `dumpasync`/`syncblk`; the signature tells you which one you have |
| 74 | Testing concurrency | Determinism through seams, not through `Thread.Sleep`; stress tests find what unit tests cannot |
| 75 | The async boundary | Library rules: no sync-over-async, take a `CancellationToken`, `ConfigureAwait(false)`, don't expose both |
| 76 | When not to be async | Console tools, CPU pipelines, tight in-memory loops, and anything where the thread was never going to block |

---

# Part A — What concurrency actually is

## Concept 1 — Concurrency, parallelism, and asynchrony are three different things

English treats these as synonyms. They are not, and separating them is the single cheapest upgrade to how you sound in an interview.

- **Concurrency** is a *structural* property: the program has multiple independent logical flows in progress at the same time. It says nothing about how many run simultaneously. A single-core machine handling 10,000 open sockets is highly concurrent.
- **Parallelism** is an *execution* property: multiple things are physically executing at the same instant, which requires multiple cores. Parallelism is a way of *implementing* concurrency, not a synonym for it.
- **Asynchrony** is a *control-flow* property: an operation is started and its completion is delivered later, rather than the caller waiting inline for it. Asynchrony is a way of *achieving* concurrency without dedicating a thread per flow.

Rob Pike's formulation is the one to remember: *concurrency is about dealing with many things at once; parallelism is about doing many things at once.* Concurrency is a design property; parallelism is a hardware fact.

The matrix that makes this concrete in .NET:

| | No parallelism | Parallelism |
|---|---|---|
| **Blocking** | Sequential code; one request at a time | `Parallel.For` over a CPU-bound workload on N cores |
| **Asynchronous** | One thread juggling thousands of in-flight I/O operations | An ASP.NET Core server: many cores, each juggling many requests |

The bottom-right cell is what a modern .NET service actually is, and it is why both halves of this module matter. You need async to get density per thread, and you need the parallel/thread-safety half to survive multiple cores touching shared state.

**Where this earns points:** when asked "does `async` make my code multithreaded?", the correct answer is *no — `async` is about not occupying a thread while waiting; it can be entirely single-threaded. What it does do is make your code re-entrant and non-deterministic about which thread resumes it, which is why shared state still needs the same care.* That answer contains three separate correct ideas and takes ten seconds.

---

## Concept 2 — The cost of a thread

Async exists because threads are expensive. Specifically:

| Cost | Typical figure |
|---|---|
| Reserved virtual address space for the stack | 1 MB (Windows default; configurable, `Thread` ctor takes `maxStackSize`) |
| Committed memory initially | One page or a few, growing on demand |
| Kernel objects | A thread kernel object, TEB, scheduler entry |
| Creation/teardown | ~0.1–1 ms including the runtime's bookkeeping |
| Context switch | ~1–10 µs, plus a much larger *hidden* cost: a cold L1/L2 cache and possibly a TLB flush |
| Runtime bookkeeping per thread | Allocation context, GC stack scan, thread-static storage |

Two consequences drive everything else in this module:

1. **Memory.** 10,000 concurrent requests at one thread each is 10 GB of reserved address space and thousands of scheduler entries. This is the "C10K problem," and thread-per-request is the answer that doesn't scale.
2. **Scheduling.** Beyond roughly the core count, more runnable threads do not add throughput; they add context switches and cache thrash. A machine with 8 cores and 500 runnable threads is spending a meaningful fraction of its cycles deciding what to run.

Note carefully what is *not* on the list: **a thread blocked in a kernel wait costs almost no CPU.** This is why "blocking wastes CPU" is a wrong explanation that gets repeated constantly. Blocking wastes a *thread* — a scarce, expensive, pre-allocated resource — and when the pool of those runs out, the consequence is a latency cliff, not high CPU. Getting this right in an interview is a clear seniority marker, because the follow-up question is always some variant of "then why was CPU low during the incident?"

The runtime's own accounting adds a wrinkle worth knowing: every managed thread's stack must be scanned by the GC at every collection, so thread count is also a GC cost (Module 14, Concept 23), and a thread that never reaches a safe point delays suspension for *everyone* (Module 14, Concept 32).

---

## Concept 3 — The two workloads: I/O-bound and CPU-bound

Every unit of work is dominated either by waiting for something external or by executing instructions. The tool that helps one actively harms the other.

| | I/O-bound | CPU-bound |
|---|---|---|
| Example | Database query, HTTP call, file read, queue receive | JSON serialization of a large graph, image resize, hashing, compression |
| Bottleneck | Network/disk latency; the thread has nothing to do | Cores; the thread has everything to do |
| Right tool | `async`/`await` over true async APIs | `Parallel.For`/PLINQ, or just efficient sequential code |
| Wrong tool | `Task.Run` around a blocking call (moves the block, doesn't remove it) | `async` (adds overhead, saves nothing) |
| Scaling limit | Downstream capacity, connection pools, sockets | Core count; Amdahl's law |
| What "more concurrency" buys | Throughput, until the dependency saturates | Nothing beyond core count; below zero past it |

The decisive question to ask about any candidate `await`: **would this thread otherwise be blocked in a kernel wait?** If yes, async is buying you a thread. If no — if the work is CPU-bound, or the "async" API is a synchronous method wrapped in `Task.Run` — you are paying the state machine and scheduling cost for nothing.

The mixed case is where judgement shows. A request handler that does 5 ms of CPU work and three 30 ms database calls is I/O-bound overall (95 ms of 105 ms is waiting) and should be async end to end; the 5 ms of CPU work stays inline on the pool thread, because moving it with `Task.Run` would add a scheduling hop to save nothing. A handler that does 400 ms of CPU work per request is CPU-bound, and no amount of `async` will save it — what it needs is either less work, a different algorithm, or to be moved out of the request path entirely (Module 11).

---

## Concept 4 — What "async I/O" actually does

This is the concept most candidates have never actually looked at, and it is the foundation for everything else.

When you call a **blocking** `socket.Receive()`, the thread makes a syscall, the kernel finds no data, and the kernel marks the thread as not-runnable and switches to something else. The thread is parked. When data arrives, an interrupt handler marks the thread runnable and the scheduler eventually resumes it. Cost: one thread, parked for the duration.

When you call an **asynchronous** `socket.ReceiveAsync()`, the thread issues the operation to the kernel and **returns immediately**. The kernel owns the operation. When data arrives, the completion is delivered through a platform mechanism, a pool thread picks it up, and *that* thread runs your continuation. Cost: no thread for the duration of the wait, one thread briefly at completion.

The platform mechanisms:

- **Windows: I/O Completion Ports (IOCP).** The handle is bound to a completion port; completed operations are queued to the port; threads dequeue completions. This is a kernel-level, natively asynchronous design, and it is why Windows server I/O has always scaled well.
- **Linux: epoll.** Linux's file I/O has historically lacked a good completion model, so .NET's socket stack uses a readiness-based design — a small number of dedicated engine threads run `epoll_wait` and, when a socket becomes ready, queue the actual read/write work to the ThreadPool. The net effect is the same from your code's point of view.
- **File I/O** is the honest exception. On Linux, "async" file reads frequently fall back to synchronous work on a pool thread because the platform has no universally available completion mechanism for regular files. `await File.ReadAllTextAsync(...)` is not necessarily doing what its name implies. On Windows, real overlapped file I/O requires the handle to have been opened for it (`FileOptions.Asynchronous`, which `FileStream` does when you ask for async).

**The sentence to be able to say:** *"During an async I/O wait, there is no thread. The operation is owned by the kernel, and a ThreadPool thread is only involved again when the completion arrives."* That single sentence answers roughly a third of all async interview questions, and it explains why `Task.Run(() => blockingCall())` is not async — it doesn't remove the block, it just relocates it onto a thread you care about more.

---

## Concept 5 — Little's Law for threads: the concurrency budget

Module 6 introduced Little's Law for queues. Applied to threads it becomes the most useful back-of-envelope in this module:

```
threads needed = arrival rate (req/s) × average time a thread is held (s)
```

Worked, out loud, in an interview:

- **Synchronous service**, 500 req/s, 200 ms of which 190 ms is waiting on a database: a thread is held for the full 200 ms. 500 × 0.2 = **100 threads permanently occupied.** On an 8-core box with `MinThreads = 8`, the pool must inject 92 extra threads at roughly two per second: about 46 seconds of degraded service *every time load arrives*, plus ~100 MB of stack and a scheduler full of mostly-parked threads.
- **The same service, async**: a thread is held only for the 10 ms of actual work. 500 × 0.01 = **5 threads.** It fits in the default pool with room to spare, and there is no injection ramp at all.

That is the whole value proposition of async, expressed as arithmetic. It is also the tool for sizing: if you know your arrival rate and your per-request thread-hold time, you know your thread requirement, and you know whether you need to touch `MinThreads`.

The same law gives you the *other* direction, which is the one people forget: **if you make everything async and put no limit on fan-out, you have removed your own admission control.** The synchronous version was implicitly limited by thread count — at 100 threads, request 101 queued. The async version happily starts 10,000 concurrent database calls and kills the database instead. This is why Concept 64 (bounded concurrency) exists, and why "we went async and our database fell over" is a real, common incident.

---

## Concept 6 — Async contagion, and why it is not a design flaw

`async` propagates upward: to `await` something you must be in an `async` method, whose callers must `await` it, and so on to the entry point. This is often complained about as "function colouring." It is worth understanding why it is unavoidable in this design rather than treating it as a wart.

A synchronous method must return a *value* when it returns. An asynchronous operation cannot produce its value at return time — that is the entire point. So it returns a *promise* of the value instead, and the signature necessarily differs. Hiding that behind a synchronous signature requires blocking, which reintroduces the thread cost you were trying to avoid. The colour is not arbitrary; it is the type system telling the truth about when the value exists.

Practical consequences you should state:

1. **"Async all the way down" is not dogma, it is the only way the benefit survives.** One `.Result` anywhere in the chain converts the whole chain back to thread-per-operation, at the entry point where it hurts most.
2. **The boundary belongs at the top**, where the framework provides an async entry point: an ASP.NET Core action, a `BackgroundService.ExecuteAsync`, an `async Main`, a message handler. Below that, everything stays async.
3. **Interfaces must be designed for it.** Adding `async` to an implementation is a breaking change to the interface. This is a real architectural cost of introducing async late, and it is the honest answer to "why can't we just make this one method async?"
4. **The alternative was considered and rejected** — see Concept 7.

---

## Concept 7 — Why .NET didn't adopt green threads

This is a genuinely differentiating question, and it comes up increasingly often now that Java has shipped virtual threads.

The alternative model — **green threads** (Java's Project Loom "virtual threads", Go's goroutines) — makes the *runtime* responsible for the trick. You write ordinary blocking code; the runtime gives each logical flow a small, growable stack, and when that flow blocks on I/O, the runtime parks the whole stack and schedules another onto the same OS thread. No `async` keyword, no colouring, no contagion.

.NET actually built and measured this, in a runtimelab experiment, and **decided not to ship it**. The reported reasons are worth knowing, because they are all architecture-flavoured trade-offs:

- **Interop is the killer.** A green thread that calls into native code cannot be parked — the native frame owns a real OS stack. .NET's ecosystem is deeply interop-bound (P/Invoke, COM, native database drivers), so the model would have had a large, confusing set of "this blocks a real thread anyway" holes.
- **Stack layout and the GC.** Growable/segmented or copied stacks conflict with .NET's existing exact stack scanning, pinning, and `ref`/`Span<T>` semantics (Module 14, Concepts 19–20). Stacks that move are a very large change to guarantees the platform already made.
- **Ecosystem duplication.** Every existing async API would need a blocking twin to be worth using from green threads, and the existing `async` surface — by then the entire BCL, ASP.NET Core, EF Core, and every major library — could not be abandoned.
- **The performance win was not there.** The experiment did not show a throughput advantage over the existing state-machine model large enough to justify the cost.

Instead, .NET invested in making the *existing* model cheaper — which is exactly what runtime async (Concept 16) is. The strategic framing to offer: **Java had to add virtual threads because its ecosystem was blocking-by-default and retrofitting async to it was impossible; .NET had already paid the async migration cost a decade earlier, so the marginal value of green threads was small and the marginal cost was large.** That answer shows you can reason about platform strategy, not just APIs.

---

# Part B — `Task` and the async machinery

## Concept 8 — `Task` as a promise: state, result, continuations

Strip away the API surface and a `Task` is three things:

1. **A state machine of its own**, held in an `int` of flags: created → scheduled/waiting-for-activation → running → one of `RanToCompletion`, `Faulted`, `Canceled`. The terminal states are final; a completed `Task` never changes again.
2. **A result slot** (`Task<TResult>` only) and an exception holder.
3. **A continuation object** — the list of things to do when it completes. This field is an optimization worth knowing: it starts `null`, becomes the *single* continuation object directly when one is registered (the common case — most tasks are awaited exactly once), and only becomes a `List<object>` when a second continuation arrives.

`await` is, mechanically, "register a continuation on this task, then return." Completion is "set the result, then run the continuations."

Two `Task` flavours that people conflate:

- **Promise-style tasks** have no delegate. They represent something that will complete because of an external event: `TaskCompletionSource.Task`, the result of `HttpClient.SendAsync`, the task returned by an `async` method. Nothing "runs" them.
- **Delegate tasks** have a body to execute and need a `TaskScheduler` to run them: `Task.Run`, `Task.Factory.StartNew`, `new Task(...).Start()`.

This distinction resolves a perennially confusing question. `Task.Run(() => Foo())` needs a thread *to run `Foo`*. `httpClient.GetAsync(...)` does not need a thread at all while the request is in flight. Both are `Task`. Only one of them consumed a pool thread to exist.

A few `Task` facts that get asked directly:

- **`Task.Status`** includes `WaitingForActivation` (a promise task with no result yet — what an in-flight `async` method's task looks like) and `WaitingForChildrenToComplete` (only relevant with attached child tasks, which is a legacy TPL feature you should avoid; `Task.Run` implies `DenyChildAttach`).
- **A `Task` is not restartable and not reusable.** `Task.CompletedTask`, `Task.FromResult`, and cached `Task<bool>` singletons exist precisely because allocating a fresh completed task for every synchronous return is wasteful.
- **`Task.Delay`** does not use a thread. It sets a timer; the callback completes the task.
- **`Task.Yield()`** completes immediately but *forces* the continuation to be scheduled rather than run inline, which is how you deliberately give the pool a chance to run something else, or break out of a synchronous-completion loop that would otherwise starve other work.

---

## Concept 9 — `TaskCompletionSource<T>`: the universal adapter

`TaskCompletionSource<T>` is a `Task` you complete manually. It is the bridge from *any* completion mechanism — an event, a callback, a native completion, a message arriving on a socket — into the `await`able world.

```csharp
public static Task<string> WaitForMessageAsync(IBus bus, CancellationToken ct)
{
    // ALWAYS RunContinuationsAsynchronously. See below.
    var tcs = new TaskCompletionSource<string>(TaskCreationOptions.RunContinuationsAsynchronously);

    void Handler(object? _, MessageEventArgs e) => tcs.TrySetResult(e.Body);
    bus.MessageReceived += Handler;

    CancellationTokenRegistration reg = ct.Register(() => tcs.TrySetCanceled(ct));

    return tcs.Task.ContinueWith(t =>
    {
        bus.MessageReceived -= Handler;   // unsubscribe: this is an event-handler leak otherwise
        reg.Dispose();                     // dispose the registration: otherwise it lives as long as the token
        return t.GetAwaiter().GetResult();
    }, TaskContinuationOptions.ExecuteSynchronously);
}
```

Three rules, all of which are interview-grade:

1. **Always pass `TaskCreationOptions.RunContinuationsAsynchronously`.** Without it, `SetResult` runs every registered continuation *inline, on the thread that called `SetResult`*. If that thread is your socket-reading loop, your protocol dispatcher, or — worst case — a thread holding a lock, you have just handed arbitrary user code your thread. This causes real production incidents: stalls, unexpected reentrancy, and deadlocks where the continuation tries to take a lock the completing thread already holds. This one flag is the single most valuable piece of `TaskCompletionSource` knowledge.
2. **Use `TrySetResult`/`TrySetException`/`TrySetCanceled`.** In a race between completion and cancellation, the non-`Try` versions throw `InvalidOperationException` on the loser. You almost never want that.
3. **Use `TaskCompletionSource` (non-generic, .NET 5+)** when there is no result, instead of the old `TaskCompletionSource<bool>` idiom.

`TaskCompletionSource` is also how you should think about *what an `async` method returns*: the compiler-generated builder is doing essentially this, with the state machine as the thing that completes it.

---

## Concept 10 — The compiler transform: what `async` actually generates

This is the centrepiece of Part B. Given:

```csharp
public async Task<int> GetLengthAsync(string url)
{
    string body = await _http.GetStringAsync(url);
    return body.Length;
}
```

the C# compiler emits roughly this (names mangled, simplified but structurally accurate):

```csharp
[AsyncStateMachine(typeof(<GetLengthAsync>d__1))]
public Task<int> GetLengthAsync(string url)
{
    var sm = new <GetLengthAsync>d__1();
    sm.<>t__builder = AsyncTaskMethodBuilder<int>.Create();
    sm.<>4__this = this;
    sm.url = url;
    sm.<>1__state = -1;
    sm.<>t__builder.Start(ref sm);   // runs MoveNext() SYNCHRONOUSLY, on the calling thread
    return sm.<>t__builder.Task;
}

private struct <GetLengthAsync>d__1 : IAsyncStateMachine
{
    public int <>1__state;
    public AsyncTaskMethodBuilder<int> <>t__builder;
    public HttpClient <>4__this;      // captured 'this'
    public string url;                // hoisted parameter
    private string <body>5__2;        // hoisted local: survives across the await
    private TaskAwaiter<string> <>u__1;

    public void MoveNext()
    {
        int state = <>1__state;
        try
        {
            TaskAwaiter<string> awaiter;
            if (state != 0)
            {
                awaiter = <>4__this.GetStringAsync(url).GetAwaiter();
                if (!awaiter.IsCompleted)                       // <-- the fast path check
                {
                    <>1__state = 0;
                    <>u__1 = awaiter;
                    <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref this); // <-- BOXES here
                    return;                                      // <-- the method RETURNS to its caller
                }
            }
            else
            {
                awaiter = <>u__1;
                <>u__1 = default;
                <>1__state = -1;
            }

            <body>5__2 = awaiter.GetResult();                    // rethrows if faulted
            <>1__state = -2;
            <>t__builder.SetResult(<body>5__2.Length);
        }
        catch (Exception e)
        {
            <>1__state = -2;
            <>t__builder.SetException(e);                        // exception goes INTO the Task
        }
    }
}
```

Everything you need to reason about async behaviour is visible here:

- **The method body runs synchronously until the first incomplete await.** `Start` calls `MoveNext` directly on the calling thread. Your argument validation, your logging, your first database call's setup — all of it happens before the caller gets its `Task` back. This is why argument validation in an `async` method throws *into the task* instead of at the call site, and why the iterator-style split (a synchronous validating wrapper around a private `async` implementation) exists.
- **Locals that live across an `await` become fields.** `body` is hoisted. Locals that don't cross an await stay as ordinary stack locals. This is the mechanism behind "async moves your stack to the heap" (Module 14, Concept 51) — and the reason a method with a 4 KB `stackalloc` or a large struct held across an await has a surprisingly large state machine.
- **`await` is three operations, not one**: `IsCompleted` (can we skip suspension?), `OnCompleted`/`AwaitUnsafeOnCompleted` (register the resumption), `GetResult` (fetch the value or rethrow).
- **Returning from `MoveNext` after registering is how the thread is freed.** The thread literally returns out of your method and goes back to the pool. Later, on some thread, `MoveNext` is called again and the `switch` jumps to the resume label.
- **Exceptions are captured, not thrown.** `SetException` puts the exception in the `Task`. It surfaces when someone `await`s it — and if nobody does, it surfaces nowhere (Concept 18).
- **In Release builds the state machine is a `struct`**; in Debug it is a `class`, so that the debugger can inspect it. This is why a "measure allocations in Debug" benchmark is meaningless here.

**The `Unsafe` in `AwaitUnsafeOnCompleted`** means "don't capture `ExecutionContext` here" — the builder captures it once at the box, rather than at every await. It is a performance detail, not a safety hole.

---

## Concept 11 — The awaitable pattern: a shape, not an interface

`await` is not tied to `Task`. The compiler requires only a *shape*:

```csharp
// awaitable: has a GetAwaiter() (instance method or extension method)
public MyAwaiter GetAwaiter();

// awaiter: implements INotifyCompletion (or ICriticalNotifyCompletion) and has:
public bool IsCompleted { get; }
public void OnCompleted(Action continuation);          // from INotifyCompletion
public void UnsafeOnCompleted(Action continuation);    // from ICriticalNotifyCompletion (skips EC capture)
public T GetResult();                                  // or void
```

Anything with that shape is awaitable. This is why all of these work and none of them are special-cased in the compiler:

- `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`
- `Task.Yield()` (returns `YieldAwaitable`, whose `IsCompleted` is **always false** — that's the whole trick)
- `ConfigureAwait(false)` (returns `ConfiguredTaskAwaitable`, a different awaitable with a different awaiter)
- `IAsyncEnumerable<T>`'s `MoveNextAsync()` returning `ValueTask<bool>`
- `await using` via `IAsyncDisposable.DisposeAsync()`
- Custom awaitables in UI frameworks, game engines (`await nextFrame`), and Unity-style coroutine adapters

The symmetric mechanism on the return side is **`[AsyncMethodBuilder]`**: an `async` method can return any type that declares a builder. That's how `ValueTask` works, how `IAsyncEnumerable<T>` works as an `async` return type, and how libraries build things like `UniTask` or pooled task types. Since C# 10 you can put `[AsyncMethodBuilder(typeof(...))]` on an individual method, not just a type.

**Why this matters in an interview:** being able to say *"awaitable is a duck-typed pattern, which is why `Task.Yield()` and `ConfigureAwait` are ordinary library types rather than compiler magic"* demonstrates you have actually read the machinery rather than memorized its behaviour.

---

## Concept 12 — The synchronous fast path

`if (!awaiter.IsCompleted)` is the most important line in the generated code. When the awaited operation is **already complete**:

- No suspension. No state machine box. No `ExecutionContext` capture. No continuation. No thread hop.
- `GetResult()` is called immediately and execution continues straight through, in the same stack frame, on the same thread.

This path is genuinely cheap — the residual cost is a few field writes and a branch. It is why the following are all fine:

- Async methods over caches that usually hit: `ValueTask<T> GetAsync(key)` returning a cached value synchronously ~95% of the time.
- `Stream.ReadAsync` over a buffered stream where the data is already in the buffer.
- `Channel<T>.Reader.ReadAsync()` when an item is already queued.

And it is why the conventional wisdom "avoid `async` on hot paths" is out of date: the *suspending* path is expensive, the *completing-synchronously* path is not. Measure which one you are actually on.

One important caveat: **a method that always completes synchronously never yields**. If you write a loop that awaits something always-complete, you have written a synchronous loop that never gives the pool thread back. `await Task.Yield()` (or reading from a bounded channel) is how you reintroduce a yield point deliberately. This is the async analogue of a thread that never reaches a GC safe point.

---

## Concept 13 — Where the allocations actually are

The complete allocation picture, which is the version Module 14 Concept 51 summarized:

| Path | Allocations |
|---|---|
| `async Task` method that completes synchronously | **Zero** (the state machine is a struct on the stack; `Task` is the cached `Task.CompletedTask`) |
| `async Task<T>` that completes synchronously | Usually one, for the `Task<T>` — unless `T` hits a cached value (`Task.FromResult` caches `true`/`false`/small ints) |
| `async ValueTask<T>` that completes synchronously | **Zero** |
| Any async method that **actually suspends** | **One** `AsyncStateMachineBox<TStateMachine>` |
| `ExecutionContext` with `AsyncLocal` values set | One `ExecutionContext` per mutation (copy-on-write) |
| Each additional continuation beyond the first on a task | A `List<object>` for the continuation field |

The box deserves a moment. `AsyncStateMachineBox<TStateMachine>` **derives from `Task<TResult>`** and holds the state machine as a field. So the one allocation is simultaneously the returned `Task`, the boxed state machine, and the continuation delegate target. This is a deliberate, elegant piece of BCL engineering: pre-.NET Core 2.1 it took three or four objects; now it takes one.

Practical guidance that follows:

1. **Don't `async`-wrap a synchronous result.** `async Task<int> GetAsync() { return 42; }` allocates; `Task<int> GetAsync() => Task.FromResult(42);` may not. But do not over-index on this: the moment the method can throw, the `async` version's exception handling is worth the allocation.
2. **Fewer awaits in a hot method is not the goal; fewer *suspensions* is.** Ten awaits that all complete synchronously cost one box total (zero, actually — they never suspend).
3. **The state machine's size is the sum of the hoisted locals.** A method holding three large structs across awaits has a large box. Restructure by extracting the awaiting portion into a small method.
4. **This is the mid-life-crisis case from Module 14.** A state machine for a long-running operation survives gen0 and gen1 and gets promoted. Thousands of concurrent long operations means thousands of promoted boxes.

---

## Concept 14 — `ValueTask<T>` and the rules you must not break

`ValueTask<T>` is a `readonly struct` that holds *one of three things*: a `T` result directly, a `Task<T>`, or an `IValueTaskSource<T>` plus a token. Its purpose is to avoid the `Task<T>` allocation when the operation usually completes synchronously.

**The consumption rules — this is the part interviewers test:**

| Rule | Why |
|---|---|
| Await it **exactly once** | The backing `IValueTaskSource` may be pooled and reused after you consume it; a second await reads someone else's result |
| Never await it **concurrently** | Same reason, with a race |
| Never call `.Result` / `.GetAwaiter().GetResult()` before it completes | Unlike `Task`, `ValueTask` has no blocking wait; the source may not support it |
| Don't store it for later | It may be invalid by then |
| Need any of the above? Call **`.AsTask()`** once and use that | `AsTask()` materializes a real `Task` that has normal semantics |
| Need to await twice specifically? **`.Preserve()`** | Converts to a `ValueTask` that can be awaited repeatedly |

**When to return `ValueTask<T>`:**
- The method usually completes synchronously (cache hit, buffered read, channel with data available), **and**
- It is on a hot path where the allocation is measurable, **and**
- The consumers are internal or documented.

**When to return `Task<T>`:** everything else. `Task` is composable (`WhenAll`, `WhenAny`, storing, caching, awaiting twice), larger by a pointer, and impossible to misuse. The BCL's own guidance is that `Task` remains the default and `ValueTask` is the considered optimization.

A `ValueTask` is 2–3 words (result/object + token + flag) versus a `Task`'s 8-byte reference — so a `ValueTask` field in a heavily-instantiated type is *bigger*, and passing it around costs more copying. "`ValueTask` is always better" is wrong in both directions.

The analyzers matter here: **CA2012** ("Use ValueTasks correctly") catches the double-await and store cases, and it should be an error, not a warning, in any codebase using `ValueTask`.

---

## Concept 15 — `IValueTaskSource` and how the BCL achieves zero-allocation I/O

You will almost certainly never implement this, but knowing it exists explains how `Socket.ReceiveAsync` manages zero steady-state allocations.

`IValueTaskSource<T>` lets an object *be* the backing of a `ValueTask<T>` without being a `Task`. The object exposes `GetStatus(token)`, `OnCompleted(...)`, and `GetResult(token)`. The **token** (a `short` version number) is the safety mechanism: each time the object is reused for a new operation, the token increments, so an accidental stale await is detected rather than silently reading the wrong result.

`ManualResetValueTaskSourceCore<T>` is the BCL's reusable implementation — you embed it as a field and forward the three methods. `Socket`'s `SocketAsyncEventArgs`, `System.IO.Pipelines`, and `Channel<T>` all use this pattern: one long-lived object per socket/pipe/channel, reused for every operation, so a server doing a million reads per second allocates essentially nothing for the read machinery.

The design lesson worth stating: **the allocation-free async path exists, it is achieved by pooling the completion source rather than the task, and the version token is what makes reuse safe.** That is exactly why the `ValueTask` consumption rules in Concept 14 are strict — they are the contract that makes the pooling sound.

---

## Concept 16 — Runtime async (.NET 11): the state machine moves into the runtime

This is the current-platform question for 2026, and it is worth knowing precisely.

**The problem with the compiler-based design.** Since C# 5, `async` has been a compiler rewrite. The runtime has no idea that `async` exists; it sees ordinary methods and generated classes. That has three costs: (1) every async method's frames are wrapped in `AsyncMethodBuilderCore.Start` infrastructure, so live stacks are cluttered and roughly 2–3× deeper than the real call chain; (2) the JIT cannot optimize across a suspension point because it does not know one exists; (3) the state machine box and the hoisted-local copying are decided by the compiler with no knowledge of the target machine.

**What runtime async changes.** The runtime becomes the owner of suspension and resumption. The JIT compiles a dedicated "async version" of the method, knows where suspension points are, and can allocate continuations itself — reusing continuation objects, skipping the save of locals that have not changed, and merging suspension points to reduce code size.

**The observable wins:**

- **Live stack traces show the real call chain.** The Microsoft documentation's example goes from 13 frames of state-machine infrastructure down to 5 real frames for the same three-deep async call. This benefits debuggers, profilers, and anything calling `new StackTrace()`. (Exception stack traces already looked right, because `ExceptionDispatchInfo` handled that case.)
- **Breakpoints bind inside async methods properly** and stepping over `await` no longer descends into generated infrastructure.
- **Fewer allocations**, through continuation reuse and not saving unchanged locals.
- **Tiered compilation applies to async versions**, so hot async methods get tier-1 optimization like anything else.
- **Continuations skip `ExecutionContext` capture/restore when there is nothing to restore** — a real throughput win for code that uses `ConfigureAwait(false)` and no `AsyncLocal<T>`.
- **The JIT recognizes `Task.FromResult`, `Task.CompletedTask`, `ValueTask.FromResult`** as intrinsics and folds them into faster paths.
- **NativeAOT and ReadyToRun are supported**, and R2R can now inline await-less async calls.

**The status, precisely, as of .NET 11 RC1 (September 2026):** runtime async is a **preview feature**. You opt in per project with `<Features>runtime-async=on</Features>`; a `net11.0` project no longer needs `<EnablePreviewFeatures>`. The .NET **runtime libraries themselves already ship compiled with it** — the BCL contains no compiler-generated async state machines any more — which is both a large-scale validation of the feature and the reason your app can benefit partially without opting in. The `DOTNET_RuntimeAsync`/`UNSUPPORTED_RuntimeAsync` environment variables have been removed; per-project opt-out is `<UseRuntimeAsync>false</UseRuntimeAsync>`.

**What does *not* change:** your source code. `async`/`await`, `Task`, `ValueTask`, `ConfigureAwait`, cancellation, and every rule in this module are unchanged. This is an implementation change, not a language change. The architectural framing to offer: *"It is the same strategic move as tiered compilation and DATAS — the platform absorbing something that used to be the caller's problem, so that the same source gets faster on a runtime upgrade."*

---

## Concept 17 — Exceptions in async code

Six behaviours, each with a reason:

1. **An exception in an `async` method is captured into the returned `Task`**, not thrown at the call site. `SetException` in the generated `catch`.
2. **`await` rethrows the *first* exception**, with its original stack trace preserved via `ExceptionDispatchInfo.Capture(...).Throw()`. This is why async stack traces are readable at all, and it is why the rethrown exception's stack shows both the original throw site and the await site.
3. **`.Wait()` and `.Result` throw `AggregateException`**, wrapping the original. This asymmetry is a frequent source of "why is my catch block not catching `HttpRequestException`?" bugs, and is one more reason not to block.
4. **`Task.WhenAll` collects all exceptions into the returned task's `AggregateException`, but awaiting it rethrows only the first.** To see all of them: `catch (Exception) when (task.Exception is { } agg)` and inspect `agg.InnerExceptions`, or `await Task.WhenAll(...).ContinueWith(...)`. The single most common bug here is a fan-out where four of five calls failed and the logs show one exception.
5. **Cancellation surfaces as `OperationCanceledException`** (or `TaskCanceledException`, which derives from it). Always catch `OperationCanceledException`, never the derived one, and check `ex.CancellationToken` when it matters whether it was *your* cancellation.
6. **`finally` blocks after an `await` run on the resuming thread**, whatever that is — which matters if the `finally` releases a thread-affine resource (Concept 44).

The pattern for validating arguments eagerly:

```csharp
public Task<Order> GetOrderAsync(int id, CancellationToken ct)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(id);   // throws AT THE CALL SITE
    return GetOrderCoreAsync(id, ct);

    async Task<Order> GetOrderCoreAsync(int id, CancellationToken ct) { ... }
}
```

The outer method is *not* `async`, so the guard throws synchronously to the caller rather than being packaged into a task that might never be awaited.

---

## Concept 18 — `async void`, and unobserved exceptions

**`async void` has no `Task`.** Consequences, all bad:

1. **Exceptions have nowhere to go.** The generated `AsyncVoidMethodBuilder.SetException` posts the exception to the captured `SynchronizationContext` — or, if there is none (ASP.NET Core, console apps), rethrows it on a ThreadPool thread, which **crashes the process**. Not "logs an error." Crashes it.
2. **It is unawaitable**, so callers cannot know when it finished, cannot sequence after it, and cannot participate in shutdown.
3. **It is untestable** for completion.

The one legitimate use: **event handlers**, because the event's delegate type is `void`-returning and you have no choice. Even there, the body should be a try/catch around an awaited call:

```csharp
private async void OnButtonClick(object sender, EventArgs e)
{
    try { await DoWorkAsync(); }
    catch (Exception ex) { _logger.LogError(ex, "Button handler failed"); }
}
```

**The related trap: unobserved exceptions on a real `Task`.** If an `async Task` method faults and nobody ever awaits the task, the exception sits in the task. When the task is finalized, `TaskScheduler.UnobservedTaskException` fires. Since .NET 4.5 this does **not** crash the process by default (it did in .NET 4.0), which means the failure is silently swallowed. Subscribe to it in production and log loudly — an unobserved exception is always a bug, and it is usually a fire-and-forget (Concept 66).

Three detection tools:
- **Analyzer CA2012 / VSTHRD100** (the Microsoft.VisualStudio.Threading.Analyzers package) flags `async void`.
- **CS4014** ("this call is not awaited") is a compiler warning and should be an error.
- `TaskScheduler.UnobservedTaskException` in your host startup, logging at Error.

---

## Concept 19 — Cancellation is cooperative

.NET has no preemptive cancellation. `Thread.Abort` is gone (it throws `PlatformNotSupportedException` on .NET Core), for the same reason Java deprecated `Thread.stop`: injecting an exception at an arbitrary instruction leaves invariants broken, locks held, and state corrupted.

What you have instead:

- **`CancellationTokenSource`** owns the cancellation; **`CancellationToken`** is the read-only view you hand out. Only the owner can cancel.
- **Three ways to observe**: poll `token.IsCancellationRequested`; call `token.ThrowIfCancellationRequested()`; register a callback with `token.Register(...)`.
- **Passing the token down is what makes it work.** A token that is not passed to the actual I/O call does nothing. Every async API in the BCL takes one; every one of yours should too.
- **`OperationCanceledException` is the protocol.** Throw it (via `ThrowIfCancellationRequested`) rather than returning a sentinel, so callers can distinguish "cancelled" from "completed with no results."

The rules that separate levels:

| | Mid-level | Senior |
|---|---|---|
| Token plumbing | Accepts `CancellationToken` but doesn't pass it on | Passes it to every call that takes one, including `SemaphoreSlim.WaitAsync` and `Channel.ReadAsync` |
| Catching | `catch (Exception)` swallows cancellation, turning shutdown into an error | `catch (OperationCanceledException) when (ct.IsCancellationRequested)` — distinguishes *my* cancellation from a downstream timeout |
| Registration lifetime | `token.Register(...)` and never disposes | Disposes the `CancellationTokenRegistration`; knows it otherwise lives as long as the token |
| Callback semantics | Assumes callbacks are safe | Knows `Cancel()` runs callbacks **synchronously on the calling thread** and that one throwing callback can disrupt the rest; uses `CancelAsync()` (.NET 8) when that matters |
| Non-cancellable work | Cancels and assumes it stopped | Knows the operation may still be running; ensures cleanup and idempotency downstream |

**The leak worth naming explicitly:** a long-lived token (a host's `ApplicationStopping`, or a request token in a long-lived scope) accumulates every `Register` and every `CreateLinkedTokenSource` that is never disposed. Each linked source registers a callback on its parent. In a loop, that is an unbounded list on a long-lived object — a textbook Module 14 retention bug, and one of the most common real leaks in .NET services.

---

## Concept 20 — Timeouts, linked tokens, and `WaitAsync`

```csharp
// 1. Timeout on a token source
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await DoWorkAsync(cts.Token);

// 2. Link caller cancellation + a timeout  — NOTE the 'using'
using var linked = CancellationTokenSource.CreateLinkedTokenSource(callerToken);
linked.CancelAfter(TimeSpan.FromSeconds(5));
await DoWorkAsync(linked.Token);

// 3. Timeout on a task that does NOT accept a token (.NET 6+)
await someTask.WaitAsync(TimeSpan.FromSeconds(5), callerToken);
```

The distinction that matters architecturally:

- **Cancellation propagated into the operation** (1 and 2) actually stops the work: the HTTP request is aborted, the SQL command is cancelled, the socket read is torn down. The resource is released.
- **`WaitAsync` / `Task.WhenAny(task, Task.Delay(...))`** only stops *your waiting*. The underlying operation keeps running, keeps holding its connection, and eventually completes into a task nobody is watching. This is the **abandoned-operation** pattern, and under load it is a resource leak that looks like a connection-pool exhaustion.

So: prefer real cancellation; use `WaitAsync` when the API gives you no choice, and know that you are abandoning, not cancelling. Also note `WaitAsync` throws `TimeoutException` on timeout (not `OperationCanceledException`) when the timeout fires rather than the token — a useful distinction to catch on separately.

Two more facts worth carrying:

- **`CancellationToken.None`** is not the same as `default` in intent but is identical in behaviour — a token that can never be cancelled, so `Register` is a no-op and `ThrowIfCancellationRequested` never throws.
- **`CancellationTokenSource.CancelAsync()`** (.NET 8) queues the callbacks rather than running them on your thread. Use it when cancelling from a path you don't want to hand to arbitrary callbacks — for instance, inside a lock or on a hot I/O loop.

---

# Part C — Scheduling: contexts, schedulers, and the ThreadPool

## Concept 21 — `SynchronizationContext`: the "resume here" policy

`SynchronizationContext` is an abstraction with essentially one interesting method, `Post(SendOrPostCallback d, object state)`, meaning "run this delegate according to my rules." Its job is to answer the question **"where should a continuation run?"**

The implementations that matter:

| Context | Installed by | `Post` does |
|---|---|---|
| `null` (none) | **ASP.NET Core**, console apps, `BackgroundService`, most library code | Nothing to post to; continuations go to the ThreadPool |
| `DispatcherSynchronizationContext` | WPF, WinUI | Queues to the UI dispatcher — **one specific thread** |
| `WindowsFormsSynchronizationContext` | WinForms | `Control.BeginInvoke` — **one specific thread** |
| `AspNetSynchronizationContext` | Classic ASP.NET (.NET Framework) | Restores `HttpContext`, `CurrentPrincipal`, culture; **allows one thread at a time** |
| xUnit's context | xUnit test runner | Serializes continuations for a test collection |

The default awaiter behaviour is: **at an `await`, capture `SynchronizationContext.Current` (or, if that is null, `TaskScheduler.Current` when it isn't `Default`); at resumption, post the continuation back to it.** That is what makes UI code work — `await`ing in a button handler and then touching a control is legal precisely because you resume on the UI thread.

The single most valuable fact in this concept: **ASP.NET Core deliberately has no `SynchronizationContext`.** It was removed in ASP.NET Core 1.0 because the classic one — which serialized request processing to one thread at a time and carried `HttpContext` — was both a performance cost and a deadlock generator. Its absence means:

1. `ConfigureAwait(false)` is a **no-op for deadlock purposes** in ASP.NET Core application code. (It is not useless — see Concept 24.)
2. The classic sync-over-async deadlock cannot happen in ASP.NET Core. This is why teams migrate from .NET Framework, see their deadlocks disappear, and incorrectly conclude that `.Result` is now safe. It is not — it still burns a pool thread (Concept 30).
3. Your continuation resumes on **a** ThreadPool thread, not necessarily the one you started on. Anything thread-affine — a thread-static, a `[ThreadStatic]` cache, a `ThreadLocal<T>`, an open `TransactionScope` (pre-`TransactionScopeAsyncFlowOption.Enabled`), a COM object, a native handle with thread affinity — is broken by an `await`.

---

## Concept 22 — `TaskScheduler`: the "run here" policy

`TaskScheduler` answers a related but different question: **when a `Task` with a delegate needs to execute, who runs it?**

- **`TaskScheduler.Default`** is the ThreadPool. This is what `Task.Run` uses.
- **`TaskScheduler.Current`** is the scheduler of the currently-executing task, or `Default` if none.
- **`TaskScheduler.FromCurrentSynchronizationContext()`** wraps a `SynchronizationContext` as a scheduler — the standard way to marshal a `ContinueWith` back to a UI thread in the pre-`async` style.
- **Custom schedulers** exist for constrained execution: `ConcurrentExclusiveSchedulerPair` (an exclusive scheduler plus a concurrent one, sharing a target) gives you reader/writer semantics for *tasks*, and `LimitedConcurrencyLevelTaskScheduler` (a well-known sample) caps parallelism.

The distinction from `SynchronizationContext` is real and worth being able to state: **`SynchronizationContext` governs where continuations are posted; `TaskScheduler` governs where task bodies are executed.** Both are captured at an `await` (context first, scheduler as a fallback), and `ConfigureAwait(false)` opts out of both.

A trap worth knowing: **`Task.Factory.StartNew` uses `TaskScheduler.Current`, not `Default`.** Inside a task already running on a custom scheduler, `StartNew` silently inherits it. `Task.Run` always uses `Default` and always passes `DenyChildAttach`. That is one of several reasons `Task.Run` is the right default and `StartNew` is the specialist tool.

---

## Concept 23 — `ExecutionContext` and `AsyncLocal<T>`

The third context, and the one people forget exists.

**`ExecutionContext` is ambient state that flows across async boundaries.** It carries `AsyncLocal<T>` values, the security context, and (historically) the logical call context. Critically:

> **`ExecutionContext` flows regardless of `ConfigureAwait(false)`.** `ConfigureAwait` controls `SynchronizationContext`/`TaskScheduler` resumption — it has nothing to do with `ExecutionContext`.

That one sentence resolves a large family of confused questions ("does `ConfigureAwait(false)` lose my correlation ID?" — no).

**`AsyncLocal<T>`** is the async-aware replacement for `[ThreadStatic]`. Its semantics are **copy-on-write down the call tree**:

```csharp
private static readonly AsyncLocal<string> _correlationId = new();

async Task ParentAsync()
{
    _correlationId.Value = "abc";
    await ChildAsync();                 // child sees "abc"
    Console.WriteLine(_correlationId.Value); // still "abc" — child's change is NOT visible here
}

async Task ChildAsync()
{
    Console.WriteLine(_correlationId.Value); // "abc"
    _correlationId.Value = "xyz";            // affects only this branch and below
}
```

Values flow **downward**, never upward. Set it before the fan-out, or use a mutable holder object (set the reference once, mutate the object) if you genuinely need upward propagation — with all the thread-safety consequences that implies.

Where `AsyncLocal<T>` is the right answer: correlation IDs, ambient tenant/user context, `Activity.Current` for distributed tracing (this is exactly how OpenTelemetry works — Module 11, Module 28), and test isolation.

Where it bites:

- **It is a retention mechanism.** Anything stored in an `AsyncLocal` is kept alive for the whole logical flow; in a long-lived flow (a `BackgroundService`, a long-running consumer), that is effectively forever.
- **The `ExecutionContext` allocates on every mutation** because of copy-on-write. Setting an `AsyncLocal` in a tight loop allocates a context per iteration.
- **`ExecutionContext.SuppressFlow()`** exists for deliberately *not* flowing — mostly used when starting long-lived background work that should not inherit a request's context (and therefore should not keep it alive).
- **.NET 11 optimizes the no-op case**: continuations now detect that there is nothing to restore and skip the capture/restore cycle entirely, which is a measurable throughput win for code that doesn't use `AsyncLocal`.

---

## Concept 24 — `ConfigureAwait(false)`: what it does and does not do

`task.ConfigureAwait(false)` means exactly one thing: **do not force the continuation back onto the captured `SynchronizationContext`/`TaskScheduler`.** The continuation runs either inline on the completing thread or on the ThreadPool.

What it does **not** do — every one of these is a real misconception you may be asked about:

| Myth | Reality |
|---|---|
| "It makes the code run on a background thread" | No. It removes a constraint on *where the continuation resumes*; it never moves work anywhere |
| "It's faster" | Only by the cost of the post, which in a context-free app is zero |
| "It prevents deadlocks" | It prevents *one specific* deadlock (Concept 31) caused by a single-threaded context |
| "It affects `AsyncLocal`/correlation IDs" | No. `ExecutionContext` flows regardless (Concept 23) |
| "I need it everywhere in ASP.NET Core" | There is no context to capture, so it changes nothing in app code |
| "`ConfigureAwait(true)` does something" | It is the default; writing it is noise |

**The rules that hold up:**

1. **In library code: always `ConfigureAwait(false)`.** You do not know who is calling you. If a WPF app calls your library, every `await` inside it would otherwise marshal back to the UI thread — hundreds of pointless dispatcher hops, and a deadlock waiting to happen if the caller blocks. This is not optional for anything shipped on NuGet.
2. **In application code with no context (ASP.NET Core, worker services, console): it is unnecessary.** Adding it is harmless but is pure noise. Teams reasonably decide either way; the defensible position is "we don't, and here's why," not "we don't know."
3. **In UI application code: you generally want the default (`true`)**, because you want to touch controls after the await. Use `ConfigureAwait(false)` for the non-UI portions, then let the final hop return to the UI.
4. **Enforce with an analyzer**, not a review convention: `CA2007` or `VSTHRD111`, scoped to library projects only.

One subtlety worth knowing for a deep round: **`ConfigureAwait(false)` does not guarantee a thread switch either.** If the awaited task is already complete, there is no suspension at all. If it completes on a pool thread, the continuation may run *inline on that thread*. `ConfigureAwait(false)` is a removal of a constraint, not an instruction.

---

## Concept 25 — `ConfigureAwaitOptions` (.NET 8)

.NET 8 added an overload on `Task` (not `Task<T>`) taking a flags enum, which makes two previously-awkward things easy:

```csharp
public enum ConfigureAwaitOptions
{
    None = 0,                       // equivalent to ConfigureAwait(false)
    ContinueOnCapturedContext = 1,  // equivalent to ConfigureAwait(true)
    SuppressThrowing = 2,           // await completes normally even if the task faulted or was cancelled
    ForceYielding = 4,              // always suspend, even if the task is already complete
}
```

- **`SuppressThrowing`** replaces the `try { await t; } catch { }` idiom when you genuinely want to wait for completion without caring about the outcome — the canonical case being awaiting a task you have already timed out on, purely so it does not become an unobserved exception. Note that it marks the exception as observed, which is exactly what you want there.
- **`ForceYielding`** is `Task.Yield()` fused with an await: "suspend here no matter what." Use it to guarantee you return to the caller before continuing (breaking a synchronous-completion loop, or ensuring you do not run arbitrary continuation work inline on a sensitive thread).

Being able to name these is a small but effective currency signal — they are recent, useful, and almost nobody mentions them.

---

## Concept 26 — The ThreadPool: two queues and work stealing

The .NET ThreadPool is a **single, process-wide, shared** resource. Its structure:

- **One global queue** (FIFO). Work queued from a non-pool thread — `Task.Run` from your `Main`, a timer callback, an I/O completion — lands here.
- **One local queue per worker thread** (LIFO). Work queued *from a pool thread* goes to that thread's local queue. LIFO order is deliberate: the most recently queued item is the most likely to have its data in cache, and for a recursively decomposed workload, taking the newest item keeps the working set small.
- **Work stealing.** A worker with an empty local queue checks the global queue, then *steals* from the tail of other workers' local queues. This is the standard Cilk-style design and it is what makes fine-grained task decomposition viable.

Implementation facts worth carrying:

- Since .NET 6, the **portable (fully managed) ThreadPool** is the default on all platforms, including Windows. It was introduced in .NET Core 3.0 and made the default in .NET 6; the old native Windows pool is gone. One consequence is that the thread-injection behaviour is now the same across platforms.
- **`ThreadPool.UnsafeQueueUserWorkItem(IThreadPoolWorkItem, preferLocal)`** lets high-performance code queue work without an `ExecutionContext` capture and without a delegate allocation. This is how Kestrel and the socket stack dispatch.
- **`ThreadPool.ThreadCount`, `ThreadPool.PendingWorkItemCount`, `ThreadPool.CompletedWorkItemCount`** are free, cheap, in-process telemetry. Expose them.

**The architectural point:** there is one pool, and *everything* uses it — your request handlers, every library's continuations, timer callbacks, logging sinks flushing asynchronously, the GC's background work items, EF Core's internals. Treating it as an infinite free resource is how a single blocking call in one component takes down an unrelated one. This is the in-process version of the noisy-neighbour problem, and the bulkhead answer (Module 13) applies: isolate genuinely different workloads onto their own execution resources.

---

## Concept 27 — Thread injection: `MinThreads` and the 500 ms rule

This is the mechanism behind the most common .NET latency outage, and you should be able to narrate it exactly.

The pool has two knobs: **`MinThreads`** (default: the processor count, separately for worker and I/O completion threads) and **`MaxThreads`** (a large number, in the thousands). The behaviour:

1. **Up to `MinThreads`, threads are created on demand, immediately, with no throttling.** If you have 8 cores and 8 work items arrive, you get 8 threads instantly.
2. **Beyond `MinThreads`, injection is throttled.** The starvation-avoidance heuristic adds roughly **one thread per 500 ms** when it sees queued work making no progress. (.NET 6's portable pool ramps more aggressively than the old one in some scenarios, but the order of magnitude — around one to two threads per second — is the number to reason with.)
3. **A hill-climbing heuristic** separately tries to find the thread count that maximizes throughput, adding and removing threads and measuring completion rate. Its goal is to *minimize* threads, which is exactly wrong when the threads are blocked rather than busy.
4. Idle threads above the minimum are retired after a period.

**Now do the arithmetic.** An 8-core service, `MinThreads = 8`, suddenly needs 100 threads because a dependency slowed down and the code blocks:

```
(100 − 8) threads × 0.5 s/thread ≈ 46 seconds to reach the needed thread count
```

Forty-six seconds during which the queue grows, latency climbs to seconds, clients time out, retries multiply the load (Module 13's retry-storm), the load balancer's health probe times out, and the instance is pulled from rotation. **The blocking call is the cause; the 500 ms injection rate is the multiplier that turns a slow dependency into an outage.**

**How to set it:**

```xml
<!-- .csproj — the modern, in-app way -->
<PropertyGroup>
  <ThreadPoolMinThreads>64</ThreadPoolMinThreads>
</PropertyGroup>
```

```jsonc
// runtimeconfig.template.json
{ "configProperties": { "System.Threading.ThreadPool.MinThreads": 64 } }
```

or `ThreadPool.SetMinThreads(64, 64)` at startup, or `DOTNET_ThreadPool_MinThreads` as an environment variable.

**The senior framing:** raising `MinThreads` is **a mitigation, not a fix**. It buys you a faster ramp so the blocking code doesn't produce a cliff; it does not make the blocking code correct, and it costs memory and context switches. State it that way: *"I'd raise `MinThreads` immediately to stop the bleeding, then find and remove the blocking call, then lower it back."* Legitimate long-term reasons to keep it raised: a genuinely blocking dependency you cannot replace (an old native driver, a synchronous-only SDK), or a very bursty arrival pattern where the ramp itself is the problem.

---

## Concept 28 — ThreadPool starvation: the mechanism and the signature

**The mechanism.** Pool threads are occupied by work that is *blocked* rather than *running*. Because they are blocked, they cannot dequeue new work items. Because new work keeps arriving, the queue grows. The pool injects threads at one per 500 ms; each new thread picks up a work item that also blocks; the pool never catches up. If the blocked work is waiting on something that itself needs a pool thread to complete, the system is *deadlocked*, not merely slow.

**The signature — memorize this, it is the diagnostic payoff:**

| Symptom | Value |
|---|---|
| CPU utilization | **Low** (10–30%) — the threads are blocked, not working |
| Latency | Very high and climbing, often in a staircase pattern |
| `ThreadPool.ThreadCount` | Climbing slowly and monotonically, far above core count |
| `ThreadPool.PendingWorkItemCount` / queue length | Large and growing |
| Thread stacks | Dozens of threads all parked in `Monitor.Wait`, `ManualResetEventSlim.Wait`, `Task.Result`, `SemaphoreSlim.Wait`, or a synchronous driver call |
| Recovery | Does not self-recover under sustained load; recovers instantly when load stops |

**Low CPU plus high latency plus a growing thread count is starvation until proven otherwise.** The contrast is worth stating alongside: *high* CPU plus high latency is a capacity or algorithmic problem; high GC time plus high latency is Module 14; low CPU plus high latency plus flat thread count is usually a downstream dependency, not you.

**The causes, in rough order of frequency:**

1. `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` on a pool thread — sync-over-async.
2. A synchronous blocking API called from an async path (a legacy driver, `File.ReadAllText` on a slow mount, a synchronous HTTP client, `Thread.Sleep`).
3. `lock`/`Monitor` contention on a hot shared resource — many threads parked on one lock.
4. `Task.Run(...).Wait()` — the classic double sin: burns two threads and blocks one of them.
5. A `SemaphoreSlim.Wait()` (synchronous) instead of `WaitAsync()`.
6. Enormous unbounded fan-out where every branch takes a pool thread.

**The fix hierarchy:** remove the block (best) → move the genuinely-blocking work onto dedicated threads outside the pool (acceptable) → raise `MinThreads` (mitigation) → add more instances (expensive mitigation that often makes the dependency worse).

---

## Concept 29 — I/O completion: there is no "IO thread pool" to tune

A persistent piece of .NET Framework-era folklore is that you tune "worker threads" and "IO threads" separately, and that async I/O runs on a distinct pool.

The accurate modern picture:

- `ThreadPool.SetMinThreads(worker, completionPort)` still has two parameters, and on Windows the completion-port value still influences the IOCP-bound threads. But in .NET Core's portable pool, **completions are dispatched as ordinary work items onto the same worker pool.**
- On Linux, the socket engine runs a small fixed number of `epoll` threads whose only job is to detect readiness and queue the real work — again, to the same pool.
- **The practical consequence:** your async continuations compete with your `Task.Run` work, your timer callbacks, and your request dispatch for the same threads. There is no separate reservoir for I/O. Starving the pool starves your I/O completions too, which is why starvation is self-reinforcing: the responses you're waiting for cannot be processed because the threads that would process them are blocked waiting for responses.

That self-reinforcement is the sentence that shows you understand the failure mode rather than just naming it.

---

## Concept 30 — Blocking on async code

The four spellings, and what each costs:

```csharp
var x = SomethingAsync().Result;                      // blocks; wraps exceptions in AggregateException
SomethingAsync().Wait();                              // blocks; wraps exceptions in AggregateException
var y = SomethingAsync().GetAwaiter().GetResult();    // blocks; rethrows the original exception
Task.Run(() => SomethingAsync()).GetAwaiter().GetResult(); // blocks a thread AND consumes another
```

All four:

1. **Occupy a ThreadPool thread for the full duration of the operation** — exactly the cost async existed to avoid.
2. **Convert your service from ~5 threads to ~100 threads** under the Little's Law arithmetic of Concept 5.
3. **Can deadlock** wherever a single-threaded `SynchronizationContext` exists (Concept 31).
4. **Are contagious in reverse**: one blocking call at the top of the stack negates every async call beneath it.

`GetAwaiter().GetResult()` is the least-bad spelling because the exception behaviour is sane, and it is the one to use in the few legitimate cases:

- **`Main` in an old-style entry point** — though `async Task Main` has existed since C# 7.1 and is the right answer.
- **A constructor or property that genuinely cannot be async** — usually a design smell pointing at a missing async factory method.
- **`IDisposable.Dispose` needing to complete async cleanup** — the actual answer is `IAsyncDisposable` (Module 14, Concept 40).
- **Test setup in a framework that doesn't support async** — modern xUnit/NUnit/MSTest all do.

**The one honest exception:** a console application or CLI tool with no concurrency, where there is no pool to starve and no context to deadlock on. Even there, `async Main` costs nothing.

---

## Concept 31 — The classic deadlock, and why ASP.NET Core doesn't have it

The canonical repro, which you should be able to draw on a whiteboard:

```csharp
// WPF / WinForms / classic ASP.NET
public void Button_Click(object sender, EventArgs e)
{
    var result = GetDataAsync().Result;   // (1) UI thread BLOCKS here
    label.Text = result;
}

private async Task<string> GetDataAsync()
{
    await _http.GetStringAsync(url);      // (2) captures the UI SynchronizationContext
    return "done";                        // (3) needs the UI thread to resume — which is blocked at (1)
}
```

Step by step: the UI thread calls `GetDataAsync`, which runs to the `await` and returns an incomplete task. The UI thread then blocks on `.Result`. When the HTTP call completes, the continuation is posted to the UI `SynchronizationContext`, which can only run it on the UI thread — which is blocked, forever. **Deadlock: a circular wait between a thread and its own continuation.**

The same shape occurs with classic ASP.NET, where the context permits one thread at a time per request.

**Why ASP.NET Core is immune:** no `SynchronizationContext`, so the continuation goes to the ThreadPool, which has other threads. The block resolves.

**Why this is not license to block:** the deadlock was the *loud* symptom. Removing it left the *quiet* symptom — the burned thread — fully intact. Under load, the quiet symptom becomes starvation (Concept 28), which is harder to diagnose than a deadlock. A senior answer says this explicitly: *"ASP.NET Core removed the deadlock, not the cost. The failure mode moved from an obvious hang in dev to a subtle latency cliff in production."*

**Two further traps in the same family:**

- **A single-threaded context can be reintroduced by a test framework.** xUnit v2 installs a context for async tests, which is why a `.Result` that works in production can deadlock in a test — and why "it deadlocks only in tests" is a real bug report.
- **A `SemaphoreSlim` with `Wait()` inside an async pipeline** produces a starvation-shaped deadlock without any `SynchronizationContext` at all: N pool threads hold or wait on the semaphore while the work that would release it sits in the pool queue.

---

## Concept 32 — `Task.Run`: when it helps and when it is cargo cult

**Legitimate uses:**

1. **Moving CPU-bound work off a thread you must not block** — a UI thread, or a protocol loop.
2. **Parallelizing CPU-bound work** deliberately: `await Task.WhenAll(chunks.Select(c => Task.Run(() => Process(c))))`.
3. **Offloading a genuinely blocking, short call** in an async method when you have no async alternative, and you have accepted that this consumes a pool thread.

**Illegitimate uses, all common:**

| Anti-pattern | Why it's wrong |
|---|---|
| `public Task<T> GetAsync() => Task.Run(() => GetSync());` in a library | Fake async. You've moved the block, not removed it, and you've hidden that fact from callers who now believe they're scalable. Stephen Cleary's "async over sync" — the library should expose the sync method and let the *caller* decide |
| `await Task.Run(() => SomethingAsync())` | Two scheduling hops to accomplish nothing; the inner method was already async |
| `Task.Run` around each item in an async fan-out | Burns a pool thread per item for work that doesn't need one |
| `Task.Run` in an ASP.NET Core request handler to "speed it up" | The request is already on a pool thread. You've added a hop and taken a *second* thread from the same pool. Throughput goes down |

**The rule that covers all of it:** `Task.Run` moves work to a ThreadPool thread. Ask "does this work need a thread, and is the thread it currently has one I must protect?" In a server, the answer to the second half is almost always no — you are already on a pool thread, and moving to another pool thread is pure loss.

**Prefer `Task.Run` over `Task.Factory.StartNew`** in all normal cases: `StartNew` uses `TaskScheduler.Current` (Concept 22), allows child attachment, and — the notorious one — returns `Task<Task>` for an async lambda, so the outer task completes when the lambda *starts* the inner one. `Task.Run` unwraps automatically. `StartNew` is only correct when you specifically need `TaskCreationOptions` such as `LongRunning`.

---

## Concept 33 — Dedicated threads and long-running work

Some work should not go on the ThreadPool at all:

- **Work that blocks for a long time by design**: a native driver's blocking loop, a serial port reader, a blocking message-queue consumer with no async API.
- **Work that runs for the process lifetime**: a dedicated dispatcher, an event loop.
- **Work with thread affinity**: STA COM objects, some native libraries.
- **Work that must not be preempted by pool scheduling policy**: a real-time-ish audio or market-data loop where you want priority control.

The options:

```csharp
// 1. A real dedicated thread (the right answer for long-lived blocking loops)
var t = new Thread(RunLoop) { IsBackground = true, Name = "market-data-reader" };
t.Start();

// 2. TaskCreationOptions.LongRunning — a hint that usually creates a dedicated thread
//    (implementation detail of TaskScheduler.Default, not a guarantee)
Task.Factory.StartNew(RunLoop, CancellationToken.None,
    TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
    TaskScheduler.Default);
```

Two details that come up:

- **`IsBackground = true`** means the thread does not keep the process alive. Forgetting it is a classic "my console app won't exit" bug; setting it when the thread has cleanup to do is a classic "my work got killed mid-write" bug. Choose deliberately and pair it with cooperative shutdown.
- **`LongRunning` is a hint.** The default scheduler currently honours it by creating a dedicated thread, but that is not contractual. For something you truly need off the pool, create a `Thread`.

**In ASP.NET Core**, the shape for "a long-lived loop" is a `BackgroundService` (Concept 67), whose `ExecuteAsync` runs on the pool — so if your loop *blocks*, you still need a dedicated thread inside it.

---

# Part D — The memory model

## Concept 34 — Why a memory model exists

Three layers are allowed to reorder your code, and each has its own reason:

1. **The C# compiler** may reorder or eliminate operations that are unobservable within a single thread.
2. **The JIT** does the real work here: register allocation (a field read hoisted into a register and never re-read), loop-invariant code motion, dead store elimination, instruction scheduling.
3. **The CPU** executes out of order, has store buffers, and has per-core caches with a coherence protocol. Even after the compiler emits instructions in order, the *visible effect order* to other cores can differ.

Every layer preserves **single-threaded semantics**: a program observing only its own thread cannot tell. Across threads, all bets are off unless you asked for a guarantee.

The canonical demonstration:

```csharp
class Worker
{
    private bool _stop;                                  // NOT volatile

    public void Run()
    {
        while (!_stop) { DoWork(); }                     // JIT may hoist: if (!_stop) while(true) DoWork();
    }

    public void Stop() => _stop = true;                  // may never be observed by Run()
}
```

This loop can run forever in Release mode. The JIT is entitled to read `_stop` once into a register and never re-read it, because within `Run`'s own thread nothing writes it. This is not a hypothetical — it is a classic, reproducible bug, and it is the minimal example that proves a memory model is necessary.

The fix is any of: `volatile bool _stop`, `Volatile.Read(ref _stop)`, reading it under a `lock`, or using a `CancellationToken` (which does this correctly for you, and is the right answer in modern code).

---

## Concept 35 — The .NET memory model: what is actually guaranteed

There are two documents and you should know they differ:

- **ECMA-335** specifies a deliberately weak model, because it must accommodate hypothetical implementations.
- **The .NET (CoreCLR) memory model** — documented in `docs/design/specs/Memory-model.md` in the `dotnet/runtime` repository — is **stronger than ECMA-335**, and is what actually holds on the runtime you ship. This document is relatively recent, authoritative, and almost nobody in an interview has read it. Reading it is disproportionately high-value.

The guarantees that matter in practice:

| Guarantee | Detail |
|---|---|
| **Atomicity** | Reads and writes of reference types, and of aligned primitives up to the native word size, are atomic — they never tear |
| **No introduced writes** | The runtime will not speculatively write to a location your program didn't write to (this is stronger than C/C++ historically allowed) |
| **No introduced reads that create races** | A read cannot be duplicated in a way that introduces a data race the source didn't have |
| **Object construction** | Publishing a reference to a newly constructed object is a **release** with respect to that object's field writes — so a reader that sees the reference sees the initialized fields. This is why the classic Java "unsafe publication" problem is *not* a problem on CoreCLR |
| **`volatile` write** | Release semantics: no prior memory operation moves after it |
| **`volatile` read** | Acquire semantics: no subsequent memory operation moves before it |
| **`Interlocked` operations** | Full fence; sequentially consistent with respect to other `Interlocked` operations |
| **`lock` acquire/release** | Acquire on enter, release on exit — a lock is a memory barrier at both ends |

**The hardware dimension, which is where career-defining bugs live:**

- **x86/x64 is Total Store Order (TSO).** Loads are not reordered with loads, stores are not reordered with stores, and loads are not reordered with earlier stores. **The only reordering permitted is store-then-load.** This means that on x64, a huge amount of subtly-incorrect lock-free code appears to work perfectly.
- **Arm64 is weakly ordered.** Loads and stores can be reordered fairly freely; the hardware needs explicit barriers (or acquire/release load/store instructions) to prevent it.

The consequence, and the sentence to say: **"A missing `volatile` or barrier is usually invisible on x64 and reproducible on Arm64. Since we run on Graviton/Ampere/Apple Silicon now, latent memory-model bugs from the x86-only era are surfacing."** That is current, specific, and demonstrates you understand why this topic became relevant again rather than being 2005 trivia.

---

## Concept 36 — Atomicity and tearing

**Atomic** (never torn) on a 64-bit runtime:

- All reference types
- `bool`, `byte`, `sbyte`, `char`, `short`, `ushort`, `int`, `uint`, `float`
- `long`, `ulong`, `double`, `IntPtr`, `nint` — **on 64-bit**, when naturally aligned
- A single-field struct wrapping one of the above

**Not atomic — can tear:**

- `long`/`double` on a **32-bit** runtime (two 32-bit stores; a reader can see half of each value)
- `decimal` (128 bits, four words)
- Any multi-field struct — `Guid`, `DateTime` + `TimeSpan` pairs, a `(int, int)` tuple, `Span`-like two-field structs
- Array elements are exactly as atomic as their element type; `Span.CopyTo` and `Array.Copy` offer **no** atomicity across elements

The practical consequences:

```csharp
private Guid _current;                          // 16 bytes — a reader can see half of an old and half of a new Guid
private (long Count, long Sum) _stats;          // torn under concurrent update

// Correct approaches:
private volatile State _state;                  // immutable class; swap the reference atomically
private long _count;                            // Interlocked.Increment(ref _count)
```

**The idiomatic fix for "atomic multi-field state" is an immutable object plus an atomic reference swap** — write the new state into a fresh object, then `Interlocked.Exchange` or `Volatile.Write` the reference. The reference write is atomic, so readers see either the whole old state or the whole new state. This is Concept 55's pattern, and it is the concurrency technique most worth having in your back pocket: it converts a hard synchronization problem into a trivially correct one, at the cost of an allocation per update.

`Interlocked.Read(ref long)` exists specifically for reading a `long` atomically on 32-bit. On 64-bit it is unnecessary but harmless.

---

## Concept 37 — `volatile` and `Volatile.Read`/`Write`

`volatile` on a field means: every read is an **acquire** and every write is a **release**. In practice:

- It prevents the JIT from caching the field in a register (the Concept 34 bug).
- It prevents reordering *across* the access in one direction each.

What it does **not** do — this is the part people get wrong:

| Myth | Reality |
|---|---|
| "`volatile` makes operations atomic" | No. `volatileField++` is still read-modify-write and still races. Use `Interlocked.Increment` |
| "`volatile` is a lightweight lock" | No. It orders individual accesses; it does not make a multi-step operation indivisible |
| "`volatile` gives sequential consistency" | No. Acquire/release is weaker than a full fence; a `volatile` write followed by a `volatile` read of a different field can still be reordered (the store–load case) |

**Restrictions to know:** the C# `volatile` keyword may **not** be applied to `long`, `ulong`, `double`, or `decimal`. For those, use `Volatile.Read<T>`/`Volatile.Write<T>` or `Interlocked`.

`Volatile.Read(ref x)` and `Volatile.Write(ref x, v)` are the general form — they apply the same semantics at a specific access rather than to every access of a field. That is usually preferable: most fields are accessed in one hot place where the ordering matters and many cold places where it does not, and marking the field `volatile` pessimizes all of them.

**When to reach for `volatile` at all:** a simple flag read in a loop and written once (the stop flag); a lazily-initialized reference in a double-checked lock; a publish/subscribe of an immutable snapshot. Almost everything else is better served by `Interlocked`, a `lock`, a `CancellationToken`, or a concurrent collection. A senior answer includes that judgement: *"`volatile` is correct for a narrow set of publication patterns, and reaching for it usually means I should have reached for something higher-level."*

---

## Concept 38 — `Interlocked` and compare-and-swap

`Interlocked` is a full memory barrier plus an atomic read-modify-write, implemented with `lock`-prefixed instructions on x86 or LL/SC (`ldxr`/`stxr`) on Arm.

The surface:

```csharp
Interlocked.Increment(ref _count);            // returns the NEW value
Interlocked.Decrement(ref _count);
Interlocked.Add(ref _total, 42);
Interlocked.Exchange(ref _ref, newValue);     // returns the OLD value
Interlocked.CompareExchange(ref _ref, newValue, comparand);  // returns the OLD value
Interlocked.Or(ref _flags, mask);             // .NET 7+
Interlocked.And(ref _flags, mask);            // .NET 7+
Interlocked.Read(ref _long);                  // atomic 64-bit read on 32-bit platforms
```

**`CompareExchange` is the primitive everything else is built from.** The universal lock-free update loop:

```csharp
// Atomically apply an arbitrary transform to a reference or value
private State _state;

public void Update(Func<State, State> transform)
{
    State original, updated;
    do
    {
        original = Volatile.Read(ref _state);
        updated  = transform(original);              // must be pure: it may run many times
    }
    while (Interlocked.CompareExchange(ref _state, updated, original) != original);
}
```

Read this carefully, because the two failure modes are interview material:

1. **The transform must be side-effect free.** Under contention it runs repeatedly. A transform that logs, increments a counter, or mutates anything will do so N times.
2. **The loop is unbounded under contention.** It is lock-free (the system as a whole makes progress) but not wait-free (an individual thread can be starved). Under heavy contention this can be *slower* than a plain `lock`, because every failed attempt wasted a full transform plus a cache-line bounce.

**The ABA problem** belongs here: `CompareExchange` compares the *value*, not the history. If a reference goes A → B → A between your read and your CAS, the CAS succeeds even though the world changed. For reference types on .NET the GC makes the classic pointer-reuse form rare, but the logical form (a version counter that wrapped, an index reused) is real. The standard mitigation is a version-tagged value — pack a counter alongside the value and CAS both together.

**Where `Interlocked` is genuinely the right tool:** counters and metrics, flags, single-reference publication, and building the fast path of something you have measured to be lock-contended. Not: multi-field invariants, anything requiring more than one CAS to be consistent, or anything where a `lock` was not measured to be a problem.

---

## Concept 39 — Double-checked locking, and why you should use `Lazy<T>` instead

The classic pattern, written correctly:

```csharp
private volatile Singleton? _instance;                 // volatile is REQUIRED
private readonly Lock _gate = new();                   // System.Threading.Lock, .NET 9+

public Singleton Instance
{
    get
    {
        var local = _instance;                         // volatile read (acquire)
        if (local is null)
        {
            lock (_gate)
            {
                local = _instance;
                if (local is null)
                {
                    local = new Singleton();
                    _instance = local;                 // volatile write (release)
                }
            }
        }
        return local;
    }
}
```

Why the `volatile` is required in principle: without a release on the write, another thread could observe a non-null `_instance` pointing at an object whose constructor had not finished. On CoreCLR specifically, the memory model's object-construction guarantee (Concept 35) makes this particular hazard not occur — but writing it without `volatile` is still wrong as portable code and wrong as a signal in an interview, because you would be relying on an implementation guarantee rather than the contract.

**The actual answer is not to write it at all:**

```csharp
private readonly Lazy<Singleton> _instance =
    new(() => new Singleton(), LazyThreadSafetyMode.ExecutionAndPublication);
```

`Lazy<T>` modes, which are worth knowing because the default differs by constructor:

- **`ExecutionAndPublication`** — the factory runs exactly once; other threads block. This is the default for the thread-safe constructors and is what you almost always want.
- **`PublicationOnly`** — the factory may run on several threads concurrently; the first to finish wins and the others' results are discarded. Use when the factory is cheap and idempotent and you'd rather not block.
- **`None`** — not thread-safe.

Also in this family: **`LazyInitializer.EnsureInitialized`** (no `Lazy<T>` object allocation, for when you have many fields), and, for the common "cache per key" case, `ConcurrentDictionary<K, Lazy<V>>` — which is the idiomatic way to get exactly-once factory execution per key, given that `GetOrAdd`'s factory can run more than once (Concept 53).

---

## Concept 40 — False sharing

Cores do not synchronize individual variables — they synchronize **cache lines**, which are 64 bytes on essentially every modern CPU (128 on some Apple Silicon). If two threads write to two *different* variables that happen to sit in the same cache line, every write invalidates the other core's copy. The variables are logically independent; the hardware treats them as one. Throughput can fall by an order of magnitude with no visible cause in the code.

```csharp
// PATHOLOGICAL: 8 counters share one or two cache lines
private readonly long[] _counters = new long[8];   // 64 bytes total — all on one line
// thread i does: Interlocked.Increment(ref _counters[i]);
```

The fixes:

```csharp
// 1. Pad so each counter owns a line
[StructLayout(LayoutKind.Explicit, Size = 128)]
private struct PaddedLong { [FieldOffset(0)] public long Value; }
private readonly PaddedLong[] _counters = new PaddedLong[8];

// 2. Stride the indices
private readonly long[] _counters = new long[8 * 8];   // use index i * 8

// 3. Better: don't share at all — per-thread accumulation, combined at read time
```

Where this shows up in real .NET code: per-core counters and metrics, striped-lock arrays, ring buffer head/tail indices (which is why the BCL's own `ConcurrentQueue` uses a padded head-and-tail struct), and any "array of per-worker state." Where it does *not* show up: anything that isn't being written concurrently at high frequency. False sharing is a scaling problem, not a correctness problem, and it only matters on genuinely hot paths.

**Mentioning `Environment.SystemPageSize`, cache line sizes, and the padded-struct pattern unprompted is a strong signal** in a performance-oriented round, because it means you have actually profiled a scaling problem rather than read about one.

---

## Concept 41 — When lock-free is worth it

Lock-free programming trades a well-understood blocking primitive for a proof obligation. The honest cost/benefit:

**Arguments for:**
- No lock convoy, no priority inversion, no deadlock.
- A thread suspended mid-operation (by the OS, or by a GC pause) does not block others.
- Lower latency variance on hot, short operations.

**Arguments against:**
- Correctness is genuinely hard: ABA, memory ordering, and the fact that testing cannot prove absence of a race.
- Under **high** contention, CAS loops can be slower than a lock, because every failure is a wasted attempt plus a cache-line transfer, whereas a lock parks the loser and stops the bouncing.
- The code is unreviewable by most of the team, which is an organizational cost an architect should weigh explicitly.

**The decision procedure to state out loud:**

1. Is there measured contention? (`lock-contention-count` counter, or a profiler showing time in `Monitor.Enter`.) If not — stop, use a `lock`.
2. Can you eliminate the sharing instead? Per-thread state, partitioning by key, or an immutable snapshot swap solves more real problems than lock-free algorithms do.
3. Can you use a BCL type that already solved it? `ConcurrentDictionary`, `Channel<T>`, `Interlocked` counters, `ImmutableList<T>`.
4. Only then, and only for a small, well-tested, well-documented core, write the CAS loop.

The senior signal here is **restraint plus capability**: being able to write the CAS loop and explaining why you wouldn't. "We used `Interlocked` for the counter because it was measurably contended, and kept a plain `lock` for the state transition because it wasn't" is exactly the shape of a good answer.

---

# Part E — Synchronization primitives

## Concept 42 — `lock` and `Monitor`: thin locks, inflation, reentrancy

`lock (obj) { ... }` compiles (pre-.NET 9 semantics) to:

```csharp
bool taken = false;
try { Monitor.Enter(obj, ref taken); ... }
finally { if (taken) Monitor.Exit(obj); }
```

The mechanics, which connect directly to Module 14's object layout:

- **The thin lock lives in the object header** (Module 14, Concept 3). Acquiring an uncontended lock is a CAS on the header writing the thread's ID plus a recursion count — a handful of nanoseconds, no kernel involvement.
- **Contention inflates the lock**: the runtime allocates a real **sync block** in a side table, and the header's sync-block index points at it. The sync block holds the wait list and the kernel event used for blocking waits.
- **Before parking, the runtime spins briefly**, because most critical sections are short and a spin is cheaper than two context switches.
- **`Monitor` is reentrant** (a thread can re-enter a lock it holds; a recursion count tracks depth) and **thread-affine** (only the owning thread may exit).
- **`Monitor.Wait`/`Pulse`/`PulseAll`** implement condition-variable semantics against the same sync block. They are correct but very easy to misuse; in modern code, `SemaphoreSlim`, `Channel<T>`, or a `TaskCompletionSource` is almost always clearer.

**The rules that get tested:**

1. **Never `lock` on `this`, on a `Type`, or on a `string`.** Any of these can be locked by code you don't control: `this` by your caller, `typeof(X)` by anyone in the process, and interned string literals globally (Module 14, Concept 18). Use a private, dedicated object — or, from .NET 9, a `Lock`.
2. **Never lock on a value type** — it gets boxed, producing a different object each time, so the lock protects nothing. The compiler warns.
3. **Keep the critical section short and call nothing unknown inside it.** Calling a virtual method, an event handler, or a callback while holding a lock is how you get a deadlock you cannot see in your own code.
4. **Never block on I/O inside a lock**, and never `await` inside one (Concept 44).

---

## Concept 43 — `System.Threading.Lock` (.NET 9)

.NET 9 introduced a dedicated lock type, and C# 13 special-cases it so `lock (aLockInstance)` compiles to `using (aLockInstance.EnterScope())` rather than to `Monitor.Enter`.

```csharp
private readonly Lock _gate = new();

public void Mutate()
{
    lock (_gate) { /* ... */ }            // compiles to _gate.EnterScope()
}

// Explicit form, and the try variant
using (_gate.EnterScope()) { /* ... */ }

if (_gate.TryEnter(TimeSpan.FromMilliseconds(50)))
{
    try { /* ... */ } finally { _gate.Exit(); }
}
```

Why it is better than `lock (new object())`:

- **Purpose-built and faster.** It does not interact with the object header or the sync-block table at all; it is a dedicated managed implementation with its own spin/park logic.
- **Intent is expressed in the type.** A field of type `Lock` cannot be mistaken for a data object, and you cannot accidentally lock on something shared.
- **`EnterScope()` returns a `ref struct`**, which means it cannot escape to the heap, cannot be held across an `await`, and cannot be captured in a closure — the type system now enforces what used to be a convention (Module 14, Concept 19).
- **Analyzers catch misuse.** Converting a `Lock` to `object` (which would silently fall back to monitor-based locking on the `Lock` instance itself) produces a warning. That warning is the whole reason the language change was needed.
- It exposes **`IsHeldByCurrentThread`**, which is genuinely useful for debug assertions.

**Migration guidance to offer:** changing `private readonly object _gate = new();` to `private readonly Lock _gate = new();` is source-compatible with existing `lock` statements and is a free improvement on .NET 9+. Two caveats: it is still thread-affine, so the `await` prohibition is unchanged; and code that passes the lock object to an API taking `object` (or stores it in a collection) will trip the analyzer, which is the analyzer doing its job.

---

## Concept 44 — Why you cannot `await` inside `lock`

The compiler refuses to compile `await` inside a `lock` body. The reason is precise and worth stating exactly:

**`Monitor` is thread-affine: only the thread that entered may exit.** An `await` may resume on a different thread. So the generated `finally { Monitor.Exit(obj); }` would run on a thread that does not own the lock — which throws `SynchronizationLockException` at best, and corrupts the lock state at worst. Beyond that, holding a lock across an arbitrarily long I/O wait is a design error regardless of thread identity: you have serialized your entire system behind the slowest external call.

**The correct primitive is `SemaphoreSlim`**, which is not thread-affine:

```csharp
private readonly SemaphoreSlim _gate = new(1, 1);

public async Task<T> GetAsync(string key, CancellationToken ct)
{
    await _gate.WaitAsync(ct).ConfigureAwait(false);
    try
    {
        return await LoadAsync(key, ct).ConfigureAwait(false);
    }
    finally
    {
        _gate.Release();     // MUST be in finally; must be exactly once
    }
}
```

Three things to note about this pattern, all of which are where people get it wrong:

- **`Release()` must be in a `finally`**, or an exception permanently consumes a permit and the semaphore is eventually exhausted. This is a slow, silent hang — the hardest kind to diagnose.
- **`SemaphoreSlim` is not reentrant.** A method that takes the gate and calls another method that takes the same gate deadlocks immediately. `Monitor` would have allowed it. This trips people migrating from `lock`.
- **Pass the `CancellationToken` to `WaitAsync`**, or a shutdown will hang waiting for a gate.

A lighter-weight alternative worth knowing: if the critical section is short and purely in-memory, take a `lock` around *just that part* and keep the `await` outside it. Restructuring so the lock and the await do not overlap is usually better than reaching for a semaphore.

---

## Concept 45 — `SemaphoreSlim`: the async-capable gate

`SemaphoreSlim(initialCount, maxCount)` is the workhorse of async synchronization, and it plays two distinct roles:

1. **A mutex** — `new SemaphoreSlim(1, 1)` — the async `lock` replacement above.
2. **A concurrency limiter** — `new SemaphoreSlim(10, 10)` — "at most 10 of these at once." This is the bulkhead pattern from Module 13 in its simplest form, and the foundation of Concept 64.

```csharp
private readonly SemaphoreSlim _limiter = new(maxConcurrency);

async Task ProcessAllAsync(IEnumerable<Item> items, CancellationToken ct)
{
    var tasks = items.Select(async item =>
    {
        await _limiter.WaitAsync(ct).ConfigureAwait(false);
        try { await ProcessAsync(item, ct).ConfigureAwait(false); }
        finally { _limiter.Release(); }
    });
    await Task.WhenAll(tasks).ConfigureAwait(false);
}
```

Note what this *doesn't* solve: `items.Select(...)` still materializes a task per item, so a million items means a million state machines even though only `maxConcurrency` run at once. For large or unbounded inputs, `Parallel.ForEachAsync` (Concept 62) or a `Channel<T>` with N consumers (Concept 57) is the better shape.

Properties to remember: **not reentrant**, **not thread-affine** (any thread may `Release`), `CurrentCount` is a snapshot and racy, and `Wait()` (synchronous) on a pool thread is a starvation source — always `WaitAsync` in async code.

---

## Concept 46 — `ReaderWriterLockSlim`, and when it loses

The promise: many concurrent readers, one exclusive writer. The reality is that it is slower per-operation than `Monitor` (it maintains substantially more state), and the break-even point requires **long read sections and rare writes**. For short reads — a dictionary lookup, a field read — a plain `lock` almost always wins, because the reader-writer bookkeeping costs more than the critical section itself.

Details that matter if you do use it:

- **`LockRecursionPolicy.NoRecursion` is the default** and is the right choice; `SupportsRecursion` is slower and enables a class of bugs.
- **`EnterUpgradeableReadLock`** exists to avoid the check-then-write race without holding a write lock for the check. Exactly one upgradeable reader is allowed at a time.
- **Every `Enter` must be paired with an `Exit` in a `finally`.**
- **It is thread-affine and has no async support.** There is no `EnterReadLockAsync`. If you need reader/writer semantics in async code, use `ConcurrentExclusiveSchedulerPair`, or restructure.

**The alternatives that usually beat it:**

| Situation | Better than `ReaderWriterLockSlim` |
|---|---|
| Read-mostly dictionary | `ConcurrentDictionary` (lock-free reads) |
| Read-mostly snapshot of state | Immutable object + `Volatile.Write` reference swap (Concept 36) |
| Short critical sections | Plain `lock` |
| Async code | `SemaphoreSlim`, or `ConcurrentExclusiveSchedulerPair` |

Saying "I'd start with a plain `lock` and only consider `ReaderWriterLockSlim` if profiling showed reader contention with long read sections" is a better answer than reaching for it by default.

---

## Concept 47 — Cross-process primitives

`Mutex`, `Semaphore`, and `EventWaitHandle` can be **named**, which makes them OS kernel objects visible across processes on the same machine.

```csharp
using var mutex = new Mutex(initiallyOwned: false, name: "Global\\MyApp-Migration");
if (!mutex.WaitOne(TimeSpan.FromSeconds(30)))
    throw new TimeoutException("Another instance holds the migration lock.");
try { RunMigration(); }
finally { mutex.ReleaseMutex(); }
```

Legitimate uses: single-instance applications, guarding a machine-local resource (a file, a named pipe, an installer step).

The caveats are serious and are exactly the ones from Module 9:

- **Machine-scoped only.** Two pods on two nodes share nothing. In any containerized or scaled-out deployment, a named mutex provides an illusion of mutual exclusion.
- **`AbandonedMutexException`** is thrown to the next waiter if the owning process dies while holding it. That exception means "the protected state may be inconsistent" and must be handled, not caught and ignored.
- **Thread affinity.** `Mutex` must be released by the acquiring thread, so it has the same `await` problem as `Monitor`. `Semaphore` does not.
- **Named objects on Linux** work but have different namespace and permission semantics than on Windows; `Global\` prefixes and ACLs are Windows concepts.

For anything genuinely distributed, the answer is Module 9's: a lease with a TTL, a fencing token, and an acceptance that the lock can be lost mid-operation.

---

## Concept 48 — Events, countdowns, and barriers

The classic coordination primitives, listed mainly so you can recognize them and explain why they're rarely the right answer in async code:

| Primitive | Semantics | Modern async equivalent |
|---|---|---|
| `ManualResetEventSlim` | Stays signalled until reset; all waiters released | `TaskCompletionSource` (await the task) |
| `AutoResetEvent` | Releases exactly one waiter, then auto-resets | `SemaphoreSlim(0)` |
| `CountdownEvent` | Waits for N signals | `Task.WhenAll` |
| `Barrier` | N participants rendezvous each phase | Rare; `Task.WhenAll` in a loop, or a dataflow block |
| `Monitor.Wait`/`Pulse` | Condition variable | `Channel<T>`, `SemaphoreSlim`, `TaskCompletionSource` |

The "`Slim`" variants spin briefly before allocating and waiting on a kernel object, which makes them much cheaper for short waits. All of them **block a thread**, which is the disqualifying property in async code: every waiting thread is a ThreadPool thread you are not using.

The one to actually know well is **`ManualResetEventSlim` vs `TaskCompletionSource`**, because the migration is so common: any place where a thread waits for a one-time signal should become "await a task that someone completes."

---

## Concept 49 — `SpinLock` and `SpinWait`

Spinning burns CPU instead of parking the thread. It wins only when the expected wait is shorter than two context switches (roughly 1–10 µs) and there are free cores for the spinning thread to occupy.

**`SpinWait`** implements the correct escalation ladder, and knowing the ladder is the point:

1. Execute `Thread.SpinWait(n)` (a `pause`/`yield` instruction) with an increasing count.
2. Then `Thread.Yield()` — give up the rest of the time slice to another thread on the same core.
3. Then `Thread.Sleep(0)` — yield to threads of equal priority.
4. Then `Thread.Sleep(1)` — a real, ~1–15 ms sleep, which is the "we've given up spinning" case.

```csharp
var spin = new SpinWait();
while (Volatile.Read(ref _flag) == 0)
    spin.SpinOnce();
```

**`SpinLock`** is a struct (deliberately — so it can be embedded in an array of per-item locks without an allocation each) implementing a spin-based mutex. Its rules are strict: never hold it while blocking, never hold it across anything that can take another lock, never copy the struct, always use `ref` when passing it, and always use the `ref bool lockTaken` pattern with a `finally`.

**When these are right:** inside a lock-free data structure, guarding a handful of instructions in a very hot path, on a machine where you control core allocation. **When they are wrong:** basically everywhere else. On a container limited to 0.5 CPU, spinning is catastrophic — you are spending your entire quota on doing nothing. That container observation is a strong, current answer to "when would you not spin?"

---

## Concept 50 — The deadlock taxonomy

Five distinct mechanisms, each with its own fix:

| Type | Mechanism | Fix |
|---|---|---|
| **Lock-ordering deadlock** | Thread A holds L1 and wants L2; thread B holds L2 and wants L1 | Establish and document a global lock order; acquire in that order always; or take one coarser lock |
| **Sync-over-async deadlock** | A thread blocks on a task whose continuation needs that same thread (Concept 31) | Don't block; async all the way |
| **Pool-exhaustion deadlock** | All pool threads are blocked on work that needs a pool thread to complete | Don't block on a pool thread; bulkhead; raise `MinThreads` as mitigation |
| **Lock convoy** | Many threads contend on one lock; each acquisition costs a context switch; throughput collapses even though there is no true deadlock | Reduce the critical section, shard the lock, or eliminate sharing |
| **Async reentrancy deadlock** | A non-reentrant `SemaphoreSlim` re-entered by the same logical flow | Restructure so the gate is taken once; never call outward while holding it |

**Prevention, in order of effectiveness:**

1. **Don't share mutable state.** By far the most effective, and the one architects should push for: message passing (Module 11), immutable data, per-request state, partitioning by key.
2. **Take one lock at a time.** If you never hold two, ordering deadlocks are impossible.
3. **Never call unknown code while holding a lock** — no virtual methods, no callbacks, no events, no `await`.
4. **Use timeouts** (`Monitor.TryEnter(obj, timeout)`, `lock.TryEnter`, `SemaphoreSlim.WaitAsync(timeout)`) so a deadlock becomes a loud, diagnosable error instead of a silent hang. This is a design decision worth defending: it converts an unbounded hang into a bounded failure you can alert on.
5. **Document the lock hierarchy** where locks are unavoidable. An undocumented hierarchy is one refactor away from a 3 a.m. page.

**Diagnosis:** a managed dump plus SOS. `!syncblk` lists held monitors and their owning threads; `!clrstack` on each thread shows what it's waiting for; the cycle is then visible by inspection. For the async variants, `!dumpasync` (Concept 73) shows the pending async state machines and what they are waiting on.

---

## Concept 51 — Async coordination you build yourself

There is no `AsyncLock`, `AsyncManualResetEvent`, or `AsyncReaderWriterLock` in the BCL. The building blocks are `SemaphoreSlim`, `TaskCompletionSource`, and `Channel<T>`, and the standard compositions are:

```csharp
// Async manual-reset event
public sealed class AsyncManualResetEvent
{
    private volatile TaskCompletionSource _tcs = new(TaskCreationOptions.RunContinuationsAsynchronously);

    public Task WaitAsync() => _tcs.Task;
    public void Set() => _tcs.TrySetResult();
    public void Reset()
    {
        while (true)
        {
            var tcs = _tcs;
            if (!tcs.Task.IsCompleted ||
                Interlocked.CompareExchange(ref _tcs, new TaskCompletionSource(
                    TaskCreationOptions.RunContinuationsAsynchronously), tcs) == tcs)
                return;
        }
    }
}
```

```csharp
// Async lazy: exactly-once async initialization
private readonly Lazy<Task<Connection>> _connection =
    new(() => ConnectAsync(), LazyThreadSafetyMode.ExecutionAndPublication);

public Task<Connection> GetConnectionAsync() => _connection.Value;
```

Practical guidance:

- **Prefer the library.** `Nito.AsyncEx` (Stephen Cleary) provides `AsyncLock`, `AsyncManualResetEvent`, `AsyncAutoResetEvent`, `AsyncCountdownEvent`, `AsyncProducerConsumerQueue` — correct, tested implementations of exactly this. So does `Microsoft.VisualStudio.Threading` (`AsyncSemaphore`, `AsyncReaderWriterLock`, `JoinableTaskFactory`), which is battle-tested inside Visual Studio.
- **`RunContinuationsAsynchronously` everywhere**, for the Concept 9 reason.
- **Most of the time you don't need any of this.** A `Channel<T>` expresses "wait for work" better than an event does, and a `SemaphoreSlim` expresses "one at a time" better than an async lock built by hand.

---

## Concept 52 — Thread-safety is a documented contract

The single most avoidable class of concurrency bug comes from assuming thread-safety rather than reading it. The table every .NET engineer should have internalized:

| Type | Thread-safe? | Notes |
|---|---|---|
| `HttpClient` | **Yes** for concurrent requests | Designed to be shared; use `IHttpClientFactory` for handler lifetime and DNS refresh |
| `DbContext` (EF Core) | **No** | One per unit of work; concurrent use throws `InvalidOperationException` if you're lucky and corrupts state if you're not (Module 19) |
| `CosmosClient`, `ServiceBusClient`, `BlobServiceClient` | **Yes** | Azure SDK clients are explicitly designed as singletons (Module 7, Module 11) |
| `SqlConnection` | **No** | One per operation; the *pool* is thread-safe, the connection is not |
| `Random` | **No** | `Random.Shared` (.NET 6+) **is** thread-safe — use it |
| `StringBuilder` | **No** | |
| `List<T>`, `Dictionary<K,V>` | **No**, even for concurrent *reads with one writer* | Concurrent reads alone are safe; a concurrent write can corrupt internal state and cause infinite loops |
| `ConcurrentDictionary` etc. | **Yes** for individual operations | Compound operations still need care (Concept 53) |
| `ImmutableList<T>` etc. | **Yes**, inherently | |
| `ILogger` | **Yes** | The abstraction requires it; sinks are expected to be safe |
| Static mutable fields | **No**, by construction | The most common source of accidental sharing |
| `IMemoryCache` | **Yes** | But `GetOrCreate`'s factory is not exactly-once (same issue as Concept 53) |

**The senior behaviour** is to state the contract when designing a type: XML docs saying "instances are safe for concurrent use" or "instances are not thread-safe; use one per request," and matching DI lifetime registration. A singleton registration of a non-thread-safe type is the **captive dependency / shared-state** bug that Module 18 covers, and it is one of the most common real defects in .NET line-of-business code.

---

# Part F — Concurrent data structures and dataflow

## Concept 53 — `ConcurrentDictionary`: lock striping and the factory trap

**The internals**, which are worth knowing because they explain every behavioural quirk:

- The dictionary holds an array of buckets and a **separate array of locks** (the "stripes"). The default lock count is derived from the processor count. A write locks only the stripe for its bucket, so N writers to different buckets proceed in parallel.
- **Reads take no lock at all.** They are volatile reads of an immutable-once-published node, which is safe because of the object-construction release guarantee (Concept 35).
- **Resizing** takes all the locks.

**The behavioural consequences that get asked about:**

| Behaviour | Detail |
|---|---|
| `GetOrAdd(key, factory)` | The **factory may run more than once** under concurrent access for the same key, and it runs **outside the lock**. Only one result is stored and returned to everyone, but the others were computed and discarded |
| `AddOrUpdate` | Same: both delegates may run multiple times |
| `Count` / `IsEmpty` | Takes **all** locks — O(stripes), not free; avoid in hot paths |
| Enumeration | A moving snapshot: no lock, no exception, but no consistency guarantee either. You may see items added during enumeration, or not |
| `ToArray()` | Takes all locks; gives a true point-in-time snapshot |
| `TryRemove(key, out value)` | Atomic. `TryUpdate(key, newValue, comparisonValue)` is a CAS on the value |
| Value type values | Reads/writes of the value are not atomic if the value is a large struct — `TryUpdate` compares with `EqualityComparer<T>.Default` |

**The factory trap, and the fix.** If the factory is expensive (opens a connection, calls a service) or has side effects, running it twice is a real bug:

```csharp
// Broken: CreateExpensiveThing may run several times per key
var thing = _cache.GetOrAdd(key, k => CreateExpensiveThing(k));

// Correct: the Lazy is cheap to construct; only one Lazy wins;
// ExecutionAndPublication guarantees the factory runs exactly once
var thing = _cache.GetOrAdd(key, k => new Lazy<Thing>(
        () => CreateExpensiveThing(k), LazyThreadSafetyMode.ExecutionAndPublication))
    .Value;

// Async version
var thing = await _cache.GetOrAdd(key, k => new Lazy<Task<Thing>>(
        () => CreateExpensiveThingAsync(k))).Value;
```

This `ConcurrentDictionary<K, Lazy<V>>` idiom is genuinely worth memorizing — it is the standard solution to the cache-stampede problem in-process, it composes with async, and mentioning it unprompted is a strong signal. (Note the second-order concern: if the async factory throws, the faulted `Lazy<Task<V>>` is cached forever. Production versions remove the entry on failure.)

The `ConcurrentDictionary` constructor's `concurrencyLevel` parameter sets the number of stripes. Tuning it is almost never worthwhile; knowing what it means is.

---

## Concept 54 — `ConcurrentQueue`, `ConcurrentStack`, `ConcurrentBag`

| Type | Structure | Ordering | Use when |
|---|---|---|---|
| `ConcurrentQueue<T>` | Linked list of **segments** (arrays), lock-free via `Interlocked` on padded head/tail indices | FIFO | General producer/consumer; the default choice |
| `ConcurrentStack<T>` | Lock-free singly-linked list, CAS on head | LIFO | Object pools, depth-first work queues, better cache locality |
| `ConcurrentBag<T>` | **Thread-local** lists with work stealing | None | Only when the same thread usually adds and removes |

**`ConcurrentBag` is the one that surprises people.** Each thread has its own local list; `Add` pushes locally (cheap, no contention), `TryTake` pops locally, and only when the local list is empty does it *steal* from another thread's list (taking a lock, and from the opposite end). So:

- If thread A adds everything and thread B takes everything, **every single take is a steal** — the worst case. `ConcurrentBag` is dramatically slower than `ConcurrentQueue` in that shape.
- The right use case is a parallel algorithm where each worker both produces and consumes, with occasional stealing for load balancing. `Parallel.ForEach`'s internals are exactly this.
- It also retains per-thread lists, which is a memory consideration if threads are numerous or short-lived.

**Facts that apply across all three:** `Count` is O(n)-ish and racy; enumeration is a snapshot taken at the start; and — importantly — **none of them block.** `TryDequeue` returning `false` means "empty right now," and a consumer loop that spins on it burns CPU. That gap is exactly what `BlockingCollection<T>` and `Channel<T>` fill.

**None of them have an async API.** A consumer that needs to wait for an item should be using `Channel<T>` (Concept 57), not a concurrent collection plus a polling loop.

---

## Concept 55 — Immutable collections as a concurrency strategy

`System.Collections.Immutable` gives you persistent data structures: every "mutation" returns a new instance sharing most of its structure with the old one (`ImmutableList<T>` is an AVL tree; `ImmutableDictionary<K,V>` is a hash array mapped trie; `ImmutableArray<T>` is a real array and copies on every change).

The concurrency pattern this enables is the one from Concept 36, and it is under-used:

```csharp
private ImmutableDictionary<string, Route> _routes = ImmutableDictionary<string, Route>.Empty;

// Readers: completely lock-free, always see a consistent whole snapshot
public Route? Find(string key) =>
    Volatile.Read(ref _routes).TryGetValue(key, out var r) ? r : null;

// Writers: CAS loop on the reference
public void Add(string key, Route route)
{
    ImmutableDictionary<string, Route> original, updated;
    do
    {
        original = Volatile.Read(ref _routes);
        updated  = original.SetItem(key, route);
    }
    while (Interlocked.CompareExchange(ref _routes, updated, original) != original);
}
// or simply: ImmutableInterlocked.Update(ref _routes, (d, k) => d.SetItem(k, route), key);
```

**When this is the right answer:** read-heavy, write-rare configuration and routing tables; state that must be consistent across many fields at once; anything you want to hand to another component without worrying about it being mutated underneath.

**When it is not:** write-heavy workloads (each write allocates), large collections rebuilt frequently, or hot paths where the tree-walk lookup cost (O(log n) with worse constants than a hash table) matters.

`ImmutableArray<T>` deserves a separate note: it is a struct wrapping a `T[]`, so it has *zero* overhead for reads and is the correct type for a small, never-changing set. It is also the one where "mutation" is O(n), so never build one in a loop — use `ImmutableArray<T>.Builder` or `ToImmutableArray()` at the end. `ImmutableInterlocked` provides the CAS-loop helpers so you don't write them by hand. And note that `FrozenDictionary<K,V>`/`FrozenSet<T>` (.NET 8) are the better choice when the collection is built once and then only read: slower to construct, faster to query than either immutable or regular dictionaries.

---

## Concept 56 — `BlockingCollection<T>` and why `Channel<T>` replaced it

`BlockingCollection<T>` wraps an `IProducerConsumerCollection<T>` and adds blocking `Take()`, bounded capacity with blocking `Add()`, and `CompleteAdding()`/`GetConsumingEnumerable()`.

It works, it is correct, and it is the wrong tool in async code for one reason: **`Take()` blocks a thread.** A consumer pool of 10 workers means 10 threads parked whenever the collection is empty. In a server, those are ThreadPool threads, and Concept 28 follows.

It remains fine for a classic thread-based pipeline with dedicated (non-pool) threads — a desktop app, a batch tool, a producer/consumer built on `new Thread(...)`. In every server scenario, `Channel<T>` is the replacement, and the migration is mechanical:

| `BlockingCollection<T>` | `Channel<T>` |
|---|---|
| `new BlockingCollection<T>(capacity)` | `Channel.CreateBounded<T>(capacity)` |
| `Add(item)` (blocks when full) | `await writer.WriteAsync(item)` (suspends when full) |
| `Take()` (blocks when empty) | `await reader.ReadAsync()` (suspends when empty) |
| `CompleteAdding()` | `writer.Complete()` |
| `foreach (var x in GetConsumingEnumerable())` | `await foreach (var x in reader.ReadAllAsync())` |

---

## Concept 57 — `Channel<T>`: the in-process queue with backpressure

`System.Threading.Channels` is the modern answer to in-process producer/consumer, and it is worth knowing in detail because it is both a practical tool and a good vehicle for demonstrating that you think about backpressure.

```csharp
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(capacity: 1000)
{
    FullMode = BoundedChannelFullMode.Wait,   // the important decision — see below
    SingleReader = false,
    SingleWriter = false,
    AllowSynchronousContinuations = false     // keep the default
});

// Producer
await channel.Writer.WriteAsync(item, ct);    // suspends (does not block) when full

// Consumer — N of these run concurrently
await foreach (var item in channel.Reader.ReadAllAsync(ct))
    await ProcessAsync(item, ct);

// Shutdown
channel.Writer.Complete();                    // ReadAllAsync then drains and ends
```

**`BoundedChannelFullMode` is an architectural decision, not a config value.** It is the local version of the overload-protection choice from Module 6:

| Mode | Behaviour | Use when |
|---|---|---|
| `Wait` | The writer suspends until space frees up | **Real backpressure.** The producer must be slowable — e.g. it is reading from a queue that supports flow control |
| `DropOldest` | Evicts the oldest queued item | Latest-value-wins telemetry, live dashboards, market data |
| `DropNewest` | Evicts the newest queued item | Rare; when the earliest items matter most |
| `DropWrite` | Silently discards the incoming item | Best-effort logging, sampling |

**Unbounded channels are a design decision to explain, not a default to take.** `Channel.CreateUnbounded<T>()` means "the queue can grow until the process dies," which is an `OutOfMemoryException` waiting for a slow consumer (Module 14, Concept 56's "leak" pathology — except it is not even a bug, it is the documented behaviour). Bounded should be your default; unbounded needs a justification.

The options flags earn real performance: **`SingleReader = true`/`SingleWriter = true`** select specialized implementations with less synchronization, and setting them when true is free performance. **`AllowSynchronousContinuations = true`** lets a completing writer run the reader's continuation inline — faster, but it hands your producer thread to consumer code, which is the Concept 9 hazard. Leave it `false` unless you've measured and understood the consequence.

.NET 9 added **`Channel.CreateUnboundedPrioritized<T>()`**, which dequeues by priority rather than FIFO.

**The architectural framing to offer:** a `Channel<T>` is an in-process queue, so everything from Module 11 applies at a smaller scale — it decouples producer and consumer rates, it needs a bound, it needs a drop policy, and **it is not durable.** Items in a channel are lost on process restart. If the work must not be lost, the channel is a buffer in front of a durable queue, not a replacement for one. That last sentence is the one that distinguishes an architect's answer from an engineer's.

---

## Concept 58 — `IAsyncEnumerable<T>` and `await foreach`

Async streams (C# 8) give you asynchronous iteration with natural backpressure: the producer's `yield return` does not proceed until the consumer asks for the next item.

```csharp
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    DateOnly from,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var conn = new SqlConnection(_cs);
    await conn.OpenAsync(ct);
    await using var reader = await cmd.ExecuteReaderAsync(ct);

    while (await reader.ReadAsync(ct))
        yield return Map(reader);
}

// Consumption
await foreach (var order in StreamOrdersAsync(from, ct).WithCancellation(ct).ConfigureAwait(false))
    await ProcessAsync(order, ct);
```

**`[EnumeratorCancellation]` is not optional** and is the detail most people miss. Because the method is an iterator, the token passed as an argument is captured *when the enumerable is created*, not when it is enumerated. The attribute tells the compiler to substitute the token supplied via `WithCancellation` at enumeration time, combining it with the argument. Without it, `WithCancellation` silently does nothing — a bug that only manifests as a shutdown that hangs.

Other things worth knowing:

- **`ConfigureAwait(false)`** on an `IAsyncEnumerable` uses a different extension (`ConfigureAwait` on the enumerable, from `System.Threading.Tasks.Extensions` / built-in on .NET Core 3.0+), and it applies to the `MoveNextAsync` and `DisposeAsync` awaits.
- **`IAsyncDisposable`** matters here: `await foreach` calls `DisposeAsync` on the enumerator, which is how the connection above gets closed if the consumer breaks early. Breaking out of an `await foreach` is safe and correct.
- **`System.Linq.AsyncEnumerable`** ships in the BCL as of .NET 10 (previously the `System.Linq.Async` package), giving you `Where`, `Select`, `Take`, and friends over async streams.
- **Where to use it:** paging through a large result set, streaming from a message consumer, reading a file line by line, an HTTP response body streamed as NDJSON. **Where not to:** when the entire result is small and already materialized — a `Task<List<T>>` is simpler and faster.
- **ASP.NET Core can return `IAsyncEnumerable<T>` from an action** and will stream the JSON array rather than buffering it, which is a genuine memory win for large responses.

---

## Concept 59 — `System.IO.Pipelines`

`Pipelines` solves a problem `Stream` does not: **who owns the buffer, and what happens when the reader is slower than the writer.**

The classic `Stream` parsing loop has four hard problems the developer must solve: allocating and sizing a buffer, handling messages that span buffer boundaries, avoiding copies, and applying backpressure. `PipeReader`/`PipeWriter` solve all four:

- The pipe owns a pool of buffers; you `GetMemory`/`Advance` to write and `ReadAsync`/`AdvanceTo` to read.
- `AdvanceTo(consumed, examined)` is the key API: it says "I consumed up to here, and I looked at up to there" — so a partial message stays in the buffer and the reader is not woken again until more data arrives.
- Data is exposed as a `ReadOnlySequence<byte>`, which may be multi-segment, so there is **no copy** to make a message contiguous.
- `PauseWriterThreshold`/`ResumeWriterThreshold` provide real backpressure: the writer's `FlushAsync` suspends when the pipe holds too much unread data.

This is the substrate under **Kestrel** and SignalR, and it is why ASP.NET Core's raw throughput is competitive with native servers. You would use it directly for a custom binary protocol, a high-throughput proxy, or a framing layer over TCP. For ordinary application code, you would not — and saying *"I'd reach for `Pipelines` if I were writing a protocol parser, and for a `Stream` otherwise"* is the right calibration.

.NET 11 adds a `ReadOnlySequenceStream` adapter, so a `ReadOnlySequence<byte>` from a pipe can be handed to any `Stream`-taking API without flattening it into a contiguous copy.

---

## Concept 60 — TPL Dataflow

`System.Threading.Tasks.Dataflow` gives you composable blocks with per-block parallelism and bounded capacity: `BufferBlock`, `ActionBlock`, `TransformBlock`, `TransformManyBlock`, `BatchBlock`, `BroadcastBlock`, `JoinBlock`.

```csharp
var download = new TransformBlock<Uri, string>(
    uri => _http.GetStringAsync(uri),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 8, BoundedCapacity = 100 });

var parse = new TransformBlock<string, Document>(Parse,
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = Environment.ProcessorCount });

var store = new ActionBlock<Document>(d => _repo.SaveAsync(d),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 4, BoundedCapacity = 50 });

download.LinkTo(parse, new DataflowLinkOptions { PropagateCompletion = true });
parse.LinkTo(store, new DataflowLinkOptions { PropagateCompletion = true });

foreach (var uri in uris) await download.SendAsync(uri);
download.Complete();
await store.Completion;
```

**Where it still wins over channels:** when the pipeline has several stages with *different* degrees of parallelism, and you want bounded capacity between each stage with automatic completion propagation. Expressing that with channels means writing the plumbing yourself.

**Where channels win:** anything simpler. Dataflow's API surface is large, its error semantics (a faulted block, and how faults propagate through links) take real study, and the library is in maintenance rather than active development. The honest summary: *"Dataflow is the right tool for a multi-stage in-process pipeline with per-stage parallelism, and overkill for anything a channel plus N consumers can express — which is most things."*

---

## Concept 61 — Data parallelism: `Parallel` and PLINQ

For **CPU-bound**, partitionable work:

```csharp
Parallel.For(0, items.Length, i => Process(items[i]));
Parallel.ForEach(items, new ParallelOptions { MaxDegreeOfParallelism = 4 }, Process);
var results = items.AsParallel().WithDegreeOfParallelism(4).Select(Transform).ToArray();
```

The mechanics that determine whether this helps:

- **Partitioning** is the whole game. `Parallel.For` over an array uses range partitioning with dynamic chunk sizing; `Parallel.ForEach` over a non-indexable `IEnumerable` uses chunk partitioning with a growing chunk size, which means a short enumerable may all land on one worker. `Partitioner.Create(..., EnumerablePartitionerOptions.NoBuffering)` disables the buffering when items are expensive and unevenly sized.
- **The per-item body must be substantial** — roughly microseconds, not nanoseconds — or the partitioning and delegate-invocation overhead dominates. Parallelizing a loop that adds two integers makes it slower.
- **Exceptions** are collected into an `AggregateException` after all iterations complete or the loop is stopped. `ParallelLoopState.Break()` (finish everything before this index) and `.Stop()` (stop as soon as possible) are different, and the difference is asked about.
- **Thread-local accumulation** (`Parallel.For`'s `localInit`/`localFinally` overload) is how you avoid the interlocked-increment false-sharing problem in a reduction.
- **PLINQ** adds ordering concerns: it is unordered by default, `AsOrdered()` restores order at a cost, and `ForAll` avoids the merge step when you don't need results back.

**The critical anti-pattern:** `Parallel.ForEach` with an `async` body.

```csharp
Parallel.ForEach(items, async item => await ProcessAsync(item));   // BROKEN
```

The lambda is `async void` from `Parallel`'s perspective — it returns immediately at the first await, `Parallel.ForEach` thinks the iteration is done, the loop completes before any work finishes, and exceptions vanish. This is one of the most common genuine bugs in .NET code, and the fix is Concept 62.

---

## Concept 62 — `Parallel.ForEachAsync`: bounded async fan-out done right

.NET 6 added the primitive that had been missing for a decade:

```csharp
await Parallel.ForEachAsync(
    items,
    new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
    async (item, token) => await ProcessAsync(item, token));
```

Why it is the right tool:

- It takes an **async body** and awaits it properly.
- **`MaxDegreeOfParallelism` is enforced**, so you get bounded concurrency without the semaphore boilerplate.
- It accepts **`IAsyncEnumerable<T>`** as a source, so it works with streaming input — and, critically, it pulls items lazily, so a million-item source does not materialize a million tasks (unlike the `Select` + `WhenAll` + semaphore pattern).
- **Cancellation** is plumbed through the `ParallelOptions` and into the body's token.
- Exceptions are aggregated and cancellation stops further iterations.

**The decision table for async fan-out:**

| Situation | Tool |
|---|---|
| A handful of known, heterogeneous operations | `Task.WhenAll(a, b, c)` |
| A bounded collection, uniform operation, need a limit | **`Parallel.ForEachAsync`** |
| A large or unbounded stream, long-lived consumers | `Channel<T>` + N consumer loops |
| Multi-stage pipeline, different parallelism per stage | TPL Dataflow |
| Need results as they complete rather than all at once | `Task.WhenEach` (.NET 9) |
| CPU-bound partitionable work | `Parallel.For`/PLINQ |

Having that table in your head — and volunteering the *limit* without being asked — is one of the clearest seniority signals in this whole module.

---

## Concept 63 — `WhenAll`, `WhenAny`, and `WhenEach`

```csharp
// All must succeed; awaits until every one completes (or faults)
var results = await Task.WhenAll(tasks);

// First to complete wins
var winner = await Task.WhenAny(tasks);

// Stream completions in order of completion (.NET 9)
await foreach (var completed in Task.WhenEach(tasks))
    Handle(await completed);
```

The semantics that get asked about:

**`Task.WhenAll`:**
- Waits for **all** tasks even if one fails early. It does not short-circuit. If you need short-circuiting, use a linked `CancellationTokenSource` and cancel on first failure.
- The returned task's `Exception` is an `AggregateException` containing *every* failure, but `await` rethrows only the **first**. Losing the other exceptions silently is a common, real bug.
- **It does not limit concurrency.** `Task.WhenAll(tenThousandTasks)` starts all ten thousand. Every one of them was already started when you created it — `WhenAll` only waits.

**`Task.WhenAny`:**
- Returns the first task to reach *any* terminal state, including faulted. "First to complete" is not "first to succeed."
- **The other tasks keep running** and, if they fault, become unobserved exceptions. You must either await them, observe them (`ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing)`), or cancel them.
- The classic timeout idiom `await Task.WhenAny(work, Task.Delay(timeout))` allocates a timer that lives until it fires even when the work wins. `task.WaitAsync(timeout)` (.NET 6) is the better spelling.
- A subtle performance trap: `WhenAny` in a loop to process results as they arrive is **O(n²)** — each call re-registers a continuation on every remaining task.

**`Task.WhenEach` (.NET 9)** exists precisely to fix that last one. It returns an `IAsyncEnumerable<Task>` that yields each task as it completes, in completion order, in O(n) total. It is the right answer to "process results as they arrive" and a good current-platform detail to volunteer:

```csharp
await foreach (var t in Task.WhenEach(tasks).WithCancellation(ct))
{
    try { Render(await t); }                         // await the already-completed task to get its result
    catch (Exception ex) { _logger.LogError(ex, "One branch failed"); }
}
```

---

# Part G — Production and architecture

## Concept 64 — Bounded concurrency as the default posture

The single most valuable habit this module can give you: **every fan-out carries a number, and you can defend where the number came from.**

Unbounded async fan-out is a self-inflicted denial of service. `orders.Select(o => _api.GetAsync(o.Id))` over 5,000 orders opens 5,000 concurrent HTTP requests, exhausts the connection pool, times out, retries (Module 13), and takes down a dependency that was operating perfectly. The synchronous version of this code was *implicitly* bounded by the thread pool; async removed that accidental safety net without replacing it with a deliberate one.

**Where the number comes from** — this is the part that separates an engineer from an architect:

| Constraint | How to derive the limit |
|---|---|
| Database connection pool | `Max Pool Size` (default 100 for SQL Server). Your limit must leave room for everything else using the same pool |
| Downstream service capacity | Their published rate limit, or measured saturation point; divide by your instance count |
| `HttpClient` connections | `SocketsHttpHandler.MaxConnectionsPerServer` (default: unlimited on .NET Core — a genuine footgun) |
| CPU-bound stages | `Environment.ProcessorCount`, adjusted for container CPU quota |
| Memory | concurrency × per-operation working set must fit the container limit (Module 14, Concept 63) |
| Provider quota | Cosmos DB RU/s, Service Bus concurrent-call settings, Event Hubs partitions |

**The instance-count trap** is worth stating explicitly: a limit of 20 per instance across 50 pods is a limit of 1,000 against the dependency. The number you defend must be the *aggregate*, which is why per-instance limits, replica counts, and downstream capacity are one conversation, not three.

The mechanisms, in preference order: `Parallel.ForEachAsync` with `MaxDegreeOfParallelism` → a `Channel<T>` with N consumers → `SemaphoreSlim` → `System.Threading.RateLimiting` when the constraint is a rate rather than a concurrency count.

---

## Concept 65 — Backpressure, end to end

Backpressure is the mechanism by which a slow consumer slows a fast producer. Without it, the buffer between them grows until something dies.

There are only four honest responses to "the queue is full," and a design must pick one **per stage**:

1. **Block/suspend the producer** (`BoundedChannelFullMode.Wait`, Pipelines' pause threshold). Correct when the producer *can* be slowed — reading from a durable queue, a socket with TCP flow control, a paged database read.
2. **Reject** the incoming item with an error (HTTP 429/503, `RateLimiter` rejection). Correct at a system boundary where the caller can retry with backoff.
3. **Drop** — oldest, newest, or by sampling. Correct for telemetry, metrics, and live data where staleness is worse than loss.
4. **Spill** to durable storage. Correct when loss is unacceptable and the burst is bounded.

What is *not* an option is "grow the buffer," which is the default when you use an unbounded queue and is simply deferring the decision to the OOM killer.

**Where this shows up in a .NET stack:**

| Layer | Backpressure mechanism |
|---|---|
| TCP | Receive window; a slow reader slows the sender automatically |
| Kestrel | `MaxConcurrentConnections`, `MaxRequestBodySize`, request-body rate limits |
| ASP.NET Core | Rate-limiting middleware (Concept 70), queue limits |
| `Channel<T>` | Bounded capacity + `FullMode` |
| `IAsyncEnumerable<T>` | Inherent — the producer waits for `MoveNextAsync` |
| Pipelines | `PauseWriterThreshold` |
| Service Bus / Event Hubs | Prefetch count, `MaxConcurrentCalls` (Module 11) |
| Your own fan-out | The concurrency limit from Concept 64 |

**The architect's version of this answer** connects it to Module 6 and Module 13: backpressure *is* overload protection applied internally. A system without it converts a transient slowdown into unbounded memory growth, and unbounded memory growth into a restart, and a restart into a thundering herd. Being able to trace that chain out loud is the answer to "what happens when your downstream gets slow?"

---

## Concept 66 — Fire-and-forget is a bug

```csharp
_ = SendEmailAsync(order);        // looks harmless; is not
```

Four separate problems:

1. **Exceptions are lost.** They land in a task nobody observes (Concept 18).
2. **Shutdown does not wait for it.** The process can exit mid-operation. In Kubernetes, SIGTERM plus a 30-second grace period means your "background" email is killed with no record.
3. **It is unbounded.** N requests means N concurrent background operations, with no limit and no queue.
4. **It captures the request scope.** In ASP.NET Core, work started in a request and outliving it holds a `DbContext`, an `HttpContext`, and everything reachable from them — a use-after-dispose bug and a retention leak in one.

**The replacements, in order of preference:**

| Situation | Answer |
|---|---|
| Work that must happen | A durable queue (Module 11). The request writes a message; a consumer does the work. Survives restarts |
| In-process deferral, loss acceptable on restart | A bounded `Channel<T>` written by the request, drained by a `BackgroundService` |
| Genuinely optional, best-effort | An explicit helper that logs failures and is bounded |

```csharp
// If you truly must, make it explicit and safe
public static void FireAndForget(this Task task, ILogger logger, string operation)
{
    _ = task.ContinueWith(
        t => logger.LogError(t.Exception, "Background operation {Operation} failed", operation),
        CancellationToken.None,
        TaskContinuationOptions.OnlyOnFaulted | TaskContinuationOptions.ExecuteSynchronously,
        TaskScheduler.Default);
}
```

The named method is the point: `_ = SomethingAsync()` is invisible in review; `SomethingAsync().FireAndForget(logger, "send-email")` is a decision someone made and can be searched for.

---

## Concept 67 — Background work in ASP.NET Core

```csharp
public sealed class OrderProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopes;
    private readonly ChannelReader<OrderId> _reader;
    private readonly ILogger<OrderProcessor> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var id in _reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                // A SCOPE PER UNIT OF WORK — not one for the service's lifetime
                await using var scope = _scopes.CreateAsyncScope();
                var handler = scope.ServiceProvider.GetRequiredService<IOrderHandler>();
                await handler.HandleAsync(id, stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;                                     // shutdown, not an error
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed processing {OrderId}", id);
                // swallow deliberately: one bad item must not kill the service
            }
        }
    }
}
```

The five things that go wrong, all of which are worth naming:

1. **Injecting a scoped service into a singleton `BackgroundService`.** `BackgroundService` is a singleton; a `DbContext` injected into its constructor lives for the process lifetime, is used concurrently, and accumulates tracked entities forever. This is the **captive dependency** problem (Module 14's leak taxonomy; Module 18 in depth). Always `IServiceScopeFactory` + a scope per unit of work.
2. **Blocking in `ExecuteAsync` before the first `await`.** The host awaits `StartAsync`, which calls `ExecuteAsync`; anything synchronous before the first suspension delays application startup. Start with `await Task.Yield()` if you have synchronous setup.
3. **Not honouring `stoppingToken`.** The host waits `HostOptions.ShutdownTimeout` (default 30 seconds) and then abandons you.
4. **Letting an exception escape.** Since .NET 6, `BackgroundServiceExceptionBehavior.StopHost` is the default — an unhandled exception in `ExecuteAsync` **stops the entire host**. That is usually correct (fail fast, let the orchestrator restart), but it must be a deliberate choice, and per-item errors must be caught inside the loop.
5. **Multiple instances.** Scale out to 3 pods and you have 3 copies of the background service. If the work must happen once, you need a leader election or a distributed lock (Module 9) — or the work belongs on a queue with competing consumers, where duplication is the design rather than a bug.

---

## Concept 68 — Graceful shutdown

The full sequence in a .NET generic host:

1. SIGTERM (or Ctrl-C) arrives; the orchestrator starts its grace-period timer.
2. `IHostApplicationLifetime.ApplicationStopping` fires; the `stoppingToken` given to every `BackgroundService` is cancelled.
3. The server stops accepting **new** connections/requests but lets in-flight ones complete.
4. `StopAsync` is called on hosted services **in reverse registration order**, with a deadline of `HostOptions.ShutdownTimeout` (default 30 s).
5. `ApplicationStopped` fires; the DI container is disposed; the process exits.

The design rules:

- **Register readiness/liveness separately** so the load balancer removes you *before* shutdown begins. Without a pre-stop delay, requests are still being routed to you when you stop accepting them — so add a `preStop` hook or a shutdown delay of a few seconds (longer than your LB's health-check interval).
- **Set `ShutdownTimeout` to less than the orchestrator's grace period** (Kubernetes `terminationGracePeriodSeconds`, default 30 s). If yours is longer, you get SIGKILL mid-flush.
- **Distinguish "drain" from "cancel."** In-flight requests should be allowed to finish; queued-but-not-started work should be abandoned and left for the next consumer. Passing `stoppingToken` to everything cancels both — which is right for a queue consumer (the message is redelivered) and wrong for a half-written file.
- **Idempotency is the safety net.** Shutdown mid-operation means the operation may be retried. Module 11's delivery-semantics discussion and Module 12's idempotency keys are what make abrupt termination survivable, and saying so connects this module back to the design phases.

---

## Concept 69 — Time, timers, and testability

| Primitive | Behaviour | Use for |
|---|---|---|
| `System.Threading.Timer` | Callback on a pool thread; **ticks can overlap** if the callback is slow | Legacy; avoid in new code |
| `PeriodicTimer` (.NET 6) | `await WaitForNextTickAsync(ct)`; **no overlap** by construction; no allocation per tick | The default for periodic async work |
| `Task.Delay` | One-shot; allocates a timer | Backoff, simple waits |
| `TimeProvider` (.NET 8) | Abstraction over time: `GetUtcNow`, `CreateTimer`, `Delay`, `GetTimestamp` | **Everything**, so it can be faked |

```csharp
// The modern periodic loop
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
while (await timer.WaitForNextTickAsync(stoppingToken))
    await DoWorkAsync(stoppingToken);
```

`PeriodicTimer`'s no-overlap guarantee is the reason to prefer it: a `System.Threading.Timer` with a 30-second period and a 45-second callback produces overlapping executions, which is a race condition nobody designed and everybody is surprised by.

**`TimeProvider` is the testability story**, and it is a strong signal to bring up unprompted. Inject `TimeProvider` rather than calling `DateTime.UtcNow` or `Task.Delay` directly; in tests, use `FakeTimeProvider` (from `Microsoft.Extensions.TimeProvider.Testing`) and *advance time manually*. A retry-with-backoff test that currently takes 30 seconds of real `Task.Delay` becomes instant and deterministic. This is the single highest-leverage change available to most .NET test suites, because it converts the slowest and flakiest tests into fast reliable ones.

---

## Concept 70 — Rate limiting

`System.Threading.RateLimiting` (.NET 7) provides the four classic algorithms as first-class types:

| Limiter | Semantics | Use for |
|---|---|---|
| `ConcurrencyLimiter` | N in flight at once | Bulkheads; protecting a dependency with a connection limit |
| `TokenBucketRateLimiter` | Tokens refill at a rate; bursts allowed up to bucket size | APIs where bursts are acceptable |
| `FixedWindowRateLimiter` | N per fixed window | Simple quotas; suffers the boundary-burst problem (2N across a boundary) |
| `SlidingWindowRateLimiter` | N per window, segmented | Smoother than fixed window, more state |

`PartitionedRateLimiter<T>` applies a limiter per key — per tenant, per API key, per IP — which is the shape almost every real requirement takes.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetTokenBucketLimiter(
            ctx.User.FindFirst("tenant")?.Value ?? "anonymous",
            _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100,
                TokensPerPeriod = 100,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1),
                QueueLimit = 0                       // reject rather than queue
            }));
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});
```

The design points worth making: **`QueueLimit = 0` (reject immediately) is usually better than queueing**, because a queued request is a request that will likely time out anyway while holding resources — Module 13's "fail fast" applied locally. And the limiter belongs on **both** sides: server-side to protect yourself from clients, client-side to protect your dependencies from you.

---

## Concept 71 — Concurrency at the data layer

In-process synchronization stops at the process boundary. The moment you have two instances, a `lock` protects nothing. The correct tools move down a layer:

| Problem | In-process answer | Correct distributed answer |
|---|---|---|
| Lost update | `lock` | Optimistic concurrency: a rowversion/ETag, checked on write (Module 12) |
| Duplicate processing | A `HashSet` of seen IDs | An idempotency key with a unique constraint (Module 11) |
| Exactly-once side effect | A flag | Transactional outbox (Module 11, Module 12) |
| Mutual exclusion | `Mutex` | A lease with a TTL and a fencing token (Module 9) |
| Counter | `Interlocked` | An atomic database increment, or a CRDT (Module 9) |
| Read-modify-write | `lock` | `UPDATE ... WHERE version = @expected` and check rows-affected |

The EF Core expression of optimistic concurrency, since it comes up constantly:

```csharp
modelBuilder.Entity<Order>().Property(o => o.RowVersion).IsRowVersion();

try { await db.SaveChangesAsync(ct); }
catch (DbUpdateConcurrencyException ex)
{
    // Someone else changed it. Reload, re-apply, retry — or surface a conflict to the user.
}
```

**The sentence that lands:** *"An in-process lock is an optimization over a database-level guarantee, never a replacement for one. If correctness depends on the lock, the design breaks the first time we scale to two instances — and autoscaling means that happens without anyone deciding."*

---

## Concept 72 — Observability for concurrency

The metrics that give you early warning, and what each one means:

| Metric | Source | Meaning |
|---|---|---|
| `dotnet.thread_pool.thread.count` | `System.Runtime` meter (.NET 9+) / `threadpool-thread-count` EventCounter | Climbing above core count ⇒ blocking |
| `dotnet.thread_pool.queue.length` | Meter / `threadpool-queue-length` | Sustained > 0 ⇒ the pool cannot keep up |
| `dotnet.thread_pool.work_item.count` | Meter / `threadpool-completed-items-count` | Throughput; a drop with a rising queue is starvation |
| `lock-contention-count` | EventCounter / `dotnet.monitor.lock_contentions` | High ⇒ a hot lock; find it before it convoys |
| `active-timer-count` | EventCounter | Growing without bound ⇒ leaked timers/registrations |
| Your own in-flight gauge | `SemaphoreSlim.CurrentCount`, channel count | Where your concurrency limit actually sits |
| `Activity` duration + status | OpenTelemetry (Module 28) | Which stage is slow |

**The alert that matters most:** ThreadPool queue length sustained above zero for more than a few seconds. It fires before latency does, it has essentially no false positives in a healthy service, and it points directly at the cause.

Two more practices worth stating:

- **Emit your own concurrency numbers as metrics.** The semaphore's available count, the channel's current depth, the number of in-flight downstream calls. These are the numbers you will want during an incident and cannot reconstruct afterwards.
- **`Activity`/`ActivitySource` flows via `AsyncLocal`**, so distributed traces survive `await` automatically (Concept 23). That is the concrete payoff for understanding `ExecutionContext`: it's why your trace context doesn't fall off at an async boundary, and why it *does* fall off if you cross a `Task.Run` with `ExecutionContext.SuppressFlow` or hop through a raw thread.

---

## Concept 73 — The diagnostic workflow

**Step 1 — Classify from counters.** `dotnet-counters monitor -p <pid> --counters System.Runtime`

| Signature | Diagnosis |
|---|---|
| Low CPU, high latency, thread count climbing, queue growing | **ThreadPool starvation** → step 2 |
| High CPU, high latency, flat thread count | CPU-bound work or a hot loop → profile with `dotnet-trace` |
| High `% Time in GC`, sawtooth memory | A Module 14 problem, not this one |
| Everything flat, latency high | A downstream dependency → check your client-side histograms |
| High `lock-contention-count` | Lock contention → step 3 |

**Step 2 — See what the threads are doing.** `dotnet-stack report -p <pid>` prints every managed stack immediately, with no dump required. Look for the repeated frame: if forty threads are sitting in `Task.Result`, `Monitor.Enter`, `SemaphoreSlim.Wait`, or a synchronous driver call, you have your answer and the stack tells you the exact line.

**Step 3 — Take a dump when you need object state or the async graph.**

```
dotnet-dump collect -p <pid>
dotnet-dump analyze core_...
> clrthreads              # all threads, their state, their locks
> syncblk                 # which monitors are held, by which thread
> clrstack -a             # managed stack with arguments/locals for the current thread
> dumpasync               # PENDING ASYNC STATE MACHINES — the async-aware view
> threadpool              # queue lengths and thread counts
```

**`!dumpasync` is the command that separates people who have actually debugged async production issues from people who have read about it.** It walks the heap for async state-machine boxes and reconstructs the logical async "stacks" — showing you a chain of awaiting methods even though no OS thread is on any of them. For an async hang, where `clrstack` shows nothing because no thread is doing anything, it is the only tool that answers the question.

**Step 4 — For intermittent problems, trace.** `dotnet-trace collect -p <pid> --providers Microsoft-Windows-DotNETRuntime:0x10000:5` captures ThreadPool events (work item enqueue/dequeue, thread injection) so you can see the injection ramp and queueing in time order. PerfView's "Thread Time" view reconstructs blocked-time attribution, which answers "what was this thread waiting on" across the whole trace.

Being able to narrate this four-step workflow — counters to classify, stacks to locate, dump plus `dumpasync` to confirm, trace for intermittents — is a complete answer to "how would you debug a production hang," and it is a question you should expect.

---

## Concept 74 — Testing concurrent code

Concurrency bugs are probabilistic, so ordinary unit tests are nearly useless at finding them and quite good at hiding them. The strategies that actually work:

1. **Design for determinism.** Inject `TimeProvider` and use `FakeTimeProvider` (Concept 69). Inject a `TaskScheduler` or an abstraction over "run this work" so tests can run it inline. Most concurrency tests that are flaky are flaky because they use real time.
2. **Test the *logic* separately from the *concurrency*.** Extract the state transition into a pure function, test it exhaustively, and then test only that the synchronization wraps it correctly.
3. **Stress tests, run many times, with the right assertions.** Launch N tasks against the type, `Task.WhenAll`, and assert an invariant (final count, no duplicates, no exceptions). Run it 1,000 times in CI. This finds real races — not reliably, but far better than nothing.
4. **Use `Interlocked` counters in the test, not shared lists**, or your test harness has the race.
5. **Run tests on more cores and on Arm64.** A race that never fires on a 2-core CI agent fires on a 32-core one; a memory-model bug that never fires on x64 fires on Graviton. Putting an Arm64 leg in CI is a cheap, high-value change and a good thing to propose in an interview.
6. **Use the analyzers as tests**: `CA1849` (call async methods in async contexts), `CA2007`, `CA2012`, `VSTHRD100/103/200`. They catch statically what tests catch probabilistically.
7. **Know the limits.** There is no `Thread.Sleep` value that makes a concurrency test correct. If a test needs a sleep to pass, it is asserting on timing, and it will be flaky forever. Use a `TaskCompletionSource` as a synchronization point instead.

---

## Concept 75 — The async boundary: library and API design

Rules for any code that others consume, each with a reason:

1. **Never expose a synchronous wrapper over an async method** (`GetAsync().Result`). Sync-over-async in a library imposes the cost on every caller and hides it. If callers need sync, give them a genuinely synchronous implementation or nothing.
2. **Never expose an async wrapper over a synchronous method** (`Task.Run(() => GetSync())`). Fake async is worse than honest sync: it lies about scalability and adds a thread hop. Expose the sync method and let the caller decide whether to offload.
3. **Take a `CancellationToken` on every async public method**, defaulted to `default`, as the last parameter. Not taking one is a permanent API break to fix.
4. **`ConfigureAwait(false)` on every `await`** in library code (Concept 24), enforced with `CA2007`.
5. **Suffix async methods with `Async`.** It is a convention the whole ecosystem relies on, including analyzers.
6. **Return `Task`, not `ValueTask`, unless you have measured.** (Concept 14.)
7. **Validate arguments synchronously** using the wrapper pattern from Concept 17.
8. **Document the thread-safety contract** explicitly (Concept 52), and register the type in DI with a lifetime that matches it.
9. **Don't expose both `Foo()` and `FooAsync()` doing the same thing** unless both are genuinely implemented. The BCL's own history here (`Stream.Read` and `Stream.ReadAsync` both being real) is the standard to meet.
10. **Don't let a `SynchronizationContext` assumption leak in.** A library that only works when resumed on a particular thread is broken for most consumers.

**The architectural version of this concept** is deciding *where* the async boundary sits in a system: async from the HTTP/queue entry point all the way to the driver, with no sync islands; CPU-bound stages explicitly offloaded; and any genuinely blocking dependency isolated behind a bulkhead with its own threads, so its blocking cannot starve the shared pool. That last clause is the design that survives a legacy synchronous SDK you cannot replace, and it is a very good thing to have an opinion about.

---

## Concept 76 — When *not* to be async

The competence to say "no" here is a seniority marker, because the industry default has overcorrected.

**Don't make it async when:**

- **Nothing blocks.** In-memory computation, a dictionary lookup, a pure transformation. An `async` method with no `await` is a compiler warning (CS1998) and a wasted state machine.
- **It's a console tool or a build script.** One operation at a time, no pool to starve. `async Main` costs nothing, so use it if it's convenient, but there is nothing to optimize.
- **It's a CPU-bound pipeline.** Use `Parallel`/PLINQ. Async adds overhead and saves no threads.
- **The operation is always sub-millisecond.** The state machine and scheduling cost can exceed the operation.
- **You'd be faking it.** `Task.Run` around blocking code (Concept 32).
- **The retrofit cost exceeds the benefit.** A low-traffic internal admin service with 5 concurrent users does not have a thread-scarcity problem. Rewriting it async is real risk for no measurable gain. Saying this in an interview — *"I'd want to see the thread count and the queue length before proposing that migration"* — is a better answer than enthusiasm.

**Do make it async when:** the operation waits on something external, and concurrency is high enough that threads are a scarce resource. That is the whole test, and it returns to the opening sentence: async is about not paying for a thread while nothing is happening.

---

# Putting it together

## Worked example 1 — "Our API's p99 went from 80 ms to 12 seconds, but CPU is at 15%. Diagnose it."

**The narration that scores:**

"Low CPU with high latency rules out a capacity or algorithmic problem immediately — if we were doing too much work, CPU would be high. So the threads are waiting, not working. Two candidates: a downstream dependency got slow, or we're starving the ThreadPool.

I'd separate those with `dotnet-counters`. If `threadpool-thread-count` is climbing steadily above core count and `threadpool-queue-length` is sustained above zero, it's starvation — our own threads are blocked. If thread count is flat and the queue is empty, it's downstream, and I'd go look at the client-side latency histogram for each dependency.

Assume starvation. `dotnet-stack report` gives me every managed stack in one shot. I'm looking for the repeated frame — forty threads all in `Task.Result`, `SemaphoreSlim.Wait`, `Monitor.Enter`, or a synchronous driver call. That frame names the line.

The mechanism, to be explicit about why a small mistake produced a twelve-second p99: a blocking call holds a pool thread for the full duration of the operation. Beyond `MinThreads`, which is the core count, the pool injects roughly one thread every 500 ms. If the load needs 100 threads and we start with 8, that's about 46 seconds of ramp. During the ramp the queue grows, and queued time is added to every request's latency — so the p99 isn't the operation's duration, it's the operation's duration plus queue wait.

The immediate mitigation is raising `ThreadPoolMinThreads` — it stops the bleeding within a deploy. The actual fix is removing the block. And if the blocking dependency genuinely has no async API, I'd bulkhead it: a dedicated thread pool or a `Channel<T>` with a fixed number of dedicated-thread consumers, so its blocking can't consume the shared pool that everything else depends on.

The thing I'd want to add afterwards is an alert on ThreadPool queue length, because it fires before latency does and it has almost no false positives."

---

## Worked example 2 — "Design the concurrency model for an ingestion service: 50k events/sec, enrich each from a cache, batch-write to storage."

**Structure the answer around the pipeline stages and give each one a number.**

**Stage 1 — Receive.** Event Hubs or Kafka consumer, N partitions. Concurrency is bounded by partition count; prefetch bounds how much is buffered. The client library's `MaxConcurrentCalls`/prefetch is the first limit, and it's set so that in-flight events × per-event memory fits the container budget (Module 14, Concept 63).

**Stage 2 — Buffer.** A `Channel<T>`, **bounded**. Capacity chosen so a brief downstream stall doesn't drop data but a sustained one doesn't OOM us: at 50k/s and a 200-byte event, a 100k-item buffer is 20 MB and about 2 seconds of runway. `FullMode = Wait`, so a full channel slows the consumer, which stops us acknowledging, which lets the broker apply real backpressure to the source. That's backpressure propagating all the way out, which is the property I actually want.

**Stage 3 — Enrich.** N consumer loops reading the channel. Cache hits are the common case, so the lookup returns `ValueTask<T>` and completes synchronously — zero allocations on the hot path. Misses go to a database with a `SemaphoreSlim` limiting concurrent lookups to something under the connection pool size, and a `ConcurrentDictionary<K, Lazy<Task<V>>>` so a thousand simultaneous misses for the same key produce one query, not a thousand.

**Stage 4 — Batch.** Accumulate into batches of 500 or 100 ms, whichever comes first — that's a `PeriodicTimer` plus a count check, or a Dataflow `BatchBlock` with a timer. Batching is what makes 50k writes/sec affordable.

**Stage 5 — Write.** Bounded concurrency on the storage client, sized from its documented throughput. Failures retry with jittered backoff and a circuit breaker (Module 13); a permanently failing batch goes to a dead-letter path rather than blocking the pipeline.

**Cross-cutting:** every stage takes the `stoppingToken`; shutdown drains the channel with a deadline; the whole thing is idempotent at the storage layer via an event ID, because at-least-once delivery plus shutdown-mid-batch means duplicates will happen (Module 11).

**Numbers I'd want before committing:** per-event CPU cost (decides consumer count), cache hit rate (decides database concurrency), storage write latency and throughput (decides batch size and write concurrency), and per-event memory (decides channel capacity).

---

## Worked example 3 — "Review this code."

```csharp
public class OrderService
{
    private static readonly HttpClient _http = new();
    private readonly List<Order> _cache = new();
    private readonly object _lock = new();

    public async void ProcessOrders(List<int> ids)
    {
        Parallel.ForEach(ids, async id =>
        {
            var order = await GetOrderAsync(id);
            lock (_lock) { _cache.Add(order); }
        });
    }

    public Order GetOrder(int id) => GetOrderAsync(id).Result;

    private async Task<Order> GetOrderAsync(int id)
    {
        var json = await _http.GetStringAsync($"/orders/{id}");
        return JsonSerializer.Deserialize<Order>(json)!;
    }
}
```

**Eight defects, in severity order:**

1. **`Parallel.ForEach` with an `async` lambda** (Concept 61). The lambda is `async void` to `Parallel`; the loop returns immediately, before a single order has been fetched, and every exception is lost. The method "succeeds" having done nothing. Fix: `await Parallel.ForEachAsync(ids, options, async (id, ct) => ...)` with a `MaxDegreeOfParallelism`.
2. **`async void` on `ProcessOrders`** (Concept 18). Unawaitable, and any exception crashes the process. Fix: `async Task`.
3. **`.Result` in `GetOrder`** (Concept 30). Burns a pool thread, wraps exceptions in `AggregateException`, and deadlocks under a single-threaded context. Fix: delete it; make callers async.
4. **Unbounded fan-out** (Concept 64). 10,000 IDs means 10,000 concurrent HTTP requests against one host. Fix: the `MaxDegreeOfParallelism` from (1), plus `MaxConnectionsPerServer` on the handler.
5. **`static HttpClient` constructed directly.** Shared is right, but a raw static `HttpClient` never refreshes DNS. Fix: `IHttpClientFactory`, or a `SocketsHttpHandler` with `PooledConnectionLifetime` set.
6. **`List<T>` guarded by a lock that doesn't cover reads.** Writes are guarded; any reader of `_cache` elsewhere is unguarded and can observe a torn resize. Also the "cache" is unbounded and never evicts — a leak (Module 14, Concept 57). Fix: `ConcurrentDictionary` or a bounded `IMemoryCache`/`HybridCache` (Module 10).
7. **No `CancellationToken` anywhere** (Concept 19). Nothing can be cancelled, so shutdown hangs for the full timeout.
8. **No error handling for a partial failure.** One bad ID takes out the batch with nothing recorded about which one.

**What good looks like:**

```csharp
public async Task ProcessOrdersAsync(IReadOnlyList<int> ids, CancellationToken ct)
{
    await Parallel.ForEachAsync(
        ids,
        new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
        async (id, token) =>
        {
            try
            {
                var order = await GetOrderAsync(id, token).ConfigureAwait(false);
                _cache.Set(id, order, _cacheOptions);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Failed to load order {OrderId}", id);
            }
        }).ConfigureAwait(false);
}
```

Saying "eight things, and here's the order I'd fix them in" is a far better answer than listing them at random — it shows you can triage, which is the actual job.

---

## Worked example 4 — "This counter is wrong under load. Why?"

```csharp
private int _processed;
private bool _running = true;

public void Worker()
{
    while (_running)
    {
        DoWork();
        _processed++;
    }
}

public int Processed => _processed;
public void Stop() => _running = false;
```

**Three distinct bugs, each from a different concept, and the ability to separate them is the point:**

1. **`_processed++` is not atomic** (Concept 38). It is read, add, write. Two threads can read the same value and both write the same result, losing an increment. Fix: `Interlocked.Increment(ref _processed)`.
2. **`Processed` may read a stale value** (Concept 37). Without a volatile read there is no acquire barrier, so the reader may observe a value cached in a register or reordered. Fix: `Volatile.Read(ref _processed)` — or, if the counter is `long`, `Interlocked.Read` on 32-bit.
3. **`_running` may never be observed as false** (Concept 34). The JIT may hoist the field read out of the loop, producing an infinite loop that ignores `Stop()`. Fix: `volatile bool`, or — better — a `CancellationToken`, which gets all of this right and composes with everything else.

**The version to write:**

```csharp
private long _processed;

public void Worker(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        DoWork();
        Interlocked.Increment(ref _processed);
    }
}

public long Processed => Interlocked.Read(ref _processed);
```

**The bonus observation that lands well:** "If this is a hot per-core counter and I have one of these per worker in an array, I'd also check for false sharing (Concept 40) — eight `long` counters in one array share a cache line and will scale badly. Either pad them or accumulate per-thread and sum on read."

---

## Common questions and what a strong answer contains

**"What does `async`/`await` actually do?"** The compiler rewrites the method into a state machine; `await` checks `IsCompleted`, and if not complete, registers a continuation and *returns* — freeing the thread. The continuation resumes `MoveNext` later, possibly on a different thread. Do not say "it runs on a background thread" (Concepts 1, 10).

**"Does `async` create threads?"** No. It *avoids* occupying one during a wait. `Task.Run` uses a pool thread; `await` on I/O uses none (Concepts 4, 32).

**"Why is my async code slower?"** Because for a single operation it is — you pay a state machine, possibly an allocation, and a scheduling hop. Async trades per-operation latency for throughput and density. If nothing was going to block, you paid for nothing (Concepts 12, 76).

**"What is `ConfigureAwait(false)` and do I need it?"** It opts out of resuming on the captured `SynchronizationContext`/`TaskScheduler`. Required in libraries; unnecessary in ASP.NET Core app code because there is no context; wrong by default in UI code. It does not change threads, suppress exceptions, or affect `AsyncLocal` (Concept 24).

**"Why does `.Result` deadlock?"** Only where a single-threaded context exists: the caller blocks the one thread the continuation needs to resume on. ASP.NET Core has no context, so it doesn't deadlock — but it still burns a pool thread, which is the real problem (Concepts 30, 31).

**"What is ThreadPool starvation and how would you diagnose it?"** Blocked pool threads plus injection at ~one thread per 500 ms. The signature is low CPU, high latency, climbing thread count, growing queue. Diagnose with counters then `dotnet-stack`; fix by removing the block, mitigate with `MinThreads` (Concepts 27, 28, 73).

**"`Task` vs `ValueTask`?"** `ValueTask` avoids the allocation when the operation usually completes synchronously, at the cost of strict consumption rules: await once, never twice, never concurrently, never `.Result` before completion. Default to `Task` (Concept 14).

**"What's new in .NET async recently?"** Runtime async in .NET 11 — the runtime owns suspension, so live stack traces are the real call chain, allocations drop, and tiered compilation applies to async versions. Preview via `<Features>runtime-async=on</Features>`; the BCL already ships compiled with it. Also `System.Threading.Lock` (.NET 9), `Task.WhenEach` (.NET 9), `ConfigureAwaitOptions` (.NET 8) (Concepts 16, 25, 43, 63).

**"Why didn't .NET do green threads like Java's virtual threads?"** It ran the experiment and rejected it: native interop can't be parked, movable stacks conflict with exact stack scanning and `ref` safety, the ecosystem would need blocking twins of every async API, and the measured win wasn't there. Java needed them because its ecosystem was blocking-by-default; .NET had already paid the async migration (Concept 7).

**"Why can't you `await` inside a `lock`?"** `Monitor` is thread-affine and the continuation may resume on another thread, so `Monitor.Exit` would run on a non-owner. Use `SemaphoreSlim` — and note it isn't reentrant, unlike `Monitor` (Concepts 44, 45).

**"What does `volatile` do?"** Acquire on read, release on write; stops the JIT caching the field in a register. It does **not** make operations atomic (`volatile x++` still races) and does not give sequential consistency. Can't be applied to `long`/`double` (Concept 37).

**"Why might a concurrency bug appear on Arm but not x64?"** x64 is Total Store Order — only store-then-load reordering is allowed — so a lot of under-synchronized code accidentally works. Arm64 is weakly ordered and will reorder. Missing barriers surface on Graviton/Ampere/Apple Silicon (Concept 35).

**"How do you limit concurrency?"** `Parallel.ForEachAsync` with `MaxDegreeOfParallelism`, a `Channel<T>` with N consumers, or `SemaphoreSlim`. Then the real question: where does the number come from? Connection-pool size, downstream capacity, CPU count, memory budget — and multiplied by instance count for the aggregate (Concepts 62, 64).

**"`ConcurrentDictionary.GetOrAdd` — any problems?"** The factory can run multiple times for the same key and runs outside the lock. Only one result is stored. Wrap in `Lazy<T>` with `ExecutionAndPublication` when the factory is expensive or has side effects (Concept 53).

**"How do you do fire-and-forget?"** You don't. It loses exceptions, ignores shutdown, is unbounded, and captures the request scope. Use a durable queue if it must happen, a bounded channel plus a `BackgroundService` if in-process is acceptable (Concept 66).

**"Cancellation — how does it work?"** Cooperative. `CancellationTokenSource` cancels, `CancellationToken` observes, and it only works if you pass it everywhere. `Thread.Abort` is gone because injecting an exception at an arbitrary point leaves invariants broken. Dispose registrations and linked sources or they leak (Concepts 19, 20).

**"How would you debug an async hang with no CPU usage?"** Dump plus `!dumpasync`, because no OS thread is executing the hung work — the state machines are on the heap. `clrthreads`/`syncblk` for monitor deadlocks, `dotnet-stack` first because it needs no dump (Concept 73).

**"When would you use `Channel<T>` over a `ConcurrentQueue`?"** When a consumer needs to *wait* for items without blocking a thread, and when you want bounded capacity with a defined full-mode policy. `ConcurrentQueue` has no async API and no backpressure (Concepts 54, 57).

**"`Task.WhenAll` — any gotchas?"** It waits for all even after one fails; `await` rethrows only the first exception while the rest sit in the `AggregateException`; and it doesn't limit concurrency — the tasks were already running (Concept 63).

**"Is `async` worth it for a service with 20 users?"** Probably not, and saying so is the right answer. Threads only become scarce under concurrency. Show me the thread count and queue length before we plan the migration (Concept 76).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "`async` makes it run in parallel / faster" | Separates concurrency, parallelism, asynchrony; frames async as thread density |
| "Blocking wastes CPU" | Explains that blocking wastes a *thread*, and that CPU is low during starvation |
| Uses `.Result`/`.Wait()` | Async all the way; knows `GetAwaiter().GetResult()` is the least-bad spelling and why |
| "ASP.NET Core can't deadlock, so blocking is fine" | Knows the deadlock went away and the thread cost didn't |
| Adds `ConfigureAwait(false)` everywhere in an ASP.NET Core app | Knows there's no context to capture; applies it in libraries, enforced by an analyzer |
| Thinks `ConfigureAwait(false)` moves work to a background thread | Knows it removes a resumption constraint and nothing else |
| `Task.Run` in a request handler "to speed it up" | Knows the request is already on a pool thread and this takes a second one |
| `Task.Run(() => SyncMethod())` in a library | Refuses to fake async; exposes the sync method and lets the caller decide |
| `async void` outside an event handler | Knows the exception goes to the process, and uses `async Task` |
| `Parallel.ForEach` with an `async` lambda | Uses `Parallel.ForEachAsync`; can explain why the original silently does nothing |
| `Task.WhenAll` over an unbounded list | Attaches a concurrency limit and can derive the number from a real constraint |
| Doesn't know `MinThreads` exists | Knows the default, the ~500 ms injection rate, and that raising it is a mitigation not a fix |
| Treats starvation and a slow dependency as the same symptom | Separates them with thread count and queue length in thirty seconds |
| `volatile` used as a lightweight lock | Knows it's acquire/release per access, doesn't make `++` atomic, and isn't allowed on `long` |
| Writes double-checked locking by hand | Uses `Lazy<T>` with the right `LazyThreadSafetyMode` |
| Ignores the memory model as academic | Explains x64 TSO vs weak Arm64 and why bugs surfaced when they moved to Graviton |
| `lock (this)` / `lock (typeof(X))` / `lock ("key")` | Private dedicated gate, or `System.Threading.Lock` on .NET 9+ |
| Unaware of `System.Threading.Lock` | Knows it's .NET 9, why `EnterScope()` returns a `ref struct`, and what the analyzer catches |
| Uses `SemaphoreSlim.Wait()` in async code | `WaitAsync(ct)`, with `Release()` in a `finally`, and knows it isn't reentrant |
| `ConcurrentDictionary.GetOrAdd` with an expensive factory | Wraps in `Lazy<T>`; knows the factory runs outside the lock and can run twice |
| Uses `ConcurrentBag` as a work queue | Knows it's thread-affine with stealing, and reaches for `ConcurrentQueue` or `Channel<T>` |
| Unbounded `Channel.CreateUnbounded` by default | Bounded by default; picks a `FullMode` deliberately and explains the drop policy |
| Fire-and-forget with `_ =` | Durable queue or bounded channel; knows about scope capture and shutdown |
| Injects a scoped service into a `BackgroundService` | `IServiceScopeFactory` and a scope per unit of work; names the captive-dependency problem |
| Assumes thread-safety from a type's name | Reads the contract; knows `HttpClient` yes, `DbContext` no, `Random.Shared` yes |
| Uses `System.Threading.Timer` for periodic work | `PeriodicTimer` — no overlapping ticks by construction |
| `DateTime.UtcNow` and `Task.Delay` in tests | `TimeProvider` + `FakeTimeProvider`; deterministic, instant tests |
| Uses an in-process `lock` for cross-instance correctness | Optimistic concurrency, idempotency keys, or a lease with a fencing token |
| Has never opened a dump for an async hang | Narrates counters → `dotnet-stack` → dump + `!dumpasync` |
| Says "we should make everything async" | Asks for thread count and queue length first, and is willing to say "leave it alone" |
| Doesn't know what changed recently | Names runtime async, `Lock`, `WhenEach`, `ConfigureAwaitOptions`, and why green threads were rejected |

---

## Practice exercises

**Exercise 1 — Cause starvation, then watch the pool ramp (45 min).** Build a minimal ASP.NET Core endpoint that calls `Task.Delay(500).GetAwaiter().GetResult()`. Load it with `bombardier`/`k6` at 200 rps. Watch `dotnet-counters monitor --counters System.Runtime` and record: thread count over time, queue length, CPU, and p99. Now set `<ThreadPoolMinThreads>200</ThreadPoolMinThreads>` and repeat. Then fix the code properly and repeat. Three graphs, one page of conclusions. **This is the highest-value exercise in the module** — it converts Concepts 27 and 28 from a story into numbers you measured.

**Exercise 2 — Read the generated state machine.** Put a three-`await` method into [sharplab.io](https://sharplab.io/), switch the output to C# and then to IL, and map every generated member back to Concept 10. Then add a `try/finally` around one await and find where the state machine puts it. Then add a `using` and find the disposal. This is 30 minutes that will make async internals permanent.

**Exercise 3 — Measure the allocation profile.** BenchmarkDotNet with `[MemoryDiagnoser]`, four cases: `async Task<int>` completing synchronously, `async Task<int>` suspending, `async ValueTask<int>` completing synchronously, `async ValueTask<int>` suspending. Explain all four numbers with Concept 13. Then add `<Features>runtime-async=on</Features>` on a .NET 11 leg and compare.

**Exercise 4 — Reproduce the deadlock, three ways.** (a) A WPF or WinForms app with `.Result` in a click handler. (b) An xUnit v2 async test doing the same. (c) An ASP.NET Core app showing the deadlock does *not* occur, but recording the thread cost under load. Write down what differs and why.

**Exercise 5 — Break the memory model.** Write the Concept 34 stop-flag loop and run it in Release. Confirm the infinite loop. Fix it four ways — `volatile`, `Volatile.Read`, `lock`, `CancellationToken` — and inspect the emitted assembly for each in `[DisassemblyDiagnoser]` or sharplab. If you have access to an Arm64 machine (a Graviton instance, an Apple Silicon Mac), write a store–load reordering test and see it fire there and not on x64. That last part is a genuinely memorable experience.

**Exercise 6 — The `ConcurrentDictionary` factory.** Instrument `GetOrAdd`'s factory with an `Interlocked` counter and hammer the same key from 32 tasks. Observe the factory running more than once. Fix with `Lazy<T>` and confirm it runs exactly once. Then make the factory async and handle the "faulted task cached forever" problem.

**Exercise 7 — Four fan-out strategies, measured.** 10,000 items, each a 50 ms simulated HTTP call. Implement with (a) `Task.WhenAll` unbounded, (b) `SemaphoreSlim` + `WhenAll`, (c) `Parallel.ForEachAsync`, (d) a bounded `Channel<T>` with N consumers. Measure wall time, peak memory, peak thread count, and peak concurrent "connections." Write a one-paragraph recommendation. This is close to a real design-review task.

**Exercise 8 — Async hang forensics.** Deliberately build a hang: a `SemaphoreSlim` whose `Release` is skipped by an exception path. Reproduce it, take a dump with `dotnet-dump collect`, and find the cause using `clrthreads`, `syncblk`, and `!dumpasync`. Write down what each command showed and what it didn't. Repeat with a `Monitor` ordering deadlock and compare the signatures.

**Exercise 9 — Backpressure end to end.** Build producer → bounded channel → slow consumer. Instrument channel depth. Show what happens with `FullMode.Wait`, `DropOldest`, and an unbounded channel (run that one until the container OOMs — set a low memory limit so it's quick). Then add a `RateLimiter` at the front and show the rejection path. One page on which policy you'd pick for telemetry, for orders, and for audit logs.

**Exercise 10 — The concurrency review write-up (one page).** For a real service you know: every fan-out point and its limit (and where the limit came from); every place that blocks; the ThreadPool configuration and its justification; every shared mutable state and what guards it; the DI lifetimes and whether each registered type's thread-safety contract matches; the shutdown path and whether in-flight work drains; the concurrency metrics you emit; and the three changes with the best ratio of risk removed to effort. This is very close to a real architect take-home.

---

## Free resources

### Primary sources — the runtime's own documentation

| Resource | What it covers | Why read it |
|---|---|---|
| [.NET Memory Model spec](https://github.com/dotnet/runtime/blob/main/docs/design/specs/Memory-model.md) | Atomicity, ordering, volatile semantics, what CoreCLR guarantees beyond ECMA-335 | **The single highest-value free document in this module.** Short, authoritative, and almost nobody has read it |
| [Book of the Runtime (BOTR)](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/README.md) | The CLR's internal design | Background for Parts B and D |
| [BOTR — Threading](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/threading.md) | Thread management, suspension, synchronization internals | The runtime's own view of Part C |
| [Runtime Async design docs](https://github.com/dotnet/runtime/blob/main/docs/design/features/runtime-async.md) | The .NET 11 runtime-async feature design | Concept 16 from the source |
| [C# spec — lock object (`System.Threading.Lock`)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/lock-object) | The language change behind `Lock` | Concept 43 from the design document |
| [Async ValueTask pooling proposal](https://github.com/dotnet/runtime/issues/13633) | The design discussion behind pooled async state machines | Concept 15's background |
| [ECMA-335 — CLI specification](https://www.ecma-international.org/publications-and-standards/standards/ecma-335/) | Partition I §12.6 covers the memory model | The "guaranteed vs implementation detail" question |
| [dotnet/runtime — `ThreadPool` source](https://github.com/dotnet/runtime/tree/main/src/libraries/System.Private.CoreLib/src/System/Threading) | `PortableThreadPool`, `ThreadPoolWorkQueue`, `HillClimbing` | The injection heuristic, as code |

### Stephen Toub — the canonical deep dives

| Resource | What it covers |
|---|---|
| [How Async/Await Really Works in C#](https://devblogs.microsoft.com/dotnet/how-async-await-really-works/) | **Start here.** The definitive article: state machines, builders, schedulers, contexts, from first principles |
| [Async/await FAQ](https://devblogs.microsoft.com/dotnet/how-async-await-really-works/) · [ValueTask FAQ](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/) | Concept 14's rules, from the person who designed the type |
| [ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/) | **The complete answer to Concept 24.** Every misconception, addressed |
| [An Introduction to System.Threading.Channels](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/) | Concept 57 from the designer |
| [Performance Improvements in .NET 11](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-11/) | Runtime async internals, `ExecutionContext` elision, async codegen |
| [Performance Improvements in .NET 10](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/) · [.NET 9](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9/) · [.NET 8](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/) | The threading sections of each are a year-by-year history of this module |
| [Parallel Programming with .NET (the old blog, archived)](https://devblogs.microsoft.com/pfxteam/) | The PFX team's archive: `TaskCompletionSource` guidance, `AsyncLock`, custom awaitables, `Task.Run` vs `StartNew` |
| ["Task.Run vs Task.Factory.StartNew"](https://devblogs.microsoft.com/pfxteam/task-run-vs-task-factory-startnew/) | Concept 32's canonical reference |

### Stephen Cleary — the practitioner's canon

| Resource | What it covers |
|---|---|
| [blog.stephencleary.com](https://blog.stephencleary.com/) | The best single blog on .NET async; twelve years of it |
| [Async and Await (intro)](https://blog.stephencleary.com/2012/02/async-and-await.html) | The clearest short introduction there is |
| [Don't Block on Async Code](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html) | **Concept 31, the original.** The deadlock explained once and for all |
| [Async/Await Best Practices (MSDN Magazine)](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming) | Avoid `async void`, async all the way, `ConfigureAwait` — still the best summary |
| [There Is No Thread](https://blog.stephencleary.com/2013/11/there-is-no-thread.html) | **Concept 4 in 900 words.** Read it, then quote it in an interview |
| [A Tour of Task (series)](https://blog.stephencleary.com/2014/04/a-tour-of-task-part-0-overview.html) | Every member of `Task`, explained with when-to-use guidance |
| [Async OOP (series)](https://blog.stephencleary.com/2013/01/async-oop-0-introduction.html) | Constructors, properties, disposal, events — the awkward cases |
| [Nito.AsyncEx](https://github.com/StephenCleary/AsyncEx) | `AsyncLock`, `AsyncManualResetEvent`, `AsyncProducerConsumerQueue` (Concept 51) |
| [Cancellation (series)](https://blog.stephencleary.com/2022/03/cancellation-1-overview.html) | The most complete treatment of Concepts 19–20 anywhere |

### Microsoft Learn — official documentation

| Resource | What it covers |
|---|---|
| [Asynchronous programming with async and await](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/) | The official learning path |
| [Task asynchronous programming model](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/task-asynchronous-programming-model) | What the compiler does, officially |
| [Async return types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/async#return-types) · [Generate and consume async streams](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-stream) | Concepts 14 and 58 |
| [Debug ThreadPool Starvation](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/debug-threadpool-starvation) | **A full worked tutorial for Concepts 28 and 73**, with a sample app that starves on demand |
| [Debug a deadlock in .NET](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/debug-deadlock) | `syncblk`, `clrstack`, the whole workflow |
| [.NET diagnostic tools overview](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/) | `dotnet-counters`, `dotnet-dump`, `dotnet-trace`, `dotnet-stack` |
| [Well-known EventCounters](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/available-counters) · [Built-in metrics](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/built-in-metrics-runtime) | The names in Concept 72 |
| [ThreadPool config settings](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/threading) | `ThreadPoolMinThreads`, `ThreadPoolMaxThreads`, autorelease |
| [Cancellation in managed threads](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads) | The official version of Concept 19 |
| [System.Threading.Channels](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels) | Concept 57 |
| [`Parallel.ForEachAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel.foreachasync) · [Data parallelism](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/data-parallelism-task-parallel-library) | Concepts 61–62 |
| [Rate limiting an HTTP handler](https://learn.microsoft.com/en-us/dotnet/core/extensions/http-ratelimiter) · [ASP.NET Core rate limiting middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit) | Concept 70 |
| [`TimeProvider`](https://learn.microsoft.com/en-us/dotnet/api/system.timeprovider) · [Testing time-dependent code](https://learn.microsoft.com/en-us/dotnet/core/extensions/timeprovider) | Concept 69 |
| [Background tasks with hosted services](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services) | Concept 67 |
| [What's new in the .NET 11 runtime](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/runtime) | **Current-state reading for Concept 16** — runtime async, in detail |
| [What's new in .NET 9](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-9/overview) | `System.Threading.Lock`, `Task.WhenEach` |
| [Managed threading best practices](https://learn.microsoft.com/en-us/dotnet/standard/threading/managed-threading-best-practices) | Dated in places, still the official position |
| [`System.Collections.Concurrent`](https://learn.microsoft.com/en-us/dotnet/standard/collections/thread-safe/) | Concepts 53–54 |
| [TPL Dataflow](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/dataflow-task-parallel-library) | Concept 60 |
| [System.IO.Pipelines](https://learn.microsoft.com/en-us/dotnet/standard/io/pipelines) | Concept 59 |

### Memory model and lock-free programming

| Resource | What it covers |
|---|---|
| [Preshing on Programming](https://preshing.com/) | **The best free writing on memory ordering anywhere.** Start with [Memory Barriers Are Like Source Control](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/) and [Acquire and Release Semantics](https://preshing.com/20120913/acquire-and-release-semantics/) |
| [Preshing — Weak vs Strong Memory Models](https://preshing.com/20120930/weak-vs-strong-memory-models/) | Exactly why x64 and Arm64 differ (Concept 35) |
| [Sasha Goldshtein / Vance Morrison — "Understand the Impact of Low-Lock Techniques"](https://learn.microsoft.com/en-us/archive/msdn-magazine/2005/october/understand-the-impact-of-low-lock-techniques-in-multithreaded-apps) | The classic .NET memory-model article; old but foundational |
| [Joe Duffy's blog](https://joeduffyblog.com/) | The author of *Concurrent Programming on Windows*; deep posts on locks, memory models, and the CLR's threading design |
| [Igor Ostrovsky — Volatile reads and writes](https://igoro.com/archive/volatile-keyword-in-c-memory-model-explained/) | A clear, short explanation of Concept 37 |
| [Martin Thompson — Mechanical Sympathy](https://mechanical-sympathy.blogspot.com/) | False sharing, cache lines, and the hardware reality behind Concept 40 |
| [LMAX Disruptor technical paper](https://lmax-exchange.github.io/disruptor/disruptor.html) | The canonical study in high-performance lock-free design and cache-line awareness |
| [The Little Book of Semaphores](https://greenteapress.com/wp/semaphores/) — Allen Downey | Free book; classic synchronization problems worked in detail |

### Talks and video

| Resource | What it covers |
|---|---|
| [Deep .NET (YouTube, Stephen Toub + Scott Hanselman)](https://www.youtube.com/@dotnet) | The async, `Task`, and threading episodes are effectively this module on video |
| [.NET Conf archives](https://www.dotnetconf.net/) | The .NET 9/10/11 performance and runtime-async sessions |
| [NDC Conferences on YouTube](https://www.youtube.com/@NDC) | Search "async" — several excellent deep dives, including Cleary's and Toub's |
| [Dotnetos](https://dotnetos.org/) | .NET performance courses and talks; much of it free |
| [Konrad Kokosa — async internals talks](https://tooslowexception.com/) | Async state machines from the memory-management angle |

### Tools

| Tool | What it's for |
|---|---|
| [sharplab.io](https://sharplab.io/) | **See the generated state machine instantly** (Exercise 2) |
| [BenchmarkDotNet](https://benchmarkdotnet.org/) | `[MemoryDiagnoser]`, multi-runtime jobs, `[ThreadingDiagnoser]` for lock contention |
| [dotnet-counters / dotnet-dump / dotnet-trace / dotnet-stack](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/) | The diagnostic workflow in Concept 73 |
| [PerfView](https://github.com/microsoft/perfview) | Thread Time view, blocked-time attribution, ThreadPool events |
| [Speedscope](https://www.speedscope.app/) | Flame graphs from `dotnet-trace` |
| [Microsoft.VisualStudio.Threading.Analyzers](https://github.com/microsoft/vs-threading) | VSTHRD analyzers — the strictest async ruleset available, plus `JoinableTaskFactory` |
| [`Microsoft.Extensions.TimeProvider.Testing`](https://www.nuget.org/packages/Microsoft.Extensions.TimeProvider.Testing) | `FakeTimeProvider` (Concept 69) |
| [Nito.AsyncEx](https://github.com/StephenCleary/AsyncEx) | Async coordination primitives (Concept 51) |
| [dotnet-gcdump](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-gcdump) | Finding retained state machines and leaked registrations |

### Books (not free, listed for completeness)

- **Concurrency in C# Cookbook, 2nd edition** — Stephen Cleary (O'Reilly). The single most practical book on this material; recipe-shaped and reliably correct.
- **Concurrent Programming on Windows** — Joe Duffy. Dated on APIs, unmatched on fundamentals: memory models, lock implementation, scheduler behaviour.
- **Pro .NET Memory Management, 2nd edition** — Kokosa, Nasarre, Gosse. Chapter coverage of async state machines and their memory profile.
- **C# in Depth, 4th edition** — Jon Skeet. The chapter on async is the best language-level explanation in print.
- **The Art of Multiprocessor Programming, 2nd edition** — Herlihy, Shavit, Luchangco, Spear. The theory: linearizability, lock-freedom, wait-freedom, the actual algorithms.
- **Designing Data-Intensive Applications** — Kleppmann. Chapters 7–9 for where in-process concurrency meets distributed concurrency.

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| What async is for | Not occupying a thread while waiting; throughput and density, not speed |
| Concurrency vs parallelism | Dealing with many things vs doing many things; async is a way to get the first without the second |
| Thread cost | ~1 MB reserved stack, kernel object, ~1–10 µs switch; a *blocked* thread costs no CPU but is still scarce |
| I/O-bound vs CPU-bound | Async for the first, parallelism for the second; the wrong tool actively hurts |
| "There is no thread" | During an async I/O wait the kernel owns the operation; a pool thread reappears only at completion |
| Thread budget | threads = arrival rate × thread-hold time; 500 rps × 200 ms = 100 threads blocking, 5 async |
| Async contagion | A promise-returning signature is the type system telling the truth; blocking to hide it reinstates the cost |
| Green threads | Ran the experiment, rejected it: interop can't be parked, movable stacks break exact scanning, ecosystem cost |
| `Task` | State flags + result + continuation object; promise-style tasks have no delegate and need no thread |
| `TaskCompletionSource` | Always `RunContinuationsAsynchronously`, always `Try*` |
| The compiler transform | Struct state machine + builder; `Start` runs `MoveNext` synchronously; locals crossing an await get hoisted |
| `await` mechanically | `IsCompleted` → `OnCompleted` → `GetResult`; awaitable is a shape, not an interface |
| Fast path | An already-complete awaitable never suspends, never allocates, never hops threads |
| Async allocations | Zero on the synchronous path; one box on first suspension; the box *is* the `Task` |
| `ValueTask` | Await once, never twice, never concurrently, never `.Result`; `AsTask()` if you need more |
| Runtime async | .NET 11 preview; runtime owns suspension; real stack traces, fewer allocations; BCL already compiled with it |
| Async exceptions | Captured into the `Task`, rethrown at `await` with `ExceptionDispatchInfo`; `.Result` wraps in `AggregateException` |
| `async void` | No task to observe; exceptions crash the process; event handlers only, with try/catch |
| Cancellation | Cooperative; pass the token everywhere; dispose registrations and linked sources |
| Timeout vs cancel | `WaitAsync` abandons; a token propagated into the call actually stops the work |
| `SynchronizationContext` | The "resume here" policy; UI has one, **ASP.NET Core does not** |
| `TaskScheduler` | The "run here" policy; `Default` is the ThreadPool; `StartNew` uses `Current`, `Task.Run` uses `Default` |
| `ExecutionContext` | Flows `AsyncLocal` across awaits **regardless of `ConfigureAwait`**; copy-on-write; it's how `Activity` flows |
| `ConfigureAwait(false)` | Opts out of resuming on the captured context. Doesn't change threads, suppress exceptions, or affect `AsyncLocal` |
| `ConfigureAwaitOptions` | .NET 8: `SuppressThrowing`, `ForceYielding` |
| ThreadPool structure | Global FIFO queue + per-thread LIFO local queues + work stealing; one pool for everything |
| Thread injection | Free to `MinThreads` (= cores), then ~1 thread per 500 ms; 92 extra threads ≈ 46 seconds |
| Starvation signature | **Low CPU + high latency + climbing thread count + growing queue** |
| I/O completions | Dispatched onto the same worker pool; starvation therefore blocks its own recovery |
| Blocking | `.Result`/`.Wait()` wrap exceptions; `GetAwaiter().GetResult()` doesn't; all of them burn a thread |
| The deadlock | Single-threaded context + blocked caller; ASP.NET Core removed the deadlock, not the cost |
| `Task.Run` | Move CPU work off a thread you must protect; never to fake async in a library |
| Memory model | Compiler + JIT + CPU all reorder; single-thread semantics preserved, cross-thread visibility is not |
| x64 vs Arm64 | x64 is TSO (only store→load reorders); Arm64 is weak — latent bugs surface on Graviton |
| Atomicity | References and aligned word-size primitives don't tear; `Guid`, `decimal`, multi-field structs do |
| `volatile` | Acquire on read, release on write; **not** atomic; not allowed on `long`/`double` |
| `Interlocked` | Full fence; `CompareExchange` is the primitive; CAS loops must be side-effect free |
| Double-checked locking | Needs `volatile`; just use `Lazy<T>` with `ExecutionAndPublication` |
| False sharing | 64-byte cache lines; two hot variables on one line destroys scaling; pad or don't share |
| `lock`/`Monitor` | Thin lock in the header, inflates to a sync block; reentrant, thread-affine; never on `this`/`Type`/`string` |
| `System.Threading.Lock` | .NET 9; `EnterScope()` returns a `ref struct`; analyzers catch conversion to `object` |
| `await` in `lock` | Forbidden: `Monitor` is thread-affine; use `SemaphoreSlim` — and it isn't reentrant |
| `SemaphoreSlim` | Async gate and concurrency limiter; `Release()` in `finally`; pass the token to `WaitAsync` |
| `ReaderWriterLockSlim` | Only for long reads and rare writes; usually lose to `lock` or an immutable snapshot |
| Deadlock types | Lock ordering, sync-over-async, pool exhaustion, convoy, async reentrancy |
| `ConcurrentDictionary` | Lock-free reads, striped writes; `GetOrAdd`'s factory runs outside the lock and can run twice → `Lazy<T>` |
| `ConcurrentBag` | Thread-affine with stealing; wrong for cross-thread producer/consumer |
| Immutable + CAS | Build the new state, swap the reference atomically; readers are lock-free and always consistent |
| `Channel<T>` | Bounded by default; `FullMode` is an architectural decision; not durable — it's a buffer, not a queue |
| `IAsyncEnumerable` | Backpressure by construction; `[EnumeratorCancellation]` is required or `WithCancellation` does nothing |
| Fan-out tool | `WhenAll` for a few, `Parallel.ForEachAsync` for a bounded collection, `Channel` for a stream, Dataflow for stages |
| `WhenAll`/`WhenAny`/`WhenEach` | `WhenAll` waits for all and rethrows one; `WhenAny` leaves the rest unobserved; `WhenEach` is .NET 9 and O(n) |
| Bounded concurrency | Every fan-out has a number derived from pool size / downstream capacity / cores / memory — × instance count |
| Backpressure | Block, reject, drop, or spill. "Grow the buffer" is deferring to the OOM killer |
| Fire-and-forget | Loses exceptions, ignores shutdown, unbounded, captures the scope. Use a queue or a channel |
| `BackgroundService` | Singleton — use `IServiceScopeFactory` per unit of work; unhandled exceptions stop the host since .NET 6 |
| Shutdown | `ApplicationStopping` → token cancelled → drain → `StopAsync` with `ShutdownTimeout` (30 s) < grace period |
| Timers | `PeriodicTimer` (no overlap) over `System.Threading.Timer`; `TimeProvider` + `FakeTimeProvider` for tests |
| Rate limiting | `System.Threading.RateLimiting`: concurrency, token bucket, fixed/sliding window; `QueueLimit = 0` usually |
| Cross-instance | In-process locks don't survive scale-out; optimistic concurrency, idempotency keys, leases with fencing |
| Key metrics | Thread count, queue length, completed items, lock contention, your own in-flight gauge |
| Diagnostic workflow | counters (classify) → `dotnet-stack` (locate) → dump + `!dumpasync`/`syncblk` (confirm) → trace (intermittent) |
| Testing | `TimeProvider`, stress runs, `Interlocked` counters in assertions, an Arm64 CI leg; never `Thread.Sleep` |
| Library rules | No sync-over-async, no fake async, take a token, `ConfigureAwait(false)`, document thread-safety |
| When not to be async | Nothing blocks, low concurrency, CPU-bound, sub-millisecond, or you'd be faking it |

---

## Progress

Module 15 complete — **Phase 4 is halfway.** Module 14 gave you the memory model of execution; this module gave you the execution model of time. Together they are the runtime foundation everything else in the phase builds on.

This module closes several loops from earlier phases:

- **Module 6's Little's Law** now has a thread expression (Concept 5): arrival rate × thread-hold time is your thread requirement, and it is the arithmetic that justifies async in one sentence.
- **Module 6's overload protection** and **Module 11's queue semantics** now have their in-process form: `Channel<T>` with a `BoundedChannelFullMode` is the same decision — block, reject, drop, or spill — at a smaller scale.
- **Module 13's cascading failures** now have a concurrency mechanism to go with the GC one: a slow dependency causes blocking, blocking causes starvation, starvation causes queueing, queueing causes timeouts, timeouts cause retries, and the loop closes. "It was ThreadPool starvation" is now a specific, diagnosable claim with a signature you can recite.
- **Module 13's bulkheads** are `SemaphoreSlim`, `Parallel.ForEachAsync`'s degree limit, and — for genuinely blocking dependencies — dedicated threads outside the shared pool.
- **Module 9's distributed coordination** now has its local counterpart, and the boundary between them is explicit (Concept 71): a `lock` is an optimization over a database guarantee, never a replacement for one.
- **Module 14's state machine allocations** (Concept 51 there) are Concept 13 here, with the full picture of when the box happens and why it is a single object.

Threads left open on purpose:

- **`Span<T>`, `Memory<T>`, and measurement discipline** — BenchmarkDotNet methodology, statistical rigour, Native AOT in practice — are **Module 17**. The `[ThreadingDiagnoser]` and the benchmark designs in Exercises 1, 3, and 7 are its groundwork.
- **The DI lifetime rules** behind the captive-dependency problem named in Concepts 52 and 67 are diagnosed properly in **Module 18**, along with the ASP.NET Core middleware pipeline and how a request flows onto a pool thread in the first place.
- **EF Core's thread-affinity and change-tracking behaviour**, and the async story for `SaveChangesAsync`, are **Module 19**.
- **Polly's concurrency primitives** — bulkhead isolation, rate limiting, hedging — are **Module 25**, where Concept 64's limits become policies.
- **Service Bus/Event Hubs concurrency settings** (`MaxConcurrentCalls`, prefetch, partition-level ordering) are **Module 27**.
- **Exporting these counters and alerting on them** — the OpenTelemetry side of Concept 72 — is **Module 28**.

Next in the curriculum: **Module 16 — Modern C#** (C# 14 field-backed properties and extension members, records, pattern matching, primary constructors, and what fluency here signals to an interviewer), which is a lighter module by design after two heavy ones, and which picks up the language-level threads left by `readonly struct`, `ref struct`, and the `Lock` type special-casing seen here.
