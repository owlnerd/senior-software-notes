# Senior Software Engineer / Architect Interview Mastery

*A comprehensive .NET & C# preparation curriculum*

---

## How This Works

This is the master roadmap for our prep. We'll move through it **one module at a time** — I'll teach the concept, show how it actually gets asked, connect it to the .NET/C# angle, and give you something concrete to reason through or practice. This curriculum is weighted toward what actually separates senior/staff candidates from mid-level ones: system design judgment, distributed-systems vocabulary, deep .NET/C# mastery, and — for the architect track specifically — the ability to make decisions that survive contact with an organization.

Skip anything you're already solid on, ask to go deeper anywhere, or jump to a module out of order. This is your course, not a script.

---

## Part 0 — Orientation

**What's actually being scored.** Past the mid-level, interviewers aren't grading whether your answer is "correct" — most system design questions don't have one right answer. They're grading judgment: how you handle ambiguity, whether you back trade-offs with real numbers, and whether your reasoning would survive being challenged by someone senior to you. Coding rounds shift from "can you solve it" to "would I want this code in production."

**Two different loops.** Depending on the target, you're preparing for one of two distinct evaluation shapes:

|  | Senior / Staff IC | Architect |
| --- | --- | --- |
| Core rounds | Coding screen, 1–2 system design rounds, a deep technical round, behavioral | Design-document review, a brownfield/incremental-redesign exercise, a round with the engineers who'd implement the decision, a stakeholder trade-off conversation |
| What's judged | Can you design and build it | Can you decide, document it, and get people to align behind it |
| Common failure mode | Weak trade-off reasoning, no numbers to back decisions | A technically sound design that ignores team size, delivery constraints, or cost |

If you know which one you're actually aiming for, tell me and I'll weight the plan accordingly — the modules below cover both, with the architect-specific track broken out separately (Part 7).

**Your baseline.** Because your DS&A is already strong, Module 36 (the coding round) is deliberately short — a calibration pass, not a grind.

---

## The Throughline: One Framework, Every Design Question

Every system design round, regardless of company, collapses into the same seven moves. This is the skeleton we'll hang every design discussion on:

1. **Requirements** — functional + non-functional (scale, latency, availability, consistency needs)
2. **Estimation** — QPS, storage, bandwidth, back-of-envelope math
3. **API design** — the contract between components
4. **Data model** — entities, relationships, what gets stored where
5. **High-level design** — boxes and arrows, the end-to-end flow
6. **Deep dive** — 1–3 components, in real depth, with trade-offs (this carries the most weight)
7. **Wrap-up** — bottlenecks, failure modes, what you'd do with more time

Module 3 covers this properly: time budgets per step, and what senior-level answers do differently at each one.

---

## The Curriculum

### Phase 1 — Foundations & Calibration

1. [What top companies actually score (rubric literacy)](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%201%20%E2%80%94%20Foundations%20%26%20Calibration/module-01-rubric-literacy.md)
2. [Senior IC vs. Architect: calibrating your prep to your actual target](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%201%20%E2%80%94%20Foundations%20%26%20Calibration/module-02-senior-ic-vs-architect.md)

### Phase 2 — The System Design Method

3. [The 7-step delivery framework, applied end to end](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%202%20-%20The%20System%20Design%20Method/module-03-seven-step-delivery-framework.md)
4. [Requirements gathering & the non-functional questions that signal seniority](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%202%20-%20The%20System%20Design%20Method/module-04-requirements-gathering-nfrs.md)
5. [Back-of-envelope estimation](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%202%20-%20The%20System%20Design%20Method/module-05-back-of-envelope-estimation.md) — the latency numbers you need cold (disk seek, memory read, cross-region round trip, and how to use them under pressure)

### Phase 3 — Distributed Systems Theory

6. [Scalability fundamentals](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%203%20-%20Distributed%20Systems%20Theory/module-06-scalability-fundamentals.md) — horizontal vs. vertical scaling, statelessness, load balancing strategies
7. [CAP theorem, PACELC, and consistency models](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%203%20-%20Distributed%20Systems%20Theory/module-07-cap-pacelc-consistency-models.md) (strong / eventual / causal)
8. [Replication & partitioning](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%203%20-%20Distributed%20Systems%20Theory/module-08-replication-partitioning.md) — leader-follower, multi-leader, quorum reads/writes, sharding, consistent hashing
9. [Consensus & coordination](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%203%20-%20Distributed%20Systems%20Theory/module-09-consensus-coordination.md) — Raft, Paxos, ZooKeeper/etcd, distributed locks, vector clocks, CRDTs
10. [Caching strategy](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%203%20-%20Distributed%20Systems%20Theory/module-10-caching-strategy.md) — cache-aside, write-through/write-back, CDNs, invalidation, Redis patterns
11. [Messaging & event-driven systems](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%203%20-%20Distributed%20Systems%20Theory/module-11-messaging-event-driven-systems.md) — queues vs. streams, delivery guarantees, dead-letter queues, the outbox pattern
12. [Data storage deep dive](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%203%20-%20Distributed%20Systems%20Theory/module-12-data-storage-deep-dive.md) — SQL vs. NoSQL trade-offs, indexing, ACID, distributed transactions (2PC vs. Saga)
13. [Reliability patterns](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%203%20-%20Distributed%20Systems%20Theory/module-13-reliability-patterns.md) — circuit breakers, retries/backoff, bulkheads, active-active vs. active-passive redundancy

### Phase 4 — .NET & C# Technical Mastery

14. [CLR & memory internals](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%204%20-%20.NET%20%26%20C%23%20Technical%20Mastery/module-14-clr-memory-internals.md) — stack vs. heap, generational GC, Server vs. Workstation GC, value vs. reference types, boxing
15. [Async/await & concurrency](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%204%20-%20.NET%20%26%20C%23%20Technical%20Mastery/module-15-async-await-concurrency.md) — `Task` internals, `SynchronizationContext`, ThreadPool starvation, Channels, `async void` and deadlock traps
16. [Modern C#](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%204%20-%20.NET%20%26%20C%23%20Technical%20Mastery/module-16-modern-csharp.md) — C# 14 field-backed properties & extension members, records, pattern matching, primary constructors, and what fluency here signals to an interviewer
17. [Performance engineering](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%204%20-%20.NET%20%26%20C%23%20Technical%20Mastery/module-17-performance-engineering.md) — `Span<T>`/`Memory<T>`, BenchmarkDotNet methodology, Native AOT, JIT tiered compilation, allocation-conscious design
18. [ASP.NET Core internals](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%204%20-%20.NET%20%26%20C%23%20Technical%20Mastery/module-18-aspnet-core-internals.md) — middleware pipeline, DI container & lifetime pitfalls (singleton/scoped/transient), minimal APIs vs. controllers
19. [EF Core deep dive](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%204%20-%20.NET%20%26%20C%23%20Technical%20Mastery/module-19-ef-core-deep-dive.md) — change tracking, query translation, the N+1 problem, compiled queries, migration strategy at scale

### Phase 5 — .NET Architecture Patterns

20. [Clean Architecture & layering](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%205%20-%20.NET%20Architecture%20Patterns/module-20-clean-architecture-layering.md) — what it buys you, where teams misuse it
21. [Modular monolith vs. microservices](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%205%20-%20.NET%20Architecture%20Patterns/module-21-modular-monolith-vs-microservices.md) — the actual decision criteria, not dogma
22. [DDD tactical & strategic patterns](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%205%20-%20.NET%20Architecture%20Patterns/module-22-ddd-tactical-strategic-patterns.md) — bounded contexts, aggregates, domain events
23. [CQRS & MediatR](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%205%20-%20.NET%20Architecture%20Patterns/module-23-cqrs-mediatr.md) — when the complexity tax is worth paying
24. [Event sourcing](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%205%20-%20.NET%20Architecture%20Patterns/module-24-event-sourcing.md) — what it solves, what it costs
25. [Resilience in .NET](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%205%20-%20.NET%20Architecture%20Patterns/module-25-resilience-dotnet-polly.md) — Polly: retry, circuit breaker, timeout, rate limiter, hedging

### Phase 6 — Cloud & Platform Architecture

26. [Compute choices](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%206%20-%20Cloud%20%26%20Platform%20Architecture/module-26-compute-choices.md) — App Service vs. AKS vs. Azure Functions/Container Apps, and when each wins
27. [Messaging & data platform](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%206%20-%20Cloud%20%26%20Platform%20Architecture/module-27-messaging-data-platform.md) — Service Bus, Event Grid/Event Hubs, Cosmos DB (partitioning & consistency levels)
28. [Observability](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%206%20-%20Cloud%20%26%20Platform%20Architecture/module-28-observability.md) — OpenTelemetry, distributed tracing, SLO/SLI/error budgets, structured logging
29. [Security architecture](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%206%20-%20Cloud%20%26%20Platform%20Architecture/module-29-security-architecture.md) — OAuth2/OIDC, Microsoft Entra ID, Key Vault, threat modeling (STRIDE)

### Phase 7 — The Architect-Specific Track

30. [Writing and defending a design document](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%207%20-%20The%20Architect-Specific%20Track/module-30-writing-defending-design-document.md)
31. [ADRs and the C4 model](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%207%20-%20The%20Architect-Specific%20Track/module-31-adrs-c4-model.md) — documenting decisions so they survive personnel changes
32. [Brownfield thinking](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%207%20-%20The%20Architect-Specific%20Track/module-32-brownfield-thinking.md) — incremental migration, the strangler fig pattern
33. [Cost, build-vs-buy, and technical debt](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%207%20-%20The%20Architect-Specific%20Track/module-33-cost-build-vs-buy-technical-debt.md) — the language executives respond to

### Phase 8 — Behavioral & Leadership

34. [STAR, calibrated to seniority](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%208%20-%20Behavioral%20%26%20Leadership/module-34-star-calibrated-to-seniority.md) — senior answers describe what you delivered; staff/architect answers explain why it mattered and how it shaped the system
35. [Your story bank](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%208%20-%20Behavioral%20%26%20Leadership/module-34-star-calibrated-to-seniority.md) — mapping real experience to the 6–8 stories that cover most behavioral questions (conflict, failure, leading through ambiguity, technical disagreement, mentoring, influence without authority)

### Phase 9 — Coding Round, Right-Sized

36. [What senior-level coding rounds actually test, given your DS&A is already strong](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%209%20-%20Coding%20Round%2C%20Right-Sized/module-36-senior-coding-rounds.md) — code quality, edge cases, testing, API design over algorithmic trivia

### Phase 10 — Applied Practice

37. [Worked system design problems with .NET-specific implementation notes](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%2010%20-%20Applied%20Practice/module-37-worked-system-design-problems.md) — (URL shortener, rate limiter, notification system, distributed cache, order/payment system)
38. [Mock interview structure & a self-scoring rubric](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%2010%20-%20Applied%20Practice/module-38-mock-interview-structure-self-scoring-rubric.md)
39. [Company/role research checklist](https://github.com/owlnerd/senior-software-notes/blob/main/Phase%2010%20-%20Applied%20Practice/module-39-company-role-research-checklist.md)

---

## Suggested Pacing

No fixed calendar — go at whatever pace your timeline allows. As a default, 2–3 modules per session keeps each one deep enough to be useful without turning into a lecture. Phases 3–5 are the heaviest; don't rush them. If you've got an actual interview date, tell me and we'll compress or reorder around it.

## Further Reading (optional, alongside our sessions)

- *Designing Data-Intensive Applications* — Martin Kleppmann (the standard reference behind Phase 3)
- *System Design Interview*, Vol. 1 & 2 — Alex Xu
- *Clean Architecture* — Robert C. Martin
- *Clean Architecture with .NET* — Dino Esposito
- Microsoft's official .NET application architecture guides (learn.microsoft.com/dotnet/architecture)
