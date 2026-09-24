# Module 19 — EF Core Deep Dive
*Phase 4: .NET & C# Technical Mastery · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **EF Core is three machines — a model built once per process, a query compiler that turns an expression tree into SQL and caches it by the tree's shape, and a change tracker that holds an in-memory object graph and diffs it into a batched write — and every EF Core problem in production is one of those three machines doing exactly what you told it to, rather than what you meant.**

That reframing matters because the naive picture of EF Core is a translation layer: write LINQ, get rows, change objects, call `SaveChanges`. A mid-level candidate can recite "use `AsNoTracking` for reads" and "avoid N+1." A senior candidate can tell you *why* a page that returns 50 rows issues 51 queries (lazy loading through a proxy, or a `foreach` over navigations outside the projection), why the same query returns 50 rows but materializes 4,000 objects (a cartesian explosion from two collection `Include`s), why memory grows until the pod dies (a long-lived context whose identity map never releases, or a `DbContext` captured by a singleton), why the database's plan cache is full of near-identical plans (inlined constants, or a `Where` closure over a captured local that EF decided to constant-fold), why `SaveChanges` takes 400 ms for 20 rows (`DetectChanges` walking a 40,000-entity graph, or 20 separate round trips because batching was disabled), why a retry policy silently broke transactional integrity (the execution strategy and a user-initiated transaction), and why a rolling deploy took the site down for 90 seconds (a migration that rewrote a table while the old code was still running).

This module closes Phase 4 and settles debts from two earlier modules. **Module 12 (Concept 60)** gave you the architect-level summary of EF Core — pooling, retries, tracking, compiled queries, and where the ORM stops — and explicitly deferred the internals here. **Module 18 (Part E)** established scoped lifetimes, captive dependencies, disposal, and the unit-of-work boundary; this module is what actually lives inside that scope. It also uses everything from Modules 14–17: the identity map is a **retention structure** (Module 14), the async query pipeline is where sync-over-async kills you (Module 15), the model is where records, `required`, and init-only properties collide with materialization (Module 16), and every claim in Part E is one you should be able to measure (Module 17).

It shows up in five places in an interview loop: the **deep technical round** ("what happens between `ToListAsync()` and the rows coming back?"), the **debugging round** ("this endpoint got 10× slower after we added a field — why?"), the **code-review round** ("what's wrong with this repository?"), the **design round** (transactional boundaries, aggregates, read models, migration strategy), and the **platform round** (zero-downtime schema change, multi-tenancy, connection resiliency, cost).

**Current platform state (verified September 2026).** EF Core 10 was released in November 2025 and is a Long Term Support release, supported until 10 November 2028; it requires the .NET 10 SDK to build and the .NET 10 runtime to run, and will not run on .NET Framework. EF Core 11 is the next release and is currently in development; it requires the .NET 11 SDK and runtime. .NET 11 RC1 shipped on 8 September 2026 with a go-live licence, ahead of GA on 10 November 2026. What changed that matters here:

**EF Core 10 (production-current, LTS):**

- **Complex types matured into the recommended replacement for owned entity types.** EF 10 added optional complex types, mapping complex types to JSON columns, and struct support. The reason is semantic, not cosmetic: owned types are entity types, so they carry reference semantics and a hidden identity — assigning a customer's billing address from their shipping address fails, and comparing two owned instances in LINQ compares identities rather than contents; complex types have value semantics, so assignment copies properties and comparison compares contents. Bulk assignment of owned types is unsupported, whereas complex types fully support `ExecuteUpdateAsync` in EF 10, and users already modelling JSON or table splitting with owned types are advised to switch.
- **A new default translation for parameterized collections.** EF ≤ 8.0 inlined collection contents as SQL constants, which generated a different SQL string per collection and caused plan-cache misses and bloat; EF 8 changed to a single JSON-array parameter unpacked with `OPENJSON`, which fixed the plan-cache problem but deprived the planner of cardinality information; EF 10 introduces a new default where each value becomes its own scalar parameter, and pads the parameter list so that a collection of 8 values generates SQL with 10 parameters, reducing the number of distinct SQL strings. The strategy is configurable globally via `UseParameterizedCollectionMode` and per query via `EF.Constant`.
- **Named query filters.** EF previously supported only one query filter per entity type, making it impossible to disable one filter selectively; EF 10 lets you name filters and disable individual ones with `IgnoreQueryFilters(["SoftDeletionFilter"])`.
- **`ExecuteUpdateAsync` accepts a regular, non-expression lambda**, replacing the manual `Expression.Lambda` construction previously required to build conditional setters dynamically. `ExecuteUpdateAsync` also gained support for relational JSON columns, but only when the type is mapped as a complex type, not as an owned entity.
- **SQL Server / Azure SQL 2025 surface:** full support for the `vector` data type and `VECTOR_DISTANCE()` via a `SqlVector<float>` property, and full support for the native `json` data type — with `UseAzureSql` or compatibility level 170 or higher, EF defaults to the new JSON type, and existing `nvarchar` JSON columns are changed to `json` on the first migration unless you opt out.
- **Two security defaults worth naming.** EF 10 redacts inlined constants from logged SQL by default, replacing them with `?`, because inlined parameters previously leaked potentially sensitive values into logs even though parameter values were never logged. And an analyzer now warns when string concatenation is performed inside a raw SQL method invocation such as `FromSqlRaw`.
- Also: `LeftJoin`/`RightJoin` LINQ operators, consistent ordering across split queries, which previously omitted the key column from the subquery's `ORDER BY` and could return incorrect data, custom default-constraint names, and on Cosmos full-text search, `Rrf` reciprocal-rank-fusion hybrid search, and vector similarity search exiting preview.

**EF Core 11 (in development; GA with .NET 11 on 10 November 2026):**

- **Two query-shape optimizations with real numbers.** EF 11 prunes unnecessary joins to reference navigations in split queries, and stops adding redundant reference-navigation keys to `ORDER BY` because a reference navigation's key is functionally determined by the parent's key; a common split-query scenario showed a 29% improvement and a single-query scenario 22%. It also strips no-op `CAST` expressions — common with value converters — which can prevent the database from using an index on the column.
- **Vector columns are no longer selected by default.** `SqlVector<T>` columns are excluded from `SELECT` when materializing entities, since vectors are usually ingested and searched but not read back; a minimal benchmark showed almost 9× against a local database and around 22× against a remote Azure SQL database. EF 11 also adds `VectorSearch()` with `WithApproximate()` over SQL Server vector indexes — both still experimental in SQL Server itself.
- **Migrations in team environments.** The model snapshot now records the ID of the latest migration, so two developers creating migrations on divergent branches produce a source-control merge conflict that alerts the team to resolve the divergence. `dotnet ef database update --add` creates and applies a migration in one step by compiling it with Roslyn at runtime, for Aspire and containerized scenarios. Foreign keys can now be modelled in EF while suppressing the database constraint via `ExcludeForeignKeyFromMigrations()`, for legacy databases and synchronization scenarios.
- **Complex types on TPT/TPC inheritance**, lambda chaining for complex-property configuration, and keys and indexes on scalar properties nested inside complex types — including paths inside JSON columns.
- **Cosmos got a new engine.** The provider now serializes with `System.Text.Json` and no longer depends on `Newtonsoft.Json`; the `__jObject` shadow property has been removed. The consequence is a breaking change: unmapped JSON properties in a document are ignored on read and lost if the entity is saved. The provider also now uses transactional batches by default (controlled by `AutoTransactionBehavior`), supports bulk execution, and exposes session-token get/set for read-your-writes across instances. Sync I/O via the Cosmos provider has been fully removed.
- **Two defaults that will surprise people on upgrade.** `UseSqlServer` now defaults to compatibility level 160 (SQL Server 2022) instead of 150, and `Migrate` throws when no migrations are found in the assembly instead of logging and returning. EF 11 also moves to `Microsoft.Data.SqlClient` 7.0, which removes Entra ID authentication dependencies from the core package — apps using `ActiveDirectoryDefault` or managed identity must add `Microsoft.Data.SqlClient.Extensions.Azure` separately.
- Also: `FullJoin`, `GroupBy` translation and materialization improvements, `MaxByAsync`/`MinByAsync`, `EF.Functions.JsonPathExists()`, `JSON_CONTAINS` translation at compatibility level 170 replacing the `OPENJSON` form for `Contains` over primitive collections, SQL Server JSON indexes and full-text catalog/index creation through migrations, plus `FreeTextTable`/`ContainsTable` with rank, temporal period columns mapped to CLR properties, and `ChangeTracker.GetEntriesForState()`, which returns tracked entries in selected states without forcing a `DetectChanges` pass.

**And the thing not to overstate:** NativeAOT support and query precompilation remain highly experimental and not suited for production use; the current support should be viewed as infrastructure toward the final feature, publishing still reports trimming and AOT warnings, and Microsoft recommends against deploying EF NativeAOT applications in production. Precompiled queries work by static analysis generating C# interceptors, so any source change invalidates them and the `dotnet ef dbcontext optimize` + `dotnet publish` steps belong in CI/CD rather than the inner loop. Note the useful half: precompiled queries can be used *without* NativeAOT — `dotnet ef dbcontext optimize --precompile-queries` generates a compiled model and interceptors that remove query-compilation overhead from startup.

This module has eight jobs:

1. **Make the model a design artifact, not configuration.** Conventions, the fluent API, keys, converters, relationships, inheritance, and the model cache — including the multi-tenancy trap that turns a cache into a memory leak.
2. **Make the query pipeline mechanical.** Expression tree → preprocessing → translation → SQL → shaper, with the query cache keyed by tree shape and the parameterization rules that decide your database's plan cache.
3. **Kill N+1 and cartesian explosion properly** — not with a rule of thumb, but by being able to predict the SQL for any LINQ you write.
4. **Explain change tracking as a data structure.** The identity map, snapshots, `DetectChanges`, graph attachment, and the write batch that `SaveChanges` produces.
5. **Get `DbContext` lifetime right in every host** — request-scoped, pooled, factory-created, background, Blazor — and know precisely what pooling resets.
6. **Turn performance into measurement.** A cost model, a diagnostic procedure, and the specific levers with the specific numbers.
7. **Make schema evolution safe at scale.** The snapshot, the history table, migration locking, expand/contract, backfills, and drift detection in CI.
8. **Raise it to architecture.** Where the ORM stops, whether to wrap it, how it meets DDD, multi-tenancy, testing strategy, and the anti-patterns reviewers look for.

Eight framings to carry through:

1. **The model is immutable and expensive; the query cache and the change tracker are not.** Know which of the three is being rebuilt when something is slow.
2. **LINQ is not the query. The expression tree is.** Everything EF does — caching, parameterization, translation — is a function of the tree's *shape*, and shape is not what your source code looks like.
3. **Tracking is a retention decision.** Every tracked entity is an object graph the GC cannot collect and a snapshot that `DetectChanges` must walk. That is the whole cost story.
4. **`SaveChanges` is a compiler.** It takes a graph, sorts it topologically, and emits a batch. If the emitted batch surprises you, your graph is not what you think.
5. **The unit of work has a boundary, and it is yours to choose.** Not "one per request" by reflex — one per *consistency requirement*.
6. **Set-based work does not belong in the change tracker.** `ExecuteUpdate`/`ExecuteDelete`, raw SQL, and bulk libraries exist because loading 100,000 entities to change one column is an architectural error, not a tuning problem.
7. **Schema changes are deployments, not code.** Forward-only, expand/contract, reviewed as SQL, applied by something that isn't your web app's startup path.
8. **The ORM is a productivity tool with a defined stopping point.** Knowing where it stops — and saying so unprompted — is the architect signal.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | Three machines | Model (per process), query compiler (cached by tree shape), change tracker (per context) |
| 2 | Model building | Conventions → annotations → fluent API; last wins; the result is frozen |
| 3 | The model cache | Built once, cached by `IModelCacheKeyFactory`; a per-tenant key is a leak |
| 4 | Compiled models | Moves model building to build time; for very large models and cold start |
| 5 | Entity / owned / complex | Complex types have value semantics; prefer them over owned types in EF 10+ |
| 6 | Keys and value generation | Identity, sequence, GUID, and the clustered-index consequence |
| 7 | Value converters | Converters need comparers; mutable converted values silently stop being tracked |
| 8 | Relationships | Principal/dependent, required/optional, and the four delete behaviours |
| 9 | Inheritance | TPH default (one table, discriminator); TPT joins; TPC unions |
| 10 | Query filters | Applied to every query on that entity; named in EF 10; the leak risk is a missing filter |
| 11 | `IQueryable` | Deferred; composition is free, enumeration is the round trip |
| 12 | The query pipeline | Preprocess → navigation expansion → translate → optimize → SQL + shaper |
| 13 | The query cache | Keyed by expression-tree shape; a constant in the tree is part of the key |
| 14 | Parameterization | Captured variables parameterize; literals inline; `EF.Constant`/`EF.Parameter` override |
| 15 | Client evaluation | Only in the final projection; anything else throws by design |
| 16 | Projection | `Select` a DTO: less data, no tracking, fewer joins, a stable contract |
| 17 | Loading related data | Eager (`Include`), explicit (`Load`), lazy (proxies — usually a mistake in a web app) |
| 18 | Cartesian explosion | Two collection `Include`s multiply rows; `AsSplitQuery` trades it for round trips |
| 19 | N+1 | Lazy loading, or navigation access after materialization, or a loop with a query in it |
| 20 | Tracking modes | Tracking, `AsNoTracking`, `AsNoTrackingWithIdentityResolution` |
| 21 | Compiled queries | `EF.CompileAsyncQuery` skips cache lookup; precompiled queries move it to build |
| 22 | Raw SQL | `FromSql` composes and parameterizes; `FromSqlRaw` is yours to sanitize |
| 23 | Pagination | Offset degrades with depth; keyset is stable and index-friendly |
| 24 | Reading the SQL | `ToQueryString`, `TagWith`, logging, interceptors, `EnableSensitiveDataLogging` |
| 25 | The state manager | An identity map keyed by primary key; one instance per key per context |
| 26 | `DetectChanges` | O(tracked entities × properties) snapshot diff, run on most public APIs |
| 27 | Notification entities | `INotifyPropertyChanged` / proxies remove the scan, at the cost of a base class |
| 28 | Disconnected graphs | `Attach`/`Update` mark the whole graph; `TrackGraph` gives per-node control |
| 29 | Entity states | Added, Modified, Deleted, Unchanged, Detached — and who sets them |
| 30 | `SaveChanges` | Detect → cascade → topologically sort → batch → execute → accept changes |
| 31 | Batching | SQL Server default `MaxBatchSize` 42, `MinBatchSize` 4; `OUTPUT` returns keys |
| 32 | Transactions | One implicit transaction per `SaveChanges`; explicit for multi-call units |
| 33 | Concurrency | `rowversion`/`xmin`/business column → `DbUpdateConcurrencyException` → resolve deliberately |
| 34 | `ExecuteUpdate`/`Delete` | Set-based, immediate, and invisible to the change tracker |
| 35 | Interceptors | Command, connection, transaction, save-changes, materialization — the outbox hook |
| 36 | Domain events | Dispatch inside the transaction, or through the outbox; never fire-and-forget |
| 37 | `DbContext` identity | Unit of work + identity map; lifetime = consistency boundary |
| 38 | Scoped registration | One per request by default; the captive-dependency and disposal rules from Module 18 |
| 39 | Context pooling | Resets EF-known state only; your own fields survive into the next request |
| 40 | `IDbContextFactory` | For Blazor, background services, parallel work, and anything not request-scoped |
| 41 | Thread safety | One operation at a time; `EnableThreadSafetyChecks` costs something to keep you honest |
| 42 | Connections | ADO.NET pools connections; EF pools contexts; connections open late, close early |
| 43 | Resiliency | `EnableRetryOnFailure` + user transactions ⇒ wrap in `strategy.ExecuteAsync` |
| 44 | Multiple contexts | One context per bounded context; migrations and history tables follow |
| 45 | The cost model | Network + database work dominate; EF overhead is micro- to low-milliseconds |
| 46 | Diagnosis | SQL logs, `ToQueryString`, metrics, OpenTelemetry, and the database's own plan tooling |
| 47 | Indexing for EF SQL | Covering indexes for projections; watch parameter sniffing and converted columns |
| 48 | Query anti-patterns | `ToList` then filter, `Count()` in a loop, `Include` everything, string-built predicates |
| 49 | Bulk operations | Batching → `ExecuteUpdate` → bulk library → `SqlBulkCopy`, in that order |
| 50 | Streaming | `AsAsyncEnumerable` for large reads; buffering holds the whole result set |
| 51 | Caching around EF | HybridCache at the application layer beats a transparent second-level cache |
| 52 | JSON and complex types | Denormalize the parts that are always read together; index the paths you filter on |
| 53 | Vector and AI | `SqlVector<float>`, `VECTOR_DISTANCE`, EF 11's `VectorSearch` and default exclusion |
| 54 | Startup cost | Model build + first query dominate cold start; compiled/precompiled queries fix both |
| 55 | What a migration is | A C# class pair plus a snapshot plus a row in `__EFMigrationsHistory` |
| 56 | The snapshot | The model as of the last migration; merge conflicts are regenerated, never hand-merged |
| 57 | Applying migrations | Idempotent script or bundle in the pipeline; not `Migrate()` at app startup |
| 58 | Migration locking | EF 9+ takes a database lock; user transactions and pending model changes now warn/throw |
| 59 | Expand/contract | Add tolerant → deploy → backfill → switch → deploy → remove; never one step |
| 60 | Data migrations | Backfills are jobs with batching and restartability, not `Up()` bodies |
| 61 | Multiple contexts | Separate history tables, separate ownership, explicit `--context` |
| 62 | Non-EF schema | Views, functions, and procedures mapped in, managed out |
| 63 | Testing migrations | Testcontainers + `has-pending-model-changes` + a drift check in CI |
| 64 | Rollback | Forward-only in practice; `Down()` is for local iteration |
| 65 | Where the ORM stops | EF for the write model, Dapper/SQL for demanding reads |
| 66 | Repository over EF | `DbSet` is already one; wrap for the aggregate boundary, not for "abstraction" |
| 67 | `IQueryable` leakage | Returning `IQueryable` from a repository exports the ORM into your domain |
| 68 | EF and DDD | Backing fields, private setters, owned/complex value objects, aggregate-sized loads |
| 69 | Multi-tenancy | Discriminator / schema / database, and the model-cache consequence of each |
| 70 | Testing strategy | InMemory lies; SQLite in-memory half-lies; Testcontainers tells the truth |
| 71 | Provider choice | Relational vs Cosmos vs in-memory; capability differences are not cosmetic |
| 72 | Anti-patterns | Repository-over-repository, `Update()` everywhere, lazy loading in APIs, startup migration |

---

# Part A — The model: what EF knows before it ever sees a query

---

## Concept 1 — Three machines, three lifetimes

Everything in this module is easier if you hold three objects separate in your head, because they have different lifetimes, different costs, and different failure modes.

| Machine | Type | Lifetime | Cost | Failure mode |
|---|---|---|---|---|
| **The model** | `IModel` | Built once, cached for the process | Expensive to build (reflection over every entity type), cheap to read | Rebuilt per request → CPU burn and unbounded cache growth |
| **The query cache** | entries in an `IMemoryCache` | Process-wide, keyed by expression-tree shape | Compilation is expensive; lookup is cheap | Cache miss on every call → compile on every request |
| **The change tracker** | `StateManager` on a `DbContext` | Per `DbContext` instance | Grows with every tracked entity | Retained too long → memory growth and quadratic `DetectChanges` |

EF Core uses `IMemoryCache` for internal caching operations such as query compilation and model building. That single sentence explains a whole class of production incidents: anything that changes the *cache key* for the model or a query turns a cached artifact into a per-call computation, and because it's a cache rather than a dictionary you get memory pressure rather than a clean failure.

The first two are **process-scoped and shared**. The third is **context-scoped and private**. Almost every "EF is slow" story resolves to putting work in the wrong one — rebuilding a process-scoped thing per request, or letting a context-scoped thing live for the process.

A fourth object matters for lifetime reasoning but isn't EF's: the **database connection**, pooled by the ADO.NET provider. Concept 42 separates it from context pooling, because interviewers conflate them constantly.

---

## Concept 2 — Model building: three sources, one frozen result

The model is built from three layers, applied in increasing order of precedence:

1. **Conventions** — EF inspects your CLR types and infers almost everything. A property named `Id` or `<Type>Id` becomes the key; a reference property plus an `<Name>Id` scalar becomes a relationship; `string` becomes `nvarchar(max)`; a nullable CLR type becomes a nullable column. Conventions are not magic: they are a documented, replaceable pipeline (`IConvention`), and EF 7+ lets you add and remove them via `ConfigureConventions`.
2. **Data annotations** — `[Key]`, `[Required]`, `[MaxLength(200)]`, `[Table]`, `[Column]`, `[ComplexType]`, `[Owned]`. Cheap, local, and they couple your domain types to a persistence library.
3. **The fluent API** — `OnModelCreating`, usually split into `IEntityTypeConfiguration<T>` classes and pulled in with `ApplyConfigurationsFromAssembly`. Highest precedence, zero coupling to the domain type, and the only place some things can be expressed at all.

**The architectural position to hold:** use conventions for the 80% that's genuinely conventional, and the fluent API in per-entity configuration classes for everything else. Reach for data annotations only when the attribute is *also* meaningful to your domain or to a validation layer. The reason is not purity — it's that `IEntityTypeConfiguration<Order>` is a single file you can hand a new engineer to explain how orders persist, and attributes scattered across a domain type are not.

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);

        builder.Property(o => o.Status)
               .HasConversion<string>()          // store the enum name, not its ordinal
               .HasMaxLength(32);

        builder.ComplexProperty(o => o.ShippingAddress);   // table splitting, value semantics

        builder.HasMany(o => o.Lines)
               .WithOne()
               .HasForeignKey(l => l.OrderId)
               .OnDelete(DeleteBehavior.Cascade);

        builder.Navigation(o => o.Lines)
               .UsePropertyAccessMode(PropertyAccessMode.Field);   // respects the private field

        builder.Property<byte[]>("RowVersion").IsRowVersion();     // shadow concurrency token
    }
}
```

Two things are true of the result and worth saying out loud in an interview: **the model is immutable once built**, and **it is validated once, at build time**. A mapping mistake is therefore a startup-time exception rather than a runtime one — the same "misconfiguration should be a deployment failure" principle from Module 18, Concept 4, applied to persistence.

---

## Concept 3 — The model cache, and the multi-tenancy trap

EF builds the model on first use and caches it. The cache key comes from `IModelCacheKeyFactory`, whose default implementation keys on the `DbContext` **type** (plus a design-time flag). One context type, one model, one build — for the life of the process.

This is also one of the most expensive mistakes available in EF Core, and it appears almost exclusively in multi-tenant code. A tempting pattern is to make the model depend on runtime state:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // DANGEROUS: the model now depends on a per-request value
    modelBuilder.Entity<Order>().ToTable($"Orders_{_tenant.Schema}");
}
```

Because the cache key is still the context *type*, whichever tenant happens to warm the cache first wins and every other tenant gets the wrong model — a correctness bug, and a cross-tenant data bug at that. The "fix" everyone reaches for next is a custom `IModelCacheKeyFactory` that includes the tenant:

```csharp
public sealed class TenantModelCacheKeyFactory : IModelCacheKeyFactory
{
    public object Create(DbContext context, bool designTime)
        => context is AppDbContext app
            ? (context.GetType(), app.TenantSchema, designTime)
            : context.GetType();
}
```

That is correct, and it is a memory-shaped time bomb: you now build and retain **one full model per tenant**. With 20 tenants that's fine. With 20,000 it is not, and the failure arrives as slowly-growing RSS plus CPU spent on model building, which looks nothing like a database problem.

**The senior answer:** keep the model tenant-independent. Put tenancy in the *data* (a `TenantId` column plus a global query filter — Concept 10) or in the *connection* (schema/database per tenant resolved at connection time, with an identical model). Only vary the model per tenant when tenants genuinely have different schemas, and then bound the number, because you are now paying model-cache memory proportional to tenant count. Concept 69 returns to this with the full decision table.

---

## Concept 4 — Compiled models

For very large models — hundreds of entity types, thousands of properties — the first-use model build is measured in seconds, and it lands squarely in your cold-start budget (Module 17, Concept 63; Module 18, Concept 7).

```bash
dotnet ef dbcontext optimize --output-dir Generated --namespace MyApp.Generated
```

This generates C# that constructs the model directly, skipping the reflection-and-convention pipeline at runtime. It's discovered automatically, or wired explicitly:

```csharp
options.UseModel(AppDbContextModel.Instance);
```

What to know beyond the mechanics:

- **It is a build-time artifact of your model**, so it must be regenerated whenever the model changes. If you forget, you run with a stale model — which is why regeneration belongs in the build, not in a developer's memory.
- **It historically had gaps** (global query filters, lazy loading, some value-converter scenarios were unsupported in early versions). Check the current limitations for your version rather than assuming; the direction of travel is toward parity.
- **It is not a query optimization.** It removes model-building time, not query-compilation time. Concept 21 (compiled and precompiled queries) is the other half of the cold-start story.

The judgement: compiled models are worth it when your model is genuinely large *and* your startup budget is tight (serverless, scale-to-zero, aggressive rolling deploys). For a 40-entity service that runs three replicas continuously, it's ceremony.

---

## Concept 5 — Entity types, owned types, and complex types

Three ways to model "a thing inside another thing," and EF 10 changed the recommendation.

**Entity type.** Has identity (a key), lives in its own table by default, is tracked independently, and can be referenced by many things. `Order`, `Customer`, `Product`.

**Owned entity type** (EF Core 2.0+). Modelled with `OwnsOne`/`OwnsMany`, mapped either into the owner's table (table splitting) or to a JSON column. The catch is structural: it's still an *entity* type underneath, with identity and reference semantics. Assigning a customer's billing address from their shipping address fails, because the same entity instance can't be referenced twice; comparing two owned instances in a LINQ query compares identity rather than contents; and bulk assignment via `ExecuteUpdate` isn't supported.

**Complex type** (EF 8 for table splitting, substantially completed in EF 10). Modelled with `ComplexProperty`. Complex types have no identity of their own and value semantics: assignment copies properties, comparison compares contents, and `ExecuteUpdateAsync` works. EF 10 added optional complex types, mapping to JSON, and struct support; users already using owned entity types for table splitting or JSON are advised to switch. EF 11 extends them further — complex types and JSON columns now work on TPT/TPC inheritance, properties can be configured by chaining member access, and keys and indexes can target scalar properties nested inside complex types, including paths inside JSON columns.

```csharp
// EF 10+: the value-object mapping you actually want
modelBuilder.Entity<Customer>(b =>
{
    b.ComplexProperty(c => c.ShippingAddress);                 // → columns on Customers
    b.ComplexProperty(c => c.BillingAddress, a => a.ToJson()); // → one json column
});
```

**Why this matters for the interview and for your architecture:** a complex type is how a DDD *value object* should map (Module 22 develops this). `Money`, `Address`, `DateRange`, `EmailAddress` — none of them has identity, all of them should compare by value, and modelling them as owned entities has been quietly producing surprising behaviour in .NET codebases for years. Being able to say *why* — value semantics versus reference semantics, not "the new API is nicer" — is the signal.

One caveat to carry: optional complex types currently require at least one required property on the complex type, and collections of structs aren't supported.

---

## Concept 6 — Keys, alternate keys, shadow properties, value generation

**Primary keys.** Conventions find `Id` or `<Type>Id`. Composite keys need the fluent API (`HasKey(x => new { x.A, x.B })`). Every entity type EF tracks needs a key; keyless types (`HasNoKey()`) exist for query-only mappings to views and raw SQL results, and they cannot be tracked or saved.

**Value generation** is where a modelling choice becomes a database performance choice, and it connects directly to Module 12:

| Strategy | EF configuration | What you get | Cost |
|---|---|---|---|
| Identity column | default for `int`/`long` keys on SQL Server | Monotonic, narrow, cache-friendly clustered index | Last-page insert contention at very high write rates |
| Sequence / HiLo | `UseHiLo()` | Client-side key allocation in blocks → keys known before insert, better batching | Gaps; a shared sequence object |
| `Guid` (random) | `ValueGeneratedOnAdd` on a `Guid` | Keys known before insert, globally unique | **Index fragmentation** — random inserts scatter across a clustered index |
| `Guid` v7 / sequential | `Guid.CreateVersion7()` (.NET 9+), or `NEWSEQUENTIALID()` | Time-ordered, so inserts stay near the end of the index | Leaks creation time; still 16 bytes |

The senior framing: **an `int` identity key is the default for a reason, and a random `Guid` clustered key is the single most common self-inflicted write-performance wound in .NET data models.** If you need client-generated keys (offline clients, distributed ID generation, avoiding a round trip before you know the ID), use a time-ordered UUID — `Guid.CreateVersion7()` in .NET 9+ — or keep the `Guid` as a non-clustered alternate key and cluster on something sequential.

**Alternate keys** (`HasAlternateKey`) create a unique constraint and can serve as the target of a foreign key. Use them for natural keys you must enforce (an `OrderNumber`, an external system's ID) while keeping a surrogate primary key.

**Shadow properties** exist in the model but not on the CLR type — `builder.Property<DateTime>("LastModifiedUtc")`. They're the right tool for persistence concerns that shouldn't pollute your domain: a `RowVersion` concurrency token, audit stamps, a discriminator, or a `TenantId` on an aggregate that has no business reason to know about tenancy. Read and write them with `EF.Property<T>(entity, "Name")` and set them in a `SaveChanges` interceptor (Concept 35).

---

## Concept 7 — Value converters, and the comparer you forgot

A value converter maps a CLR type to a provider type:

```csharp
builder.Property(o => o.Status).HasConversion<string>();                 // enum ⇄ string
builder.Property(o => o.Total).HasConversion(
    m => m.Amount, a => new Money(a, "EUR"));                            // value object ⇄ decimal
builder.Property(o => o.Tags).HasConversion(
    t => JsonSerializer.Serialize(t, opts),
    s => JsonSerializer.Deserialize<List<string>>(s, opts)!);            // collection ⇄ json string
```

Three consequences that separate people who have used converters from people who have shipped them:

**1. Converted columns can break index usage.** Once a column's stored form differs from the CLR form, some LINQ operations either can't translate or translate into an expression the database can't seek on. The classic shape is a `CAST` in the predicate. EF 11 helps here — it detects and strips no-op `CAST` expressions, which commonly occurred with value-converted properties whose CLR type has an implicit conversion to the provider type, and such casts can prevent the database from using an index on the column — but the general rule stands: **check the generated SQL for any converted column you filter on.**

**2. Mutable converted values need a value comparer, or changes are silently lost.** The change tracker snapshots by copying. For a `List<string>` behind a JSON converter, the snapshot is a reference to the *same list*, so mutating the list in place leaves snapshot and current value identical and `DetectChanges` reports no change. The fix is an explicit `ValueComparer` that snapshots by deep copy and compares by structure:

```csharp
var comparer = new ValueComparer<List<string>>(
    (a, b) => a!.SequenceEqual(b!),
    v => v.Aggregate(0, (h, s) => HashCode.Combine(h, s.GetHashCode())),
    v => v.ToList());                       // snapshot = a real copy

builder.Property(o => o.Tags).HasConversion(/* ... */, comparer);
```

This is a genuinely good interview question and a genuinely common production bug: *"we save the entity, no exception, and the change isn't there."* Be able to explain it in terms of snapshot semantics, not as a quirk.

**3. Prefer the model over the converter where the database has a real type.** On modern SQL Server, a primitive collection or a complex type maps to the native `json` data type at compatibility level 170 or with `UseAzureSql`, which is queryable and indexable — strictly better than a hand-rolled string converter. Use converters for genuine impedance mismatches (an enum you want stored as text, a strongly-typed ID wrapping a `Guid`), not as a JSON serializer of last resort.

---

## Concept 8 — Relationships, and the four delete behaviours

EF models a relationship as: a **principal** entity, a **dependent** entity, a **foreign key** on the dependent, and up to two **navigations**. Optionality follows the FK's nullability. That's the whole abstraction; everything else is configuration.

The part people get wrong is delete behaviour, because it has two audiences — the database's `ON DELETE` clause *and* EF's in-memory behaviour when a principal is deleted or a relationship is severed.

| `DeleteBehavior` | In the database | In EF, when the principal is deleted | Use for |
|---|---|---|---|
| `Cascade` | `ON DELETE CASCADE` | Dependents loaded into the context are also deleted | True composition — order lines, aggregate internals |
| `ClientCascade` | `NO ACTION` | EF deletes loaded dependents; the database won't | Cycles the database refuses (SQL Server multiple-cascade-path errors) |
| `Restrict` | `NO ACTION` | EF does nothing; `SaveChanges` fails on the FK violation | References you want to protect — don't delete a `Customer` with orders |
| `SetNull` | `ON DELETE SET NULL` | EF nulls the FK on loaded dependents | Optional references — a `Ticket.AssignedUserId` when the user is removed |

Two rules worth stating in an interview:

- **Cascade means "part of," not "related to."** Cascade from `Order` to `OrderLine` is correct — a line has no life without its order. Cascade from `Customer` to `Order` is almost always wrong, because deleting a customer should not silently destroy financial history. The EF configuration is where an aggregate boundary becomes physical (Module 22).
- **EF's cascade only touches what is loaded.** If the dependents aren't in the change tracker, EF emits nothing for them and relies on the database's `ON DELETE`. A `Restrict`/`NO ACTION` model plus unloaded dependents produces an FK violation at `SaveChanges`, not a clean domain error. Decide which layer enforces the rule, and make the other one agree.

**Required vs. optional on the CLR side** is a nullable-reference-types question in modern C# (Module 16): with NRTs enabled, `public Customer Customer { get; set; } = null!;` reads as required and EF maps it that way, while `public Customer? Customer { get; set; }` reads as optional. Your nullable annotations are now load-bearing *schema*, which is a nice example to have ready for "how have modern C# features changed your data layer?"

---

## Concept 9 — Inheritance: TPH, TPT, TPC

Three ways to map a class hierarchy, with materially different query shapes.

| Strategy | Schema | `SELECT` for the base type | `SELECT` for one derived type | Notes |
|---|---|---|---|---|
| **TPH** (default) | One table, all columns, a `Discriminator` | One table scan, no joins | Same table + discriminator predicate | Derived properties must be **nullable** in the database |
| **TPT** | One table per type, joined by shared key | Base table + `LEFT JOIN` per derived table | Base `JOIN` derived | Clean schema, non-null constraints preserved; the join cost is real and grows with hierarchy depth |
| **TPC** | One table per concrete type, no base table | `UNION ALL` across all concrete tables | Single table, no join | Best read performance for the "one concrete type" case; duplicated columns; keys must be unique across tables |

**The default is TPH, and for most models the default is right.** It's a single table, a single index strategy, and no joins. The objections are schema purity (you get nullable columns that are logically required) and width (a 12-type hierarchy produces a very wide table).

**Choose TPT** when the hierarchy is wide, derived types have many distinct properties, and you genuinely need database-level non-null constraints — accepting a join per derived type on every polymorphic query.

**Choose TPC** when you rarely query polymorphically and mostly query one concrete type at a time. Note the key constraint: since there's no shared table, key values must be unique *across* all concrete tables, which usually means `Guid` keys or a shared sequence (`UseHiLo`).

The interview move: don't recite the three. Say **"TPH by default because polymorphic queries stay single-table; TPT when the derived types diverge enough that nullable columns become a correctness problem; TPC when we almost never query the base type, and then I'd use a shared sequence for keys."** That's the trade-off, delivered as a decision.

---

## Concept 10 — Global query filters, soft delete, and multi-tenancy

A global query filter is a predicate EF appends to **every** query against an entity type:

```csharp
// EF 10+: named filters that can be disabled individually
modelBuilder.Entity<Blog>()
    .HasQueryFilter("SoftDeletionFilter", b => !b.IsDeleted)
    .HasQueryFilter("TenantFilter", b => b.TenantId == _tenantProvider.TenantId);
```

Before EF 10 there was one filter per entity type, which made it hard to have multiple filters and disable only some of them; named filters let each be managed separately, so `IgnoreQueryFilters(["SoftDeletionFilter"])` disables exactly one. That closes a real safety hole: the old `IgnoreQueryFilters()` was all-or-nothing, so an admin screen that wanted to see deleted rows also turned off **tenant isolation** — a cross-tenant data leak hiding inside a convenience method. Module 12, Concept 53 flagged this; EF 10 is the fix.

Five things to know precisely:

1. **Filters apply to the root of a query and to navigations**, so an `Include` of a filtered type is also filtered. They do *not* apply to `FromSqlRaw` results or to `ExecuteUpdate`/`ExecuteDelete` in the way you might hope — verify, don't assume.
2. **A filter that captures a service is captured per model build, not per query** — which is why tenant filters conventionally capture a field on the `DbContext` itself. EF re-evaluates the field's value per query because it becomes a parameter, but the *expression* is baked into the model. Capturing a scoped service directly in `OnModelCreating` is a captive dependency (Module 18, Concept 42) wearing a different hat.
3. **Filters are defined per entity type, not inherited across the model.** Every tenant-scoped entity needs one. A missing filter on one entity type is a data-leak bug, so generate them from a marker interface rather than hand-writing 60 of them:

```csharp
foreach (var type in modelBuilder.Model.GetEntityTypes()
             .Where(t => typeof(ITenantScoped).IsAssignableFrom(t.ClrType)))
{
    modelBuilder.Entity(type.ClrType)
        .HasQueryFilter("TenantFilter", BuildTenantPredicate(type.ClrType));
}
```

4. **Soft delete is more expensive than it looks.** Module 12 said it and it bears repeating here in EF terms: deleted rows stay in every index, every query pays the predicate, the table never shrinks, and GDPR erasure is not satisfied by an `IsDeleted` flag. If you use it: filtered indexes (`WHERE IsDeleted = 0`), a *named* filter so it composes with tenancy, and a real hard-delete job after a retention window.
5. **Defence in depth belongs in the database.** A query filter is application-layer isolation, and it is one `IgnoreQueryFilters` or one raw SQL query away from being bypassed. For genuine multi-tenant isolation, pair it with Row-Level Security in the database. Saying "query filter plus RLS, because the filter protects against mistakes and RLS protects against bypass" is an architect-level answer.

---

# Part B — The query pipeline: from expression tree to SQL

---

## Concept 11 — `IQueryable`, deferred execution, and what actually causes a round trip

`IQueryable<T>` is `IEnumerable<T>` plus two things: an **expression tree** describing the query, and a **provider** that knows how to execute it. Every LINQ operator you chain doesn't run anything — it wraps the tree in another node.

```csharp
var q = db.Orders.Where(o => o.Status == OrderStatus.Open);   // nothing happened
q = q.OrderByDescending(o => o.PlacedAtUtc);                  // still nothing
q = q.Take(20);                                               // still nothing
var page = await q.ToListAsync();                             // ← now: compile + execute
```

The operators that **execute**: `ToListAsync`, `ToArrayAsync`, `ToDictionaryAsync`, `FirstAsync`/`FirstOrDefaultAsync`, `SingleAsync`/`SingleOrDefaultAsync`, `AnyAsync`, `AllAsync`, `CountAsync`, `LongCountAsync`, `SumAsync`/`MinAsync`/`MaxAsync`/`AverageAsync`, `AsAsyncEnumerable` (on enumeration), and `foreach`. Everything else composes.

Two practical consequences:

**Composition is free; enumeration is not.** This is what makes conditional query building clean:

```csharp
IQueryable<Order> q = db.Orders;
if (status is not null)     q = q.Where(o => o.Status == status);
if (customerId is not null) q = q.Where(o => o.CustomerId == customerId);
if (from is not null)       q = q.Where(o => o.PlacedAtUtc >= from);
var results = await q.OrderBy(o => o.Id).Take(50).ToListAsync();
```

One round trip, one SQL statement, and — importantly for Concept 13 — a *different* cached query per combination of branches, because each combination is a different expression tree.

**The moment you leave `IQueryable` you leave the database.** `db.Orders.ToList().Where(...)` loads the table and filters in memory. This is the single most common EF performance bug in the wild, and it's nearly always an accident of an extension method or a helper that returns `IEnumerable<T>`. In C#, `IEnumerable<T>`'s `Where` takes `Func<T, bool>` and `IQueryable<T>`'s takes `Expression<Func<T, bool>>` — the type system is telling you where evaluation happens, if you read it. Concept 67 makes this an architectural point.

**Async is not optional here.** Every one of these executions is I/O. Module 15's rules apply without modification: `await` the async overload, pass the `CancellationToken` through (`ToListAsync(ct)`), and never `.Result` a query — a synchronous database call on a request thread is exactly the ThreadPool starvation shape from Module 15, Concept 40. EF 11 even ships an analyzer for a related trap: `ToAsyncEnumerable()` on an `IQueryable` wraps *synchronous* enumeration in an `IAsyncEnumerable`, so `AsAsyncEnumerable()` is the correct call.

---

## Concept 12 — The compilation pipeline, stage by stage

When you enumerate a query, EF runs it through a pipeline. You don't need to name every internal class, but you do need the shape, because every query bug lives at one of these stages.

1. **Query cache lookup.** The expression tree is hashed into a cache key (Concept 13). On a hit, everything below is skipped and you go straight to execution.
2. **Preprocessing / normalization.** Method calls are normalized, subqueries are lifted, `DefaultIfEmpty` patterns are recognized, query filters (Concept 10) are injected, and parameters are extracted from captured closures (Concept 14).
3. **Navigation expansion.** `o.Customer.Name` and `Include(o => o.Lines)` are rewritten into joins or correlated subqueries against the real tables. This is the stage where a cartesian explosion (Concept 18) is born.
4. **Translation.** The rewritten tree is walked and converted into a provider-agnostic SQL expression tree. Anything that cannot be translated fails here — loudly (Concept 15).
5. **SQL-level optimization.** Provider-independent and provider-specific passes simplify the tree. EF 11's work lives mostly here: pruning unneeded joins to reference navigations, dropping redundant `ORDER BY` columns, stripping no-op `CAST`s, and simplifying `CASE` expressions into `NULLIF` or nothing.
6. **SQL generation + shaper compilation.** The provider emits the SQL string, and EF compiles a **shaper**: a delegate that reads a `DbDataReader` row and produces your objects — including fixing up navigations and, for tracked queries, consulting the identity map (Concept 25).
7. **Execution.** Connection opened, command executed, reader consumed by the shaper, connection returned to the pool.

Two things follow from this diagram.

**Query compilation is expensive**, which is why step 1 exists at all: EF caches queries by the query tree's shape so that queries with the same structure reuse internally cached compilation outputs.

**Materialization is generated code, not reflection.** The shaper is a compiled delegate. This matters for two reasons: it's why EF's per-row overhead is small compared to the database round trip (Concept 45), and it's why NativeAOT is hard — generating and JIT-compiling that delegate is exactly what AOT forbids, which is the entire motivation for precompiled queries (Concept 21).

---

## Concept 13 — The query cache is keyed by tree *shape*

This is the concept that explains a surprising number of incidents, so it's worth stating precisely: **EF caches compiled queries by the shape of the expression tree, and a constant embedded in the tree is part of that shape.**

Consider two ways of writing the same filter:

```csharp
// A: the value is a captured variable → it becomes a PARAMETER
var status = OrderStatus.Open;
await db.Orders.Where(o => o.Status == status).ToListAsync();

// B: the value is a literal in the source → it becomes a CONSTANT in the tree
await db.Orders.Where(o => o.Status == OrderStatus.Open).ToListAsync();
```

Both are fine, because in B the constant is fixed at compile time — one shape, one cache entry, one plan. The pathological case is building predicates dynamically so that a *runtime value* ends up as a constant node:

```csharp
// BAD: a new expression tree, with a new constant, on every call
var param = Expression.Parameter(typeof(Order), "o");
var body  = Expression.Equal(
    Expression.Property(param, nameof(Order.CustomerId)),
    Expression.Constant(customerId));            // ← runtime value as a constant
var predicate = Expression.Lambda<Func<Order, bool>>(body, param);
await db.Orders.Where(predicate).ToListAsync();
```

Now every distinct `customerId` produces a distinct cache key: EF recompiles the query on every call, the cache grows without bound, and the database's plan cache fills with near-identical plans. CPU climbs in the application *and* the database, and nothing in the SQL looks wrong. The fix is `Expression.Constant` → a captured closure or `Expression.Parameter`-based parameterization, or better, avoid hand-built trees and use a specification/predicate-builder library that parameterizes correctly.

The diagnostic: watch the compiled-query cache hit rate. EF publishes it — `Microsoft.EntityFrameworkCore` metrics include a compiled-query-cache hit-rate counter, exposed through `EventCounters` and OpenTelemetry (Concept 46). **A cache hit rate below ~99% in steady state means something is generating fresh trees**, and that is a specific, checkable claim to make in a debugging round.

---

## Concept 14 — Parameterization: what becomes a parameter, and why the database cares

Module 12 established why this matters on the database side: a parameterized statement gets one cached plan, an inlined-literal statement gets a plan per distinct literal, and plan-cache bloat is a real production failure mode. EF's rules:

- **Captured variables become parameters.** `o.Status == status` where `status` is a local, field, or captured closure → `@status`.
- **Literals in source become constants**, inlined into the SQL.
- **Collections are the interesting case**, and EF 10 changed the default.

EF ≤ 8.0 inlined collection contents as SQL constants, which generated a different SQL string per collection and caused plan-cache misses and bloat. EF 8 changed to a single JSON-array parameter unpacked with `OPENJSON`, which fixed the plan-cache problem but deprived the planner of cardinality information. EF 10 introduces a new default where each value becomes its own scalar parameter, and pads the parameter list so that a collection of 8 values generates SQL with 10 parameters, reducing the number of distinct SQL strings.

```sql
-- EF 10 default for  ids.Contains(b.Id)  with 8 ids
WHERE [b].[Id] IN (@ids1, @ids2, ..., @ids8, @ids9, @ids10)
```

The padding is the clever part: without it, every distinct collection *length* produces a distinct SQL string, so you'd trade one cache problem for another.

EF exposes the choice because no default is right for every query. The strategy is configurable globally via `UseParameterizedCollectionMode` and per query via `EF.Constant`.

```csharp
// Global default
options.UseSqlServer(cs, o => o.UseParameterizedCollectionMode(ParameterTranslationMode.Parameter));

// Per query: inline these, because the set is small and stable and the planner benefits
var users = await db.Users.Where(u => EF.Constant(roles).Contains(u.Role)).ToListAsync();

// Per query: force a parameter, because this value is high-cardinality
var rows = await db.Events.Where(e => e.TenantId == EF.Parameter(tenantId)).ToListAsync();
```

Note the security interaction, which is a nice detail to know: EF 10 redacts inlined constants from logged SQL by default, replacing them with `?`, because inlined parameters previously leaked potentially sensitive values into logs even though parameter values were never logged. So `EF.Constant` no longer implies "this value is now in my log files."

And a SQL Server ceiling worth carrying: a command is limited to **2,100 parameters**. A `Contains` over a 5,000-element list cannot be one parameterized `IN` — you need `OPENJSON`/a table-valued parameter, a temp table, or chunking. Knowing the number matters; "it just fails at some point" doesn't.

---

## Concept 15 — Client evaluation, and why EF throws

Early EF Core (1.x/2.x) would silently evaluate untranslatable parts of a query on the client — pulling the table into memory and filtering there. It was the single worst design decision in the product's history, because it turned a compile-time impossibility into a runtime performance cliff that only appeared at scale.

Since EF Core 3.0, the rule is: **client evaluation happens only in the top-level projection.** Anything untranslatable anywhere else throws `InvalidOperationException: The LINQ expression ... could not be translated.`

```csharp
// THROWS: IsEligible() is a C# method; SQL has never heard of it
await db.Orders.Where(o => o.IsEligible()).ToListAsync();

// FINE: the projection runs client-side, after the rows arrive
await db.Orders.Select(o => new { o.Id, Label = Describe(o.Status) }).ToListAsync();
```

Treat the exception as a feature. When you hit it, you have four honest options, in order of preference:

1. **Rewrite the predicate in translatable terms.** `o.Status == OrderStatus.Open && o.Total > 0` instead of `o.IsEligible()`.
2. **Express it once, as a reusable expression**, so it's shared without being a method call EF can't see:
   ```csharp
   public static readonly Expression<Func<Order, bool>> IsEligible =
       o => o.Status == OrderStatus.Open && o.Total > 0;
   // usage: db.Orders.Where(Order.IsEligible)
   ```
   This is the seed of the specification pattern (Concept 67).
3. **Map it into the database** — a computed column (`HasComputedColumnSql`), a database function mapped with `HasDbFunction`, or a mapped view.
4. **Split the query deliberately**: filter what you can in SQL, materialize a bounded result, then finish in memory — and know how bounded it is.

What you must *not* do is `.AsEnumerable()` in the middle of the query to make the error go away. That's the EF 2.x behaviour, reintroduced by hand, with the same consequences.

---

## Concept 16 — Projection is the highest-leverage habit in EF Core

If you take one performance practice from this module, take this one: **for reads, project into a DTO instead of materializing entities.**

```csharp
// Entities: SELECT *, tracked, every column, plus joins for every Include
var orders = await db.Orders
    .Include(o => o.Customer)
    .Where(o => o.Status == OrderStatus.Open)
    .ToListAsync();

// Projection: exactly the columns you need, no tracking, join only if needed
var orders = await db.Orders
    .Where(o => o.Status == OrderStatus.Open)
    .Select(o => new OrderSummary(o.Id, o.PlacedAtUtc, o.Total, o.Customer.Name))
    .ToListAsync();
```

Four wins, all of them structural rather than micro:

1. **Less data on the wire.** A wide entity with `nvarchar(max)` columns, a JSON blob, and a vector embedding costs orders of magnitude more than four scalars. EF 11 made this concrete for one case: `SqlVector<T>` columns are excluded from `SELECT` when materializing entities, since vectors are usually ingested and searched but not read back; a minimal benchmark showed almost 9× against a local database and around 22× against a remote Azure SQL database. The same principle applies to every wide column you didn't need — EF just can't guess for the rest.
2. **No tracking, automatically.** A projection to a non-entity type is not tracked. No identity map entry, no snapshot, no `DetectChanges` cost, no retention (Concepts 20, 26).
3. **Fewer joins.** `Select(o => new { o.Id, o.Customer.Name })` emits one join for one column. `Include(o => o.Customer)` emits a join for the whole customer row.
4. **A stable contract.** Your API response type stops being your persistence type. Adding a column to `Orders` no longer changes what your endpoint returns, and your OpenAPI document stops leaking schema.

**The one thing to watch:** projecting a *collection* navigation inside a projection still produces a join or a correlated subquery, and the cartesian rules of Concept 18 still apply. `Select(o => new { o.Id, Lines = o.Lines.Select(l => l.Sku).ToList() })` is fine and often exactly right; two such collections in one projection is the explosion again.

**When to materialize entities instead:** when you are going to *change* them. That's the whole rule. Reads project; writes load the aggregate. That sentence is also a one-line summary of CQRS at the persistence level (Concept 65).

---

## Concept 17 — Loading related data: eager, explicit, lazy

**Eager** — `Include` / `ThenInclude`. The related data comes back with the parent, in one query (or several, with `AsSplitQuery`). Predictable, explicit, and filterable since EF 5:

```csharp
var order = await db.Orders
    .Include(o => o.Lines.Where(l => !l.Cancelled).OrderBy(l => l.Position))
    .Include(o => o.Customer)
        .ThenInclude(c => c.PreferredAddress)
    .FirstOrDefaultAsync(o => o.Id == id, ct);
```

`AutoInclude` exists (`Navigation(...).AutoInclude()`) for navigations that are genuinely always needed; use it sparingly, because it makes the cost of a query invisible at the call site. `IgnoreAutoIncludes()` opts out.

**Explicit** — load on demand, deliberately:

```csharp
await db.Entry(order).Collection(o => o.Lines).LoadAsync(ct);
await db.Entry(order).Reference(o => o.Customer).LoadAsync(ct);

// Or query the navigation without materializing it:
var lineCount = await db.Entry(order).Collection(o => o.Lines).Query().CountAsync(ct);
```

This is the right tool when whether you need the data depends on a branch — and the `Query()` form is underused: it lets you filter, count, or page a navigation without loading it.

**Lazy** — `Microsoft.EntityFrameworkCore.Proxies` plus `UseLazyLoadingProxies()`, requiring `virtual` navigations; or the `ILazyLoader` injection form, which avoids proxies but puts a persistence concern in your constructor.

**The position to hold on lazy loading in a server application: don't.** Three reasons, and you should be able to give all three:

1. **It manufactures N+1 invisibly.** Touching `order.Customer.Name` in a loop is a query per iteration, and nothing at the call site says so (Concept 19).
2. **It's synchronous.** There is no async lazy load. Every proxy-triggered load is a blocking database call on whatever thread you're on — Module 15's sync-over-async, in the worst possible place.
3. **It breaks outside the context's lifetime.** Serialize a lazily-loading entity after the request scope is disposed and you get `ObjectDisposedException`, or worse, a serializer that walks the whole object graph and loads your database into the response.

If you inherit a codebase that uses it, the migration is mechanical: project reads into DTOs (Concept 16) and make writes load aggregates eagerly.

---

## Concept 18 — Cartesian explosion, and what `AsSplitQuery` actually trades

This is the EF problem that most looks like a database problem and isn't.

```csharp
var orders = await db.Orders
    .Include(o => o.Lines)        // 20 lines per order
    .Include(o => o.Payments)     // 3 payments per order
    .Where(o => o.CustomerId == id)
    .ToListAsync();
```

One query with two collection joins returns **order × lines × payments** rows: for 100 orders, 100 × 20 × 3 = **6,000 rows**, each repeating the full order row and the full customer columns, to produce 100 orders with 2,000 lines and 300 payments. The database does more work, the network carries multiples of the data, and EF's shaper deduplicates it all back down. Latency is bad; memory is worse.

EF warns about it (`MultipleCollectionIncludeWarning`), and offers `AsSplitQuery()`:

```csharp
var orders = await db.Orders
    .Include(o => o.Lines)
    .Include(o => o.Payments)
    .AsSplitQuery()
    .ToListAsync();
```

Now EF issues **three** queries — orders, then lines, then payments — and stitches them together in the shaper. No duplication. The trade you have made:

| | Single query | Split query |
|---|---|---|
| Round trips | 1 | 1 + one per collection |
| Rows transferred | product of collection sizes | sum of collection sizes |
| Consistency | One statement, one snapshot | **Multiple statements — rows can change between them** unless you wrap in a transaction or use a snapshot isolation level |
| Best when | one collection, or small collections | multiple or large collections |

That consistency row is the senior detail. Split queries are not free correctness-wise: the separate statements can see different states of the database. If the read must be consistent, run it in an explicit transaction with an appropriate isolation level (Module 12, Concepts 20–24).

EF has fixed real bugs in this area, which is worth knowing when you meet an older codebase: EF 10 made ordering consistent across split queries, where the subquery previously omitted the key column from its `ORDER BY` and could return incorrect data. And EF 11 prunes unnecessary joins to reference navigations in split queries — a common split-query scenario showed a 29% improvement.

You can set the default globally (`UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)`), and I'd generally advise against it: the right behaviour is per query, and a global default hides the decision. **The genuinely better answer for read-heavy endpoints is usually neither** — it's Concept 16, project only what you need, at which point there's nothing to explode.

---

## Concept 19 — The N+1 problem, in its three EF Core costumes

N+1 is one query to get N parents, then one query per parent for its children. In EF Core it arrives in three disguises:

**1. Lazy loading.** The classic. `foreach (var o in orders) Console.Write(o.Customer.Name);` is 1 + N queries, and the source code contains no query at all. This is the strongest argument against proxies (Concept 17).

**2. A query inside a loop.** Explicit, but easy to write:

```csharp
foreach (var id in ids)                                        // N round trips
    result.Add(await db.Orders.FirstAsync(o => o.Id == id));
```
Fix: one query. `await db.Orders.Where(o => ids.Contains(o.Id)).ToListAsync();` — and now Concept 14's parameterization rules and the 2,100-parameter ceiling matter.

**3. Navigation access after materialization without an `Include`.** With lazy loading off, this doesn't secretly query — it returns `null` or an empty collection, which is arguably worse, because you get *wrong data* instead of slow data. This is the shape behind "the list renders but every customer name is blank."

**Detection, in order of usefulness:**

- **Log the SQL and count the statements per request.** The crudest and most reliable method. One request, one log scrape, count the `SELECT`s.
- **Distributed tracing.** With OpenTelemetry (Module 11, Module 18, Concept 70), an N+1 shows up as a sawtooth of identical short spans inside one request span. Once you've seen the shape you never miss it.
- **Assert on it in tests.** Count commands via a `DbCommandInterceptor` in an integration test and fail the test above a threshold. This is the only method that stops it coming back.
- **Analyzers and the `MultipleCollectionIncludeWarning`** catch the cartesian case, not this one.

**The fix hierarchy:** project (Concept 16) → `Include` (Concept 17) → explicit load in one batched query → split query if the includes are genuinely needed and large. Not: "turn on lazy loading and hope."

---

## Concept 20 — Tracking, no-tracking, and identity resolution

Three modes, and the middle one has a subtlety most candidates miss.

| Mode | Identity map | Snapshot | Duplicate instances for shared entities? | Use for |
|---|---|---|---|---|
| Tracking (default) | Yes | Yes | No — one instance per key | Anything you will modify |
| `AsNoTracking()` | No | No | **Yes** — one instance per row | Read-only, especially when projecting is impossible |
| `AsNoTrackingWithIdentityResolution()` | Yes (query-local) | No | No | Read-only reads over graphs with shared references |

The `AsNoTracking` subtlety: with no identity map, if 50 orders share one customer, you get **50 distinct `Customer` objects** — one per row. Usually harmless; occasionally a correctness surprise (reference equality fails, mutating one doesn't affect the others) and occasionally a memory surprise on wide shared entities. `AsNoTrackingWithIdentityResolution()` restores one-instance-per-key for the duration of the query at a small CPU cost.

You can set the default per context, which is a reasonable posture for a read-heavy service:

```csharp
options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
// and opt back in where you write:  db.Orders.AsTracking().FirstAsync(...)
```

Two clarifications worth having ready:

- **Projections to non-entity types are not tracked regardless.** `Select(o => new OrderDto(...))` never enters the change tracker. So "always use `AsNoTracking`" is advice for people who haven't adopted projection; if you project, it's usually redundant.
- **`AsNoTracking` is not primarily a speed optimization — it's a retention decision.** The per-entity CPU saving is small. The saving that matters is that the entity, its snapshot, and everything it references become collectible immediately instead of living until the context dies (Module 14). On a query that materializes 50,000 rows into a Gen 2-bound graph, that is the difference between a healthy service and an OOM.

---

## Concept 21 — Compiled queries and precompiled queries

Concept 13 established that EF caches compiled queries, keyed by tree shape. **Explicitly compiled queries** skip even the cache lookup:

```csharp
private static readonly Func<AppDbContext, Guid, CancellationToken, Task<Order?>> GetOrderById =
    EF.CompileAsyncQuery((AppDbContext db, Guid id, CancellationToken ct) =>
        db.Orders.Include(o => o.Lines).FirstOrDefault(o => o.Id == id));

// usage
var order = await GetOrderById(db, id, ct);
```

EF can automatically compile and cache queries based on a hashed representation of the query expressions; the explicit API bypasses computing the hash and the cache lookup, letting the application invoke an already-compiled delegate. The guidance on magnitude is honest about it: **the more complex the LINQ query — the more operators and the bigger the expression tree — the more gains can be expected from compiled queries.**

So the honest framing: this is a **micro-optimization for a hot path**, worth single-digit microseconds per call on a complex query. On a request that also does a 3 ms database round trip, it is noise. Reach for it when you have measured a genuinely hot query in a high-throughput service, not as a default style. Note the constraints too: a compiled query's parameters are fixed, so conditional query building (Concept 11) and compiled queries are mutually exclusive, and `Include`s must be baked in.

**Precompiled queries** are a different thing with a similar name, and much more interesting architecturally. Precompiled queries work by static analysis generating C# interceptors that contain the finished SQL and the materialization code, so the heavy work of translating LINQ to SQL no longer happens at startup. The mechanism is the same one that makes NativeAOT possible at all — but the useful half is available without it: `dotnet ef dbcontext optimize --precompile-queries` generates a compiled model and interceptors that remove query-compilation overhead from startup.

The AOT status, stated carefully because interviewers ask and candidates overclaim: **NativeAOT support and query precompilation remain highly experimental and are not yet suited for production use**; the current support should be viewed as infrastructure toward the final feature, publishing still reports trimming and AOT warnings, and Microsoft recommends against deploying EF NativeAOT applications in production. Because interceptors are invalidated by any source change, `dotnet ef dbcontext optimize` and `dotnet publish` belong in the CI/CD pipeline rather than the inner development loop. The known constraint that bites hardest in real code is **dynamic LINQ**: if a query's `DbSet` is reached through a generic wrapper, an extension method, or an abstract base class, the precompiler classifies it as dynamic and refuses it — which is to say, a well-abstracted repository layer is exactly the thing that defeats it today.

This connects back to Module 17's AOT section and Module 18, Concept 75 with a clean summary: **for a .NET web service today, AOT is viable for minimal APIs and gRPC, and EF Core is the usual blocker.** If AOT is a hard requirement, the pragmatic architecture is a seam at the data layer — EF behind an interface, with a raw-ADO.NET implementation for the AOT-published service — rather than fighting the precompiler.

---

## Concept 22 — Raw SQL, safely

EF has three entry points and the differences are exactly what a security-conscious interviewer probes.

```csharp
// 1. FromSql — interpolated string; EVERY hole becomes a DbParameter. Composable.
var orders = await db.Orders
    .FromSql($"SELECT * FROM Orders WHERE TenantId = {tenantId}")
    .Where(o => o.Status == OrderStatus.Open)      // composes in SQL, as a subquery
    .OrderBy(o => o.PlacedAtUtc)
    .ToListAsync(ct);

// 2. SqlQuery<T> — scalars and unmapped types, no entity required
var totals = await db.Database
    .SqlQuery<decimal>($"SELECT SUM(Total) FROM Orders WHERE TenantId = {tenantId}")
    .ToListAsync(ct);

// 3. ExecuteSql — non-query; returns rows affected. Bypasses the change tracker entirely.
await db.Database.ExecuteSqlAsync($"UPDATE Orders SET Archived = 1 WHERE PlacedAtUtc < {cutoff}", ct);
```

`FromSql` accepts a .NET `FormattableString`, and any hole is automatically sent as a separate SQL parameter, preventing SQL injection. `FromSqlRaw` is the escape hatch for genuinely dynamic SQL — a column name chosen at runtime, say — and it is entirely your responsibility to sanitize. EF 10 added an analyzer that warns when string concatenation is performed inside a raw SQL method invocation; if the fragment is trusted or sanitized, suppress the warning deliberately rather than by habit.

Four rules for using raw SQL well:

1. **Composability requires a composable shape.** `FromSql` can be composed over (EF wraps it in a subquery) only if the SQL is a single `SELECT` without a trailing `;`, and on SQL Server stored procedures can't be composed over at all — call `.AsEnumerable()`/`ToListAsync()` first, and know you've left the database.
2. **The result must match the entity's expected columns** for `FromSql` on an entity type. Missing columns fail at materialization, not at translation.
3. **Tracking still applies.** `FromSql` over an entity type returns tracked entities by default; add `AsNoTracking()` for reads.
4. **Global query filters are not a safety net here.** Verify what a raw query does with tenancy; this is precisely where multi-tenant isolation leaks (Concept 10).

The architectural point, which Module 12, Concept 60 stated and this module inherits: **raw SQL is not an admission of failure, it's the seam.** EF for aggregate-shaped reads and writes, SQL (directly or via Dapper) for reporting queries, set operations, and anything where you must control the plan. Concept 65 develops it.

---

## Concept 23 — Pagination: offset vs. keyset

```csharp
// Offset pagination — simple, and degrades with depth
var page = await db.Orders.OrderByDescending(o => o.PlacedAtUtc).ThenBy(o => o.Id)
    .Skip(pageIndex * 50).Take(50).ToListAsync(ct);
```

`Skip(n)` becomes `OFFSET n ROWS FETCH NEXT 50`. The database must still *produce and discard* those n rows, so page 1 is instant and page 2,000 is a scan. It also has a correctness flaw: if rows are inserted between requests, items shift across page boundaries and the user sees duplicates or misses.

```csharp
// Keyset (seek) pagination — constant cost, stable under concurrent writes
var page = await db.Orders
    .Where(o => o.PlacedAtUtc < lastSeenDate
            || (o.PlacedAtUtc == lastSeenDate && o.Id < lastSeenId))   // tuple comparison
    .OrderByDescending(o => o.PlacedAtUtc).ThenByDescending(o => o.Id)
    .Take(50).ToListAsync(ct);
```

The predicate becomes an index seek on `(PlacedAtUtc DESC, Id DESC)` and costs the same on page 2,000 as on page 1. The cost is interface: you get "next/previous," not "jump to page 137." For infinite scroll, feeds, and APIs, that's the right trade. For an admin grid that genuinely needs page numbers, offset is acceptable — bound the maximum page, and know why the last page is slow.

**Two details worth having:** you must include a **tiebreaker** column (the key) in the ordering, or rows with equal sort values shuffle between pages; and a total count is a second query (`CountAsync`) which is often the more expensive half of the request — consider caching it, approximating it, or not showing it.

---

## Concept 24 — Reading the SQL: the diagnostic toolkit

You cannot reason about any of the above without seeing what EF emitted. Four tools, from cheapest to most invasive.

**1. `ToQueryString()`** — no execution, no database, just the SQL:
```csharp
var sql = db.Orders.Where(o => o.Status == OrderStatus.Open).ToQueryString();
```
Use it in a unit test to assert that a query still translates the way you expect. That's a genuinely good practice: it turns "did my refactor break the SQL?" into a red test.

**2. `TagWith()`** — prepends a comment to the generated SQL:
```csharp
var q = db.Orders.TagWith("OrderSummary/GetOpenOrders").Where(...);
// -- OrderSummary/GetOpenOrders
// SELECT ...
```
This is how you connect a slow statement in the *database's* DMVs or Query Store back to a line of C#. On a codebase with 400 queries, this saves hours. `TagWithCallSite()` adds file and line automatically.

**3. Logging.** Wire EF's logs through `ILogger`, filter to `Microsoft.EntityFrameworkCore.Database.Command` at `Information`, and you get every statement with its elapsed time and parameters (redacted). In development only:
```csharp
options.EnableSensitiveDataLogging()     // parameter VALUES — never in production
       .EnableDetailedErrors();          // per-property materialization errors
```
The reason `EnableSensitiveDataLogging` is off by default is the same reason EF 10 started redacting inlined constants: SQL in logs is a PII exfiltration path.

**4. Interceptors** (Concept 35) for anything programmatic — counting commands in a test, failing a build on a query above a duration threshold, adding query hints.

And the tool that isn't EF's: **the database's own plan tooling.** Query Store on SQL Server/Azure SQL, `pg_stat_statements` and `EXPLAIN (ANALYZE, BUFFERS)` on PostgreSQL. EF tells you what it sent; only the database tells you what the plan did. A senior debugging answer moves between the two — "the SQL looks right, so I pulled the actual plan and found a key lookup because the index doesn't cover the projection."

---

# Part C — Change tracking and `SaveChanges`

---

## Concept 25 — The state manager: an identity map with snapshots

The change tracker is not magic and it is not a diff engine over the database. It is a concrete data structure hanging off your `DbContext`:

- An **identity map**: a dictionary from (entity type, primary key) → `EntityEntry`. This guarantees **one CLR instance per key per context**. Query the same order twice in one context and you get the same object back — the second query still hits the database, but the shaper returns the already-tracked instance rather than a new one (which also means *the second query's values are discarded by default* — a surprise worth knowing, controlled by `QueryTrackingBehavior` and the `ReloadAsync`/`GetDatabaseValues` APIs).
- For each tracked entity: its **state** (Concept 29), its **current values**, its **original values** (the snapshot taken at materialization or attach), and its **relationship fixups**.

Two consequences that drive everything else in this part:

**The identity map is a GC root.** Every tracked entity, its snapshot, and everything it references are reachable from the context for as long as the context lives. Module 14's retention analysis applies directly: a `DbContext` that lives for a request is fine; one that lives for the process is a leak with a respectable-looking name. This is why Concept 38's lifetime question matters more than any tuning knob in this module.

**Relationship fixup is automatic and one-way surprising.** When you materialize an `Order` and later materialize its `OrderLine`s into the same context, EF wires `order.Lines` and `line.Order` for you — even if you never wrote an `Include`. This is genuinely helpful and it is also how people convince themselves lazy loading is on when it isn't: the data arrived because a *previous* query in the same context pulled it in.

```csharp
var entry = db.Entry(order);
entry.State;                                     // EntityState.Modified
entry.Property(o => o.Total).IsModified;         // true
entry.Property(o => o.Total).OriginalValue;      // what the DB had
entry.Property(o => o.Total).CurrentValue;       // what you set
entry.Collection(o => o.Lines).IsLoaded;         // did we load it?
db.ChangeTracker.Entries<Order>();               // everything tracked, by type
db.ChangeTracker.DebugView.LongView;             // ← the single best debugging tool here
```

`ChangeTracker.DebugView` is worth committing to memory. It prints every tracked entity, its state, its key, which properties are modified, and its relationships. When someone asks "why did `SaveChanges` issue that `UPDATE`?", this answers it in one line of code.

---

## Concept 26 — `DetectChanges`: the scan you're paying for

EF's default change tracking is **snapshot-based**. It has no hooks into your POCOs, so when you ask it what changed, it walks every tracked entity and compares every mapped property against the snapshot it took when the entity was loaded.

The cost is `O(tracked entities × mapped properties)`, plus collection comparisons for converted values. With 30 entities it's invisible. With 30,000 it is tens of milliseconds — **each time it runs**, and it runs more often than people expect. `DetectChanges` is invoked automatically by `SaveChanges`, by `ChangeTracker.Entries()`, by `Local` access, and by several `DbSet` APIs.

The pathological shape is a loop:

```csharp
foreach (var row in rows)             // 10,000 iterations
{
    db.Orders.Add(Map(row));          // each Add triggers DetectChanges over everything added so far
}
await db.SaveChangesAsync();          // 1 + 2 + 3 + ... + 10,000 property comparisons
```

That's quadratic, and it's the real explanation behind most "EF is slow at inserts" complaints. Three fixes, in order:

1. **`AddRange`** instead of `Add` in a loop — one detect pass.
2. **`ChangeTracker.AutoDetectChangesEnabled = false`** around a bulk section, calling `ChangeTracker.DetectChanges()` yourself once before `SaveChanges`. Restore it in a `finally`. This is a scalpel, not a default — turning it off globally means your ordinary `SaveChanges` calls silently stop noticing modifications.
3. **Don't put bulk work in the change tracker at all** (Concept 49). If you're inserting 100,000 rows, the change tracker is the wrong tool; use `SqlBulkCopy` or a bulk library.

EF 11 adds a targeted escape hatch: `ChangeTracker.GetEntriesForState()` returns tracked entries in selected states **without** forcing a `DetectChanges` pass — useful in long-lived contexts or high-volume change-tracking code where you already know which states you care about.

```csharp
var modified = context.ChangeTracker.GetEntriesForState(
    added: false, modified: true, deleted: false, unchanged: false);
```

The mental model to carry into an interview: **`DetectChanges` is the price of POCOs.** EF gives you clean domain classes with no base type and no interface, and the cost is a scan. The alternative is Concept 27.

---

## Concept 27 — Notification entities and change-tracking proxies

If your entities implement `INotifyPropertyChanged` (and `INotifyPropertyChanging` for original values), EF can subscribe and learn about changes as they happen — no snapshot, no scan:

```csharp
modelBuilder.Entity<Order>()
    .HasChangeTrackingStrategy(ChangeTrackingStrategy.ChangingAndChangedNotifications);
```

Or let the proxies package generate it: `UseChangeTrackingProxies()`, requiring `virtual` properties and a public/protected constructor.

The trade is explicit: **you eliminate the `DetectChanges` cost and you give up POCO purity.** Every property setter now raises an event; your domain types carry a persistence-driven interface; proxies mean your entities aren't the types you wrote, which breaks `GetType()` comparisons, some serializers, and pattern matching on sealed hierarchies.

**The honest recommendation:** this is a specialized tool for long-lived contexts with large graphs — a desktop or Blazor Server application holding thousands of entities in an editor, say. For a request-scoped web context tracking 20 entities, it's pure downside. Know it exists, know *why* it exists (the `DetectChanges` scan), and be able to say when it would be right. That sequence is the signal; reciting the API is not.

---

## Concept 28 — Disconnected graphs: `Attach`, `Update`, and `TrackGraph`

In a web application, the entity you're saving usually wasn't loaded by the context that saves it. It arrived as JSON. This is the **disconnected graph** problem, and it's where a lot of subtly wrong code lives.

```csharp
db.Attach(order);    // whole graph → Unchanged (except entities with no key set → Added)
db.Update(order);    // whole graph → Modified  (except entities with no key set → Added)
db.Remove(order);    // → Deleted, cascading to loaded dependents per DeleteBehavior
```

`Attach` and `Update` use **key-value heuristics**: an entity whose key is the CLR default (0, `null`, `Guid.Empty`) is assumed new and marked `Added`; anything with a key set is assumed existing. That's a reasonable default and it is why `Update()` on a graph is dangerous: it marks **every property of every entity in the graph** as modified, so `SaveChanges` writes every column of every row — including ones the user never touched, including ones another request just changed, and including your concurrency token if you're not careful.

`TrackGraph` gives you per-node control:

```csharp
db.ChangeTracker.TrackGraph(order, node =>
{
    var entry = node.Entry;
    entry.State = entry.IsKeySet ? EntityState.Modified : EntityState.Added;
    // or: skip audit columns, honour a client-supplied version, refuse unknown types
});
```

**The pattern I'd defend in a design review** is neither of these. It's **load-then-mutate**:

```csharp
var order = await db.Orders
    .Include(o => o.Lines)
    .FirstOrDefaultAsync(o => o.Id == dto.Id, ct)
        ?? throw new NotFoundException();

order.Apply(dto);                 // a domain method that enforces invariants
await db.SaveChangesAsync(ct);    // EF writes only what actually changed
```

It costs one extra round trip and buys four things: only changed columns are written, domain invariants get enforced by the aggregate rather than by the mapper, concurrency tokens work naturally, and you cannot be tricked into writing a field the client shouldn't control (mass assignment). For anything with business rules, that round trip is the cheapest correctness you will ever buy. Module 22 makes this a first-class rule.

---

## Concept 29 — The entity state machine

| State | Meaning | What `SaveChanges` emits | How you get here |
|---|---|---|---|
| `Detached` | Not tracked | Nothing | `new`, a projection, `AsNoTracking`, after `SaveChanges` on a deleted entity |
| `Unchanged` | Tracked, matches snapshot | Nothing | Materialized by a tracking query; `Attach` |
| `Added` | New | `INSERT` | `Add`/`AddRange`; `Attach`/`Update` on a keyless-value entity |
| `Modified` | At least one property differs | `UPDATE` of the modified columns | Mutating a tracked entity; `Update` |
| `Deleted` | Marked for removal | `DELETE` | `Remove`/`RemoveRange`; cascade from a deleted principal |

Three details that get asked:

- **Modified is per property, not per entity.** EF emits `UPDATE Orders SET Total = @p0 WHERE Id = @p1 AND RowVersion = @p2` — only the changed columns. This is why `Update()` on a whole graph is wasteful, and why partial updates come free from load-then-mutate.
- **After a successful `SaveChanges`, `AcceptAllChanges` runs**: `Added` and `Modified` become `Unchanged` with a fresh snapshot, and `Deleted` becomes `Detached`. The context is reusable; whether it *should* be reused is Concept 37.
- **You can set state directly** (`db.Entry(x).State = EntityState.Modified`) and occasionally should — updating a single known field without loading the row:
  ```csharp
  var stub = new Order { Id = id, Status = OrderStatus.Cancelled };
  db.Attach(stub).Property(o => o.Status).IsModified = true;
  await db.SaveChangesAsync(ct);          // UPDATE Orders SET Status = @p0 WHERE Id = @p1
  ```
  This is the hand-rolled version of what `ExecuteUpdate` now does better (Concept 34) — but it's a useful demonstration that you understand the state machine rather than the API surface.

---

## Concept 30 — What `SaveChanges` actually does

Walk this sequence in an interview and you've demonstrated more than most candidates cover in the whole topic:

1. **`DetectChanges`** (unless disabled) — the snapshot scan of Concept 26.
2. **Cascade rules applied** — deleted principals propagate `Deleted` or `SetNull` to *loaded* dependents per `DeleteBehavior` (Concept 8); severed required relationships mark orphans deleted.
3. **Value generation on save** — `ValueGeneratedOnAdd`/`OnUpdate` properties, temporary key values assigned for `Added` entities so relationships can be wired before the database issues real keys.
4. **Topological sort.** EF builds a dependency graph from the foreign keys and orders the commands so principals are inserted before dependents and dependents deleted before principals. **A cycle that cannot be broken throws**, which is the real explanation of `Unable to save changes because a circular dependency was detected` — and the reason a self-referencing required FK or a mutual dependency needs an explicit two-phase save.
5. **Batching.** Commands are grouped into batches per Concept 31.
6. **Execution inside a transaction.** If no transaction is in progress, EF starts one for the duration of `SaveChanges` and commits at the end — so a single `SaveChanges` is **atomic** across every entity it touches (Concept 32).
7. **Propagate generated values back.** Identity keys, computed columns, and `rowversion` values come back via `OUTPUT`/`RETURNING` and are written into your objects.
8. **`AcceptAllChanges`** — states reset, snapshots refreshed.

Two things worth naming because they're common misconceptions:

- **`SaveChanges` does not validate.** EF Core dropped EF6's entity validation. `[Required]` and `[MaxLength]` become *schema*, so violating them produces a database error, not a friendly validation message. Validation belongs at the edge (Module 18, Concept 57) and invariants belong in the domain (Module 22).
- **The return value is the number of state entries written**, not rows affected in the SQL sense. Useful as an assertion in tests; not useful as a business fact.

---

## Concept 31 — Batching

EF Core minimizes round trips by automatically batching updates into a single round trip: an entity loaded and changed plus two entities added become two `INSERT`s and one `UPDATE` sent together rather than one at a time.

The thresholds are provider-tuned, and the numbers are worth knowing precisely because they come up:

- **SQL Server: `MinBatchSize` 4, `MaxBatchSize` 42.** Performance analysis showed batching to be generally less efficient for SQL Server below 4 statements, and the benefits degrade after around 40, so EF executes up to 42 statements in a batch and sends the rest in separate round trips.
- **PostgreSQL (Npgsql): 1,000** by default.

```csharp
options.UseSqlServer(cs, o => o.MinBatchSize(1).MaxBatchSize(100));
```

Raise it only with a benchmark, and know the ceiling: **SQL Server allows 2,100 parameters per command** (PostgreSQL 65,535), so a large batch of wide rows hits the parameter limit before it hits your batch size.

Mechanically, EF 7+ uses `INSERT ... OUTPUT` on SQL Server (and `INSERT ... RETURNING` on PostgreSQL) to insert many rows and read generated keys back in the same statement — replacing the older `MERGE`-with-table-variable technique, which had triggers-and-OUTPUT complications. That history matters in one practical case: **if a table has triggers**, `OUTPUT` semantics change and EF needs to be told (`HasTrigger`), otherwise you get a runtime error on save. It's a favourite "have you actually run this in production" detail.

---

## Concept 32 — Transactions: implicit, explicit, ambient

**Implicit.** Every `SaveChanges` runs in a transaction. If 50 entities change, all 50 succeed or none do. No code required, and this covers the majority of well-designed aggregate writes.

**Explicit.** When one unit of work spans multiple `SaveChanges` calls, or mixes EF with raw SQL:

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
try
{
    await db.SaveChangesAsync(ct);
    await db.Database.ExecuteSqlAsync($"INSERT INTO Outbox ...", ct);
    await tx.CommitAsync(ct);
}
catch { await tx.RollbackAsync(ct); throw; }
```

`BeginTransactionAsync(IsolationLevel)` sets the level; Module 12, Concepts 20–24 is the reference for choosing it. EF does not manage isolation for you and defaults to the provider's default (`READ COMMITTED` on SQL Server, unless the database has RCSI on).

**Ambient (`TransactionScope`).** Works, and carries the trap Module 12 flagged: **the default constructor is `TransactionScopeOption.Required` with isolation `Serializable` and no async flow.** You must write `new TransactionScope(TransactionScopeAsyncFlowOption.Enabled)` or every `await` inside it leaves the scope. And if two different connections enlist, you've silently promoted to a distributed transaction — which on modern platforms may simply not be available. Prefer `BeginTransaction`.

**Savepoints.** EF 7+ creates a savepoint before `SaveChanges` when a transaction is already in progress, so a failed `SaveChanges` rolls back to the savepoint rather than killing your whole transaction. You can manage them yourself (`CreateSavepointAsync`, `RollbackToSavepointAsync`). Some providers and configurations don't support them — notably when retries are involved, see Concept 43.

**Cosmos is different, and EF 11 changed it.** The Cosmos provider now uses transactional batches by default, grouping operations by container and partition and executing batches sequentially, controlled by `AutoTransactionBehavior` (`Auto`, `Never`, `Always`). That is best-effort atomicity **within a partition**, not across partitions — the distributed-transaction limits from Module 12 are unchanged; only the ergonomics improved.

**The architectural sentence to have ready:** *"A transaction is a consistency boundary, and in a well-designed model it should coincide with an aggregate. If I need a transaction spanning two aggregates, that's a signal I've drawn the boundary wrong — or that I need a saga or an outbox instead of a bigger transaction."* That's Module 11 and Module 12 arriving in EF.

---

## Concept 33 — Optimistic concurrency, properly resolved

Module 12, Concept 26 introduced this; here's the EF mechanics in full.

```csharp
// SQL Server: a database-maintained 8-byte token
builder.Property<byte[]>("RowVersion").IsRowVersion();

// PostgreSQL: the system column
builder.UseXminAsConcurrencyToken();

// Portable: any business column
builder.Property(o => o.LastModifiedUtc).IsConcurrencyToken();
```

EF then emits `UPDATE Orders SET ... WHERE Id = @id AND RowVersion = @original`. If zero rows are affected, it throws `DbUpdateConcurrencyException`.

The part that separates candidates: **catching it is not resolving it.** A retry loop that blindly reloads and re-applies has just implemented last-write-wins with extra steps. The three legitimate resolutions:

```csharp
catch (DbUpdateConcurrencyException ex)
{
    foreach (var entry in ex.Entries)
    {
        var databaseValues = await entry.GetDatabaseValuesAsync(ct);

        if (databaseValues is null)
            throw new ConflictException("The record was deleted by another user.");

        // (a) Store wins: discard our changes
        // entry.CurrentValues.SetValues(databaseValues);

        // (b) Client wins: keep our values, refresh the token and retry
        // entry.OriginalValues.SetValues(databaseValues);

        // (c) Merge: surface both to the user, or merge field-by-field in the domain
    }
}
```

**The judgement to voice:** which resolution is correct is a *business* decision, not a technical one. For a "last edited wins" content field, client-wins is fine. For an inventory count or a balance, neither blind option is acceptable — the correct answer is usually to make the operation **commutative** at the database level (`SET Quantity = Quantity - @n WHERE Quantity >= @n`, via `ExecuteUpdate`), so there is no conflict to resolve. That's Module 9's CRDT intuition applied to a single row.

One more thing worth saying: optimistic concurrency detects *concurrent writes*, not *stale reads*. If a user's decision was based on data that's now old, the token tells you the row changed; it does not tell you whether the change invalidates the decision. That's a domain question.

---

## Concept 34 — `ExecuteUpdate` / `ExecuteDelete` and the change-tracker divergence

```csharp
// The bad old way: N round trips of data in, N updates out
foreach (var e in db.Employees) e.Salary += 1000;
await db.SaveChangesAsync(ct);

// The right way: one statement, no materialization, no tracking
await db.Employees
    .Where(e => e.DepartmentId == deptId)
    .ExecuteUpdateAsync(s => s.SetProperty(e => e.Salary, e => e.Salary + 1000), ct);

await db.Sessions.Where(s => s.ExpiresAtUtc < now).ExecuteDeleteAsync(ct);
```

The load-and-modify version does a round trip to load every employee (bringing every column, not just salary), builds a snapshot for each, runs `DetectChanges` over all of them, and emits individual updates. `ExecuteUpdate` emits one `UPDATE`.

EF 10 made this much more usable. `ExecuteUpdateAsync` now accepts a regular, non-expression lambda, so conditional setters no longer require hand-built expression trees:

```csharp
await db.Blogs.ExecuteUpdateAsync(s =>
{
    s.SetProperty(b => b.Views, 8);
    if (nameChanged) s.SetProperty(b => b.Name, "foo");
}, ct);
```

And it works into JSON columns: EF 10 allows referencing JSON columns and properties within them in `ExecuteUpdateAsync` — though only when the type is mapped as a **complex type**, not as an owned entity (one more reason for Concept 5's recommendation).

**The four caveats, all of which matter:**

1. **They execute immediately.** They are not deferred to `SaveChanges`. If you're inside an explicit transaction they participate in it; if not, they're their own transaction.
2. **They bypass the change tracker entirely.** Entities already tracked in your context are now **stale** — EF has no idea the database changed. Mixing `ExecuteUpdate` with tracked entities in the same context is a correctness hazard; use a fresh context, or do it before you load anything, or `Reload()` afterwards.
3. **They bypass everything built on the tracker**: no concurrency-token check, no cascade behaviour, no `SaveChanges` interceptors, no domain events, no audit stamps from your interceptor. If your outbox is written by a `SaveChanges` interceptor (Concept 35), an `ExecuteDelete` will not produce an event. This is the single biggest design trap in this concept.
4. **They don't compose with `Include`**, and their interaction with global query filters deserves verification rather than assumption.

**So the rule:** `ExecuteUpdate`/`ExecuteDelete` for *maintenance and set-based operations that are not domain events* — expiring sessions, archiving old rows, backfills, counters. For domain state changes that other parts of the system must learn about, go through the aggregate and `SaveChanges`.

---

## Concept 35 — Interceptors: the real extensibility model

EF's interception points, and what each is actually for:

| Interface | Fires on | Canonical use |
|---|---|---|
| `IDbCommandInterceptor` | Before/after every command | Query logging, command counting in tests, query hints, slow-query alerts |
| `IDbConnectionInterceptor` | Connection open/close | Per-tenant connection strings, `SESSION_CONTEXT` for Row-Level Security, access tokens |
| `IDbTransactionInterceptor` | Begin/commit/rollback | Transaction-scoped diagnostics |
| `ISaveChangesInterceptor` | Around `SaveChanges` | **Audit stamps, soft delete, the transactional outbox, domain-event dispatch** |
| `IMaterializationInterceptor` | Entity creation/initialization | Injecting services into entities, post-load invariants |
| `IQueryExpressionInterceptor` | The query tree, before translation | Global query rewriting |

The one that matters architecturally is `ISaveChangesInterceptor`, because it's where you get the **transactional outbox** (Module 11) without asking every developer to remember it:

```csharp
public sealed class OutboxInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData e, InterceptionResult<int> result, CancellationToken ct = default)
    {
        var db = e.Context!;

        // 1. Audit stamps on shadow properties
        foreach (var entry in db.ChangeTracker.Entries()
                     .Where(x => x.State is EntityState.Added or EntityState.Modified))
        {
            entry.Property("LastModifiedUtc").CurrentValue = _clock.UtcNow;
        }

        // 2. Drain domain events into outbox rows — in the SAME transaction
        var events = db.ChangeTracker.Entries<IHasDomainEvents>()
            .SelectMany(x => x.Entity.DrainEvents())
            .Select(OutboxMessage.From)
            .ToList();

        db.Set<OutboxMessage>().AddRange(events);
        return base.SavingChangesAsync(e, result, ct);
    }
}
```

Registered with `options.AddInterceptors(...)`. Two things this buys: **the dual-write problem is gone** — the state change and the outbox row commit in one transaction, exactly as Module 11 specified — and it is *impossible to forget*, because no developer has to call it.

Two cautions. Interceptors run for **every** `SaveChanges` in the process, including migrations and seeding, so guard for that. And note what the previous concept said: `ExecuteUpdate`/`ExecuteDelete` do not pass through here, so anything you rely on this interceptor for must not be reachable via a bulk path.

---

## Concept 36 — Domain events: before, after, or outbox

Three placements, each correct in different circumstances. This is a real design question and interviewers like it because there's no single right answer.

**A. Dispatch before `SaveChanges`, in-process.** Handlers run inside your unit of work and can add to the same `SaveChanges`. Good for keeping a derived field consistent. Bad for anything with a side effect outside the database — you'll send an email for a transaction that then rolls back.

**B. Dispatch after `SaveChanges`, in-process.** The state is committed, so handlers see reality. But a handler failure now leaves you inconsistent — the order is placed and the confirmation was never queued — and there is no retry.

**C. Write to an outbox inside the transaction; dispatch asynchronously.** (Concept 35.) The state change and the intent to publish commit atomically. A separate process (a `BackgroundService`, or a change-data-capture pipeline) reads the outbox and publishes with at-least-once semantics and retries. Consumers must be idempotent, which Module 11 already established as the price of admission.

**The answer to give:** *"In-process dispatch inside the transaction for events that only touch my own database and must be consistent with the write. The outbox for anything that leaves the process — because otherwise I've reintroduced the dual-write problem, and a failure between the commit and the publish is silent data loss."* Then name the cost: the outbox adds a table, a background worker, latency, and a poison-message policy, and it earns that cost only when the integration actually matters.

---

# Part D — `DbContext` lifetime, pooling, and concurrency

---

## Concept 37 — `DbContext` is a unit of work, and that decides its lifetime

`DbContext` is two patterns in one object: a **unit of work** (a set of changes committed together) and an **identity map** (one instance per key). Every lifetime question follows from that, and answering it from the pattern rather than from a DI default is the seniority marker.

- It is **cheap to create**: constructing and disposing one performs no database operation. Concept 39 qualifies "cheap."
- It is **not thread-safe** (Concept 41).
- It **accumulates state** — every tracked entity stays until it dies (Concept 25).
- It **holds no connection between operations** (Concept 42).

So the rule is: **a `DbContext` should live exactly as long as one unit of work.** In a web request that's usually the request, because a request usually *is* one unit of work. But "scoped" is a consequence, not a principle — which is why the correct answer to "how long should a `DbContext` live?" is *"as long as the consistency boundary it serves,"* not *"scoped."*

The corollaries:

- A context that lives for the process is a bug: unbounded identity map, `DetectChanges` degrading quadratically, and every entity ever loaded pinned in memory.
- A context shared across parallel work is a bug (Concept 41).
- A single context that serves five unrelated operations in one request is *usually* fine and occasionally wrong — if operation 3 fails, operations 1 and 2 may already be tracked as modified and will be written by a later `SaveChanges` you didn't intend. When operations are genuinely independent, give each its own context via `IDbContextFactory`.

---

## Concept 38 — Scoped registration, and the three ways it goes wrong

```csharp
builder.Services.AddDbContext<AppDbContext>(o => o.UseSqlServer(cs));   // Scoped by default
```

Module 18's Part E is the full reference; here are the three EF-specific failures.

**1. Captive dependency.** A singleton that injects `AppDbContext` captures one context for the process. Symptoms: stale reads, `ObjectDisposedException`, concurrency exceptions under load, and unbounded memory growth. Prevented by `ValidateScopes`/`ValidateOnBuild` (Module 18, Concept 42), which turns it into a startup failure.

**2. A background service resolving a context from the root provider.** `BackgroundService` is a singleton. It must create its own scope per unit of work:

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await ProcessBatchAsync(db, ct);          // one scope = one unit of work
        await Task.Delay(_interval, ct);
    }
}
```
A scope per *loop iteration*, not per service. One scope for the lifetime of the worker is the long-lived-context bug wearing a scope.

**3. Capturing the request scope in a closure.** `_ = Task.Run(() => DoWork(db));` — the request completes, the scope disposes, the context is disposed, and the background task throws `ObjectDisposedException` at a random time under load. Module 18, Concept 46 covers the shape; the EF-specific version is by far the most common instance of it in .NET codebases.

---

## Concept 39 — Context pooling: what gets reset, and what doesn't

A `DbContext` is generally a lightweight object: creating and disposing one doesn't involve a database operation, and most applications can do so with no noticeable performance impact. However, each context instance sets up various internal services and objects, and the overhead of doing so continuously can be significant in high-performance scenarios. Pooling lets you pay context setup costs once at program start rather than continuously: on dispose, EF resets the instance's state and stores it in an internal pool; a new request gets the pooled instance instead of a fresh one.

```csharp
builder.Services.AddDbContextPool<AppDbContext>(o => o.UseSqlServer(cs), poolSize: 1024);
```

The `poolSize` parameter sets the maximum number of instances retained (defaulting to **1024**); once exceeded, new instances are not cached and EF falls back to non-pooling behaviour. (The default used to be 128, and the ASP.NET team's own benchmarks measured about a **5.8%** throughput gain from raising it — a useful calibration for how much this is worth: real, and small.)

**The rule that produces bugs:** EF only resets the state *it knows about*. Avoid context pooling if you maintain your own state — for example, private fields — in your derived `DbContext` class that should not be shared across requests.

That is precisely the multi-tenant collision:

```csharp
public sealed class AppDbContext : DbContext
{
    public Guid TenantId { get; private set; }     // ← survives into the next request under pooling
}
```

With pooling, tenant B's request can get a context still carrying tenant A's `TenantId` — and since your global query filter reads that field (Concept 10), you have built a cross-tenant data leak out of a performance optimization. The supported fix is the pooling-aware factory hook, `IDbContextFactory<T>` with `PooledDbContextFactory`, or `AddDbContextPool` with the overload that lets you reset per-lease state via `IResettableService` / the `DbContext`'s own reset hook. Verify the mechanism for your EF version; the *principle* — **pooling resets EF state, not yours** — is what you must be able to state.

**When pooling is worth it:** high-RPS services where context construction shows up in a profile. Note the honest caveat in the guidance itself: DI management of DbContext pooling incurs a slight overhead, so it isn't unconditionally faster. Measure (Module 17), don't assume.

---

## Concept 40 — `IDbContextFactory` and the non-request-scoped world

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(o => o.UseSqlServer(cs));
// or pooled:
builder.Services.AddPooledDbContextFactory<AppDbContext>(o => o.UseSqlServer(cs));
```

```csharp
await using var db = await _factory.CreateDbContextAsync(ct);
```

Four situations where the factory is the correct answer and scoped registration is not:

1. **Blazor Server.** A circuit is long-lived — minutes or hours — so a scoped context would be a long-lived context with all of Concept 37's problems, and concurrent UI events would hit Concept 41. One context per operation, created by the factory.
2. **Background services and hosted workers**, where you may prefer a factory to scope juggling.
3. **Parallel work.** Three independent queries in parallel need three contexts:
   ```csharp
   var (a, b, c) = await (LoadA(), LoadB(), LoadC());   // each creates its own context
   async Task<X> LoadA() { await using var db = await _factory.CreateDbContextAsync(ct); /* ... */ }
   ```
   With one shared context this throws the concurrency exception of Concept 41 — reliably in tests, intermittently in production, which is worse.
4. **Anything with no ambient scope** — a console tool, a Function, a test fixture.

You can register both: `AddDbContext` for request-scoped work *and* `AddDbContextFactory` for the rest. EF supports this explicitly, and it's a clean composition-root decision (Module 18, Concept 72).

---

## Concept 41 — Thread safety: one operation at a time

Using the same `DbContext` instance concurrently from different threads isn't supported. EF has a safety feature which detects this programming bug in many cases (but not all) and immediately throws an informative exception — `A second operation was started on this context instance before a previous operation completed.`

The two causes, in descending frequency:

1. **A missing `await`.** `var t1 = db.Orders.ToListAsync(); var t2 = db.Customers.ToListAsync(); await Task.WhenAll(t1, t2);` — two operations, one context. This is the single most common way the exception appears, and Module 15's rules about fire-and-forget `Task`s are the underlying lesson.
2. **Genuine parallelism** over a shared context (Concept 40).

The safety check has a cost, and EF lets you disable it: `EnableThreadSafetyChecks(false)` — with the guidance's own warning attached, that you should only disable thread safety checks after thoroughly testing that your application doesn't contain such concurrency bugs. Treat it as a last-percent optimization for a service you've already proven correct, not a way to silence an exception. Silencing it doesn't make the operation safe; it makes the corruption quiet.

---

## Concept 42 — Connections vs. contexts: two pools, orthogonal

This confusion is so common it's worth being explicit. Connection pooling is completely orthogonal to EF's `DbContext` pooling: the low-level database driver pools **database connections** to avoid the overhead of opening and closing them, while EF pools **context instances** to avoid context allocation and initialization overhead. Regardless of whether a context instance is pooled, EF generally opens connections just before each operation and closes them right afterwards, returning them to the pool, to avoid keeping connections out of the pool longer than necessary.

Three consequences:

- **A `DbContext` does not hold a connection between operations**, so a request-scoped context that spends 200 ms rendering a view is not holding a connection for 200 ms. Contexts are not the thing that exhausts your connection pool.
- **What does exhaust it**: an explicit transaction held open across slow work (the connection is pinned for the transaction's duration), a missing `await` leaving a reader open, and — most often — too many concurrent requests each doing a query, which is a capacity problem, not an EF problem. Size `Max Pool Size` against the database's own connection limit, and remember that a pool exhaustion presents as a *timeout on connect*, not a slow query.
- **Tuning the connection string matters more than most EF tuning.** `Max Pool Size`, `Connect Timeout`, `Application Name` (so the DBA can attribute load), and — on Azure SQL — `MultipleActiveResultSets` off unless you specifically need it.

---

## Concept 43 — Connection resiliency, and the execution-strategy trap

Cloud databases fail transiently by design — failovers, throttling, reconfiguration:

```csharp
options.UseSqlServer(cs, o => o.EnableRetryOnFailure(
    maxRetryCount: 5,
    maxRetryDelay: TimeSpan.FromSeconds(10),
    errorNumbersToAdd: null));
```

This installs an **execution strategy** that retries transient errors with exponential backoff. Azure SQL, Cosmos, and PostgreSQL providers all offer one.

**The trap, which Module 12 flagged and which is a genuine .NET interview differentiator:** with a **user-initiated transaction**, EF cannot retry automatically and throws. The reason is sound — EF can retry a single self-contained operation, but it cannot know how to replay an arbitrary block of your code that spans several operations inside one transaction. So you must make the retriable unit explicit:

```csharp
var strategy = db.Database.CreateExecutionStrategy();

await strategy.ExecuteAsync(async ct =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    await db.SaveChangesAsync(ct);
    await db.Database.ExecuteSqlAsync($"INSERT INTO Outbox ...", ct);
    await tx.CommitAsync(ct);
}, ct);
```

Now the *whole block* is the retry unit. Two follow-on requirements that separate a correct answer from a recited one:

- **The block must be idempotent**, because it can run more than once. If it reads state, mutates it, and commits, a retry after a *successful commit whose acknowledgement was lost* will apply it twice. This is the at-least-once problem from Module 11, in your data layer.
- **Reset the context between attempts** if the block mutated tracked state, or use a fresh context inside the strategy — otherwise the second attempt starts from a change tracker that already thinks the work is done.

EF 11 extends the strategy to schema operations: `EnsureCreated`, `EnsureCreatedAsync`, `Migrate` and `MigrateAsync` now use an execution strategy to retry on transient failures — which matters for containerized startup against a database that's still coming up.

Finally, the relationship to Polly (Module 25): **use the provider's execution strategy for database transient faults**, because it knows which error numbers are transient for that engine. Use Polly for everything else, and don't stack both on the same call — nested retries multiply into a thundering herd (Module 13).

---

## Concept 44 — Multiple contexts and bounded contexts

One database does not imply one `DbContext`. A `DbContext` is a *model*, and separate models over the same database are legitimate and often right:

- **Bounded contexts** (Module 22). `OrderingContext` and `BillingContext` map different subsets, with different aggregates, and possibly different names for the same table. This is the modular-monolith pattern (Module 21): separate models, separate schemas, one database, one deployment.
- **Read and write models.** A lean `DbContext` (or Dapper) for queries, a rich one for commands (Concept 65).
- **A tooling context** for admin/migration purposes with wider access.

Three mechanics to get right:

1. **Each context owns its own migrations**, and by default they share `__EFMigrationsHistory` — which is wrong. Give each its own: `o.UseSqlServer(cs, x => x.MigrationsHistoryTable("__EFMigrationsHistory", "ordering"))`, and put each context's tables in its own **schema**. Then every CLI command needs `--context`. EF 11 helps with the tedium: several `dotnet ef` commands accept `--context "*"` to target all contexts at once.
2. **A transaction cannot span two contexts** unless they share a connection. They can: `db2.Database.UseTransaction(db1.Database.CurrentTransaction.GetDbTransaction())` with both built on the same `DbConnection`. It works — and needing it is usually a sign the boundary is wrong. Prefer eventual consistency across contexts (outbox, Concept 36).
3. **Schema ownership becomes a governance rule**, not a convention: context A must not read context B's tables directly. Enforce it with database permissions if the organization is big enough to need it, and with review if it isn't.

---

# Part E — Performance engineering with EF Core

---

## Concept 45 — The cost model: where the milliseconds actually are

Module 17's discipline applies: know the cost model before you optimize. For a typical request doing one query:

| Stage | Typical cost | Notes |
|---|---|---|
| Query cache lookup | sub-microsecond | On a hit. On a miss, compilation is ~hundreds of µs to ms |
| Parameter binding + command setup | microseconds | |
| **Network round trip** | **0.3–2 ms same-region; 30–100+ ms cross-region** | Usually the dominant term |
| **Database execution** | **microseconds to seconds** | Entirely dependent on the plan and indexes |
| Reading rows + materialization | ~microseconds per row | Compiled shaper, not reflection |
| Change tracking on materialization | ~microseconds per entity | Snapshot allocation; zero if not tracking |
| `DetectChanges` on save | O(entities × properties) | The one that goes quadratic (Concept 26) |

**Read that table as an instruction:** the EF-attributable overhead is small and roughly constant per row; the round trip and the database plan are large and variable. So the optimizations that pay, in order:

1. **Fewer round trips** (N+1, batching, split vs. single).
2. **Less data** (projection, paging, excluding wide columns).
3. **Better plans** (indexes, parameterization, query shape).
4. **Less tracking** (retention, `DetectChanges`).
5. *Then* micro-optimizations (compiled queries, pooling, thread-safety checks).

Anyone who opens with #5 is optimizing the 2% — a good thing to notice out loud when reviewing someone's proposal, and a good thing not to do in an interview.

---

## Concept 46 — Diagnosis: a procedure, not a guess

The procedure I'd describe in a debugging round, in order:

**1. Count the statements per request.** Enable command logging at `Information` and scrape one request. If the count is surprising, you have an N+1 or an accidental loop, and nothing else matters until it's fixed.

**2. Look at the SQL for the slow one.** `ToQueryString()` in a test, or the log. Check for: `SELECT *` on a wide table, a join you didn't ask for, a `CAST` around an indexed column, an `IN` with inlined constants, a missing `TOP`/`OFFSET`, and `ORDER BY` columns you don't need.

**3. Get the actual plan from the database.** Query Store / `pg_stat_statements` / `EXPLAIN (ANALYZE, BUFFERS)`. `TagWith` is how you find your statement in there (Concept 24). Look for scans where you expected seeks, key lookups (a non-covering index), and a row-estimate that's wildly off actual (parameter sniffing or stale statistics).

**4. Check the metrics.** EF publishes counters through `EventCounters` and OpenTelemetry under `Microsoft.EntityFrameworkCore`: active `DbContext`s, queries executed per second, `SaveChanges` per second, **compiled-query cache hit rate**, optimistic-concurrency failures per second, and execution-strategy retries. The cache hit rate and the active-context count are the two that diagnose architectural problems rather than query problems.

**5. Trace it.** With OpenTelemetry (Module 11, Module 18), database spans nest inside the request span. The visual signatures are learnable: N+1 is a picket fence of identical short spans; a cartesian explosion is one long span with a large payload; pool exhaustion is a gap *before* the database span starts.

**6. Only now, benchmark in isolation** (Module 17) if you're changing the data-access strategy and need a number to defend it.

The thing that makes this senior is step 3. A candidate who stops at "EF generated this SQL" is describing the client; a candidate who pulls the plan is describing the system.

---

## Concept 47 — Indexing for EF-generated SQL

Module 12, Part 2 is the reference for indexing theory. What's EF-specific:

**Declare indexes in the model**, so they're part of the migration and travel with the code:

```csharp
builder.HasIndex(o => new { o.TenantId, o.Status, o.PlacedAtUtc })
       .HasDatabaseName("IX_Orders_Tenant_Status_Placed")
       .IncludeProperties(o => new { o.Total, o.CustomerId })     // covering
       .HasFilter("[IsDeleted] = 0");                             // filtered
```

Four EF-specific rules:

1. **Leading column = the thing every query filters on.** In a multi-tenant system that's `TenantId`, in every index, because the global query filter puts it in every `WHERE` clause (Concept 10). Module 12 said the same thing about clustered keys.
2. **`IncludeProperties` should mirror your projections.** If your list endpoint projects four columns, an index covering those four turns a key-lookup-per-row into a single index scan. This is the concrete payoff of Concept 16 — projection doesn't just reduce bytes, it makes covering indexes *possible*.
3. **Value converters can defeat indexes.** A `CAST` around the column means no seek. EF 11's no-op-`CAST` stripping removes one cause; the rest is on you to check in the plan.
4. **Filtered indexes pair with named query filters.** Soft delete plus `HasFilter("[IsDeleted] = 0")` keeps the index small and matches the predicate EF always emits.

And the one that's about the database, not EF: **parameter sniffing.** Because EF parameterizes (Concept 14), SQL Server caches a plan built for the *first* parameter values it saw. If your data is skewed — one tenant with 10 million rows and 500 with a thousand — the cached plan can be catastrophic for the other case. The remedies are database-side (`OPTIMIZE FOR UNKNOWN`, `RECOMPILE`, Query Store plan forcing) and EF gives you the hook: `EF.Constant` to inline a low-cardinality discriminator so the planner sees it, or a command interceptor that appends a hint to that one tagged query. Knowing that parameterization and parameter sniffing are two sides of one coin is a strong signal.

---

## Concept 48 — The query anti-pattern catalogue

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| `db.Orders.ToList().Where(...)` | Loads the table, filters in memory | Keep it `IQueryable` (Concept 11) |
| `Include` everything "to be safe" | Cartesian explosion, wide rows | Project (Concept 16) |
| `Count()` inside a loop | N round trips | One `GroupBy`, or a projection with `Count()` inside it |
| `Any()` vs `Count() > 0` | `COUNT(*)` scans; `EXISTS` short-circuits | `AnyAsync()` |
| `FirstOrDefault()` without `OrderBy` | Non-deterministic row | Always order, always include a tiebreaker |
| `Skip(n)` on deep pages | Produce-and-discard | Keyset pagination (Concept 23) |
| `Where(o => o.Name.ToLower() == x)` | Function on the column ⇒ no seek | Use a case-insensitive collation, or store a normalized column |
| String-built predicates with `Expression.Constant` | Cache-key explosion | Parameterize (Concept 13) |
| `.Include(...).AsNoTracking()` for a list page | Still fetches whole rows | Project |
| Mixing `ExecuteUpdate` with tracked entities | Stale tracker | Separate context or reload (Concept 34) |
| `AsEnumerable()` to dodge a translation error | Silent full load | Rewrite, map, or split deliberately (Concept 15) |
| Lazy loading in a web app | Invisible N+1, sync I/O, disposal bugs | Eager or explicit (Concept 17) |
| `SaveChanges` inside a loop | One transaction and round trip per iteration | One `SaveChanges` per unit of work |
| A `DbContext` per repository in one request | Several units of work; no shared transaction | One context per unit of work (Concept 66) |

---

## Concept 49 — Bulk operations: the escalation ladder

Four rungs, and the skill is knowing which one a problem sits on.

**1. Batched `SaveChanges`** (Concept 31). Up to a few thousand rows, with `AddRange`, `AutoDetectChangesEnabled = false`, and a sensible `MaxBatchSize`. Full EF semantics: generated keys, relationships, interceptors, concurrency tokens.

**2. `ExecuteUpdate` / `ExecuteDelete`** (Concept 34). Any number of rows, one statement — when the change is expressible as a set operation and you can accept losing tracker semantics.

**3. A bulk library.** `EFCore.BulkExtensions` and `Entity Framework Extensions` add `BulkInsert`/`BulkUpdate`/`BulkDelete`/`BulkMerge`. Note the licensing before you take a dependency: EFCore.BulkExtensions is dual-licensed — free for individuals and companies under $1M USD annual gross revenue, with a commercial licence from $1,000/year above that. That's exactly the shape of licensing shift Module 11 noted for MassTransit, and noticing it before it's a procurement problem is an architect behaviour.

**4. `SqlBulkCopy` / `COPY`.** The provider's native bulk path. Hundreds of thousands to millions of rows, minimal logging, no EF involvement at all. For an ETL load or an initial import, this is simply the right answer, and reaching for EF there is the error.

**The pattern for very large loads** is worth knowing as a shape: bulk-copy into a staging table, then one set-based `MERGE`/`INSERT ... ON CONFLICT` from staging into the target inside a transaction. You get the throughput of bulk copy and the atomicity of a single statement, and you never hold a million entities in a change tracker.

---

## Concept 50 — Streaming vs. buffering

`ToListAsync()` **buffers**: the whole result set is materialized into memory before you see any of it. For a large export, that's an allocation of the entire result — and, if it's big enough, the Large Object Heap and Gen 2 (Module 14).

```csharp
await foreach (var row in db.Orders
    .Where(o => o.PlacedAtUtc >= from)
    .Select(o => new ExportRow(o.Id, o.Total, o.PlacedAtUtc))
    .AsAsyncEnumerable()
    .WithCancellation(ct))
{
    await writer.WriteAsync(row, ct);        // constant memory
}
```

Streaming holds the connection and reader open for the duration, so the trade is **memory for connection-hold time**. Three rules:

- Stream for exports, reports, and anything unbounded; buffer for pages and small result sets.
- **Don't `SaveChanges` while streaming** the same context — you're using the connection.
- Combine with a projection and `AsNoTracking`, or the identity map grows exactly as fast as the buffer you were avoiding.

Use `AsAsyncEnumerable()`, not `ToAsyncEnumerable()` — EF 11 ships an analyzer (EF1004) for exactly this, because `ToAsyncEnumerable()` on an `IQueryable` wraps synchronous enumeration in an `IAsyncEnumerable<T>`, evaluating the EF query synchronously.

---

## Concept 51 — Caching around EF

Module 10's three questions govern here too: *what's the staleness budget, what invalidates it, and what happens on a miss storm?*

**Application-level caching (recommended).** Cache the *result* — a DTO — with `HybridCache` (Module 10's current default), keyed by the query's inputs, with explicit invalidation on write:

```csharp
var summary = await _cache.GetOrCreateAsync(
    $"order-summary:{tenantId}:{orderId}",
    async ct => await _db.Orders
        .Where(o => o.Id == orderId)
        .Select(o => new OrderSummary(...))
        .FirstOrDefaultAsync(ct),
    ct: ct);
```

You cache a plain, immutable DTO — not an entity, which would be a tracked object graph with a dead context behind it. HybridCache gives you stampede protection and the L1/L2 split for free.

**Second-level cache interceptors** (e.g. `EFCoreSecondLevelCacheInterceptor`) intercept commands and cache result sets transparently. Attractive, and I'd push back in a design review: transparent caching makes staleness invisible at the call site, invalidation becomes a heuristic over table names, and the first genuinely stale read is very hard to trace. If you use one, scope it to a small allow-list of queries you've reasoned about explicitly.

**What EF caches natively** is worth being precise about, because candidates over-claim: the model, compiled queries, and the internal service provider. **EF does not cache result data.** There is no first-level cache in the NHibernate sense — the identity map is per context and only prevents duplicate *instances*, not duplicate *queries*. `Find()`/`FindAsync()` is the one exception, and it's a useful one: it checks the identity map first and only queries if the key isn't already tracked.

---

## Concept 52 — JSON columns and complex types as a denormalization tool

Module 12 made the point that "we need schema flexibility for part of the model" is frequently a reason to use a JSON column rather than to add a document database. EF is where that becomes concrete.

EF 10 maps complex types to JSON columns, and with `UseAzureSql` or compatibility level 170 (SQL Server 2025) it defaults to the native `json` type rather than `nvarchar`. Existing `nvarchar` JSON columns are converted on the first migration unless you opt out — a migration-review item, not a surprise you want in production.

```csharp
modelBuilder.Entity<Order>(b =>
{
    b.ComplexProperty(o => o.ShippingAddress, a => a.ToJson());
    b.ComplexCollection(o => o.Items, i => i.ToJson());
});
```

You query into it with ordinary LINQ, and EF translates to `JSON_VALUE(...)` with the `RETURNING` clause. EF 11 adds the rest of the toolkit: `EF.Functions.JsonPathExists()` translating to `JSON_PATH_EXISTS`; `JSON_CONTAINS` replacing the `OPENJSON`-based translation for `Contains` over primitive collections at compatibility level 170 (with the caveat that `JSON_CONTAINS` doesn't support null values, so EF falls back to `OPENJSON` when it can't prove non-nullability); and JSON indexes created and scaffolded for paths inside complex types mapped to JSON columns.

**The design rule:** JSON columns are for data that is **always read and written together as a unit, and rarely filtered on individually**. Order line snapshots, a settings blob, an audit payload, a document's metadata. The moment you need to filter or aggregate across a JSON path at scale, you want a real column with a real index — or you want that path indexed explicitly, now that EF 11 can create a JSON index for it.

And the boundary: this is denormalization inside a relational database, with all of Module 12's trade-offs. You've traded update anomalies and join flexibility for locality. Say that out loud when you propose it.

---

## Concept 53 — Vector search and AI workloads

This is the newest surface and a legitimate differentiator right now, because most candidates don't know it exists.

EF 10 brought full support for the `vector` data type and `VECTOR_DISTANCE()`, available on Azure SQL Database and SQL Server 2025, via a `SqlVector<float>` property:

```csharp
public class Blog
{
    [Column(TypeName = "vector(1536)")]
    public SqlVector<float> Embedding { get; set; }
}

var similar = await db.Blogs
    .OrderBy(b => EF.Functions.VectorDistance("cosine", b.Embedding, queryVector))
    .Take(3)
    .ToListAsync();
```

That's **exact** k-nearest-neighbour — correct, and O(rows). EF 11 adds the approximate path: SQL Server 2025 supports approximate search over a vector index, giving much better performance at the cost of returning approximately rather than exactly similar items, and EF 11 can create vector indexes through migrations (`HasVectorIndex`) and query them with `VectorSearch(...).Take(k).WithApproximate()`, translating to the `VECTOR_SEARCH()` table-valued function. Both `VECTOR_SEARCH()` and vector indexes are currently experimental in SQL Server itself and subject to change — a caveat to state, not skip.

Two more EF 11 items that are directly architectural:

- **Vectors are excluded from entity `SELECT`s by default**, because vectors are usually ingested and searched but not read back — with a benchmark showing almost 9× locally and about 22× against a remote Azure SQL database. This is Concept 16's lesson promoted to a framework default.
- **Hybrid search** — combining full-text ranking with vector distance and merging the rankings (reciprocal rank fusion) — is now expressible in LINQ, using `FreeTextTable`/`ContainsTable` (which return a rank alongside the entity) joined against a `VectorSearch`, with `FullJoin` letting highly-ranked rows from either side contribute.

On Cosmos, the equivalent set arrived in EF 10: full-text search, the `Rrf` reciprocal-rank-fusion function for hybrid search, and vector similarity search exiting preview.

**The architectural point to make:** for a RAG feature over data you already store relationally, the cheapest correct architecture is usually **no new datastore**. Vectors in a column next to the row they describe, one query that filters by tenant and permission *and* ranks by similarity, one transaction, one backup. A dedicated vector database earns its place when scale, index type, or ingestion rate demands it — not by default. That's the same "consider the cheaper option first" move Module 12 rewarded.

---

## Concept 54 — Startup cost

Module 17, Concept 63 and Module 18, Concept 7 established the startup budget. EF's contributions to it:

| Cost | When | Mitigation |
|---|---|---|
| Model building | First use of the context | Compiled model (Concept 4) |
| Query compilation | First execution of each distinct query | Precompiled queries (Concept 21) |
| Provider/internal service-provider setup | First context | Cached by default; don't disable `EnableServiceProviderCaching` |
| JIT of the shaper and pipeline | First query | ReadyToRun; AOT when it's viable |
| First connection + TLS handshake + login | First query | Warm-up query at startup, before readiness |

The pattern for a serverless or scale-to-zero service is a **warm-up**: after `Build()` and before `Run()`, create a scope, run one trivial query against each context, and only then report ready. It pays the model build, the first query compilation, the JIT, and the connection handshake on your time rather than a user's.

```csharp
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    _ = await db.Orders.Where(o => false).Select(o => o.Id).FirstOrDefaultAsync();
}
```

Note that this belongs after `Build()` and before `Run()` — Module 18, Concept 5 — and that it is *not* where migrations go (Concept 57).

For AOT, the honest summary from Concept 21 stands: experimental, not production-ready, dynamic LINQ unsupported, and the pragmatic answer for an AOT-required service is a seam at the data layer rather than a fight with the precompiler.

---

# Part F — Migrations and schema evolution at scale

---

## Concept 55 — What a migration actually is

Three artifacts, and knowing all three is the difference between using migrations and understanding them:

1. **A migration class** — `Up(MigrationBuilder)` and `Down(MigrationBuilder)`, describing schema operations in a provider-agnostic API. Its ID is a timestamp plus your name, and **ordering is lexicographic on that ID**, which is why the timestamp prefix exists.
2. **The model snapshot** (`AppDbContextModelSnapshot.cs`) — the complete model *as of the last migration*. `dotnet ef migrations add` works by diffing your current model against this snapshot; it never looks at the database.
3. **`__EFMigrationsHistory`** — a table in the database with one row per applied migration ID, plus the EF version that produced it. This is how "which migrations are pending" is answered.

Two implications people get wrong:

- **`migrations add` does not connect to the database.** The diff is model-vs-snapshot. So if someone changed the database by hand, EF doesn't know, and your next migration will try to re-apply or conflict. **Schema drift is invisible to EF by construction** — Concept 63 is how you detect it.
- **The generated migration is a suggestion.** EF sometimes generates a table drop-and-recreate for a column change that could be an `ALTER COLUMN`, and it generates data-destroying operations without hesitation. **Read every generated migration before committing it.** This is the single most useful habit in this whole part, and saying it unprompted reads as production experience.

---

## Concept 56 — The snapshot and team merges

The snapshot is a generated file that two developers will inevitably edit on two branches. The rule: **never hand-merge a model snapshot.** Take one side, then regenerate — usually: remove your migration, rebase onto main, re-add the migration.

EF 11 makes divergence impossible to miss. The model snapshot now records the ID of the latest migration; when two developers create migrations on divergent branches, both branches modify that value, causing a source-control merge conflict that alerts the team to resolve the divergence — typically by discarding one migration tree and creating a unified one.

That's a nice example of a design principle worth naming in an interview: **turn a silent correctness problem into a loud, early failure.** Same family as `ValidateOnStart` (Module 18, Concept 4) and `ValidateOnBuild`.

EF 11 also improves a related diagnostic: when `Migrate()` detects pending model changes, EF can now distinguish "you really changed your model" from "your snapshot was generated by an older major version of EF Core," raising `OldMigrationVersionWarning` instead of `PendingModelChangesWarning` and pointing at regenerating the snapshot rather than adding a redundant migration.

---

## Concept 57 — Applying migrations: four options, one recommendation

| Option | What it is | Verdict |
|---|---|---|
| `db.Database.Migrate()` at app startup | The app applies pending migrations when it boots | **No** for production |
| `dotnet ef database update` in the pipeline | The CLI applies them | Needs the SDK and the project on the deploy agent |
| `dotnet ef migrations script --idempotent` | Generate SQL; a DBA or pipeline step runs it | **Yes** — reviewable, auditable, tool-agnostic |
| `dotnet ef migrations bundle` | A self-contained executable that applies migrations | **Yes** — no SDK needed on the target, good for containers |

The case against startup migration, which Module 12 stated and which deserves its full argument here:

1. **It races across instances.** Three replicas boot simultaneously and all three try to apply the same migration. EF 9+ mitigates this with a lock (Concept 58), but a mitigated race is still a startup dependency.
2. **There is no review gate.** Nobody sees the SQL before it runs against production.
3. **It couples schema change to deployment timing.** You cannot apply schema first and code second, which is the entire basis of zero-downtime change (Concept 59).
4. **Failure mode is terrible.** A migration that fails mid-way leaves the app crash-looping with a half-applied schema, during a deploy, at the worst moment.
5. **It requires the app's identity to have DDL rights.** Your web app should be able to read and write rows, not drop tables. This is a security argument, and it's the one that usually wins the conversation.

**The recommendation:** generate an idempotent script or a bundle in CI, apply it as a **separate pipeline step** — a Kubernetes Job, an Azure DevOps/GitHub Actions task, a DBA-executed script — that runs *before* the new code rolls out, with a migration identity that has DDL rights and an application identity that doesn't.

```bash
dotnet ef migrations script --idempotent --output artifacts/migrate.sql --context AppDbContext
dotnet ef migrations bundle --self-contained -r linux-x64 --output artifacts/migrate
```

`--idempotent` wraps every operation in a "has this been applied?" check, so the script is safe to run against a database at any migration level — which is what makes it deployable rather than environment-specific.

EF 11 adds a genuinely useful convenience for constrained environments: `dotnet ef database update --add` creates and applies a migration in one step, compiling it with Roslyn at runtime, for scenarios like .NET Aspire and containerized applications where recompilation isn't possible. Useful in development and for Aspire workflows; it does not change the production recommendation.

---

## Concept 58 — Migration locking, user transactions, and pending-model-changes

Three behaviours introduced around EF Core 9 that every senior .NET engineer should know, because they change what upgrade does to existing code.

**1. Migration locking.** EF 9+ acquires a database-wide lock before applying migrations, so two processes cannot migrate concurrently. The implementation is provider-specific: SQL Server uses a session-level application lock (`sp_getapplock`) that is automatically released when the connection closes; PostgreSQL uses `LOCK TABLE "__EFMigrationsHistory" IN ACCESS EXCLUSIVE MODE`; SQLite has no application locks, so EF creates a `__EFMigrationsLock` table and inserts a row. The SQLite caveat is worth knowing because it bites in CI: if the process is killed mid-migration, the lock row may not be cleaned up, blocking every subsequent migration indefinitely, and the fix is to drop the `__EFMigrationsLock` table.

**2. The user-transaction warning.** Starting with EF Core 9, `Migrate` and `MigrateAsync` start a transaction and execute commands using an execution strategy, so the old pattern of wrapping `Migrate()` in your own `TransactionScope`/`BeginTransaction` now throws `MigrationsUserTransactionWarning` — because an explicit transaction prevents the database lock from being acquired, leaving the database unprotected against concurrent migration. The mitigation is to remove the external transaction; if your scenario genuinely requires one and you have another mechanism preventing concurrency, suppress the specific event ID.

**3. `PendingModelChangesWarning`.** Starting with EF Core 9, `Migrate()`/`MigrateAsync()` throw when the model has pending changes relative to the last migration — i.e. someone changed an entity and forgot to add a migration. The intended detection point is CI: `dotnet ef migrations has-pending-model-changes` as a pipeline check before deployment.

That last one is the highest-value line in this concept. **Add `has-pending-model-changes` to your CI build.** It converts "we deployed and the column wasn't there" into a red build, for one line of YAML.

---

## Concept 59 — Expand/contract: the only safe way to change a schema under load

If old code and new code run simultaneously — and during any rolling deploy, blue/green, or canary, they do — then **every schema change must be compatible with both versions**. That constraint produces the expand/contract (parallel change) pattern, and it is the single most important operational idea in this part.

**Renaming `Customer.Name` to `Customer.FullName`, done safely:**

| Step | Deploy | Schema | Code |
|---|---|---|---|
| 1 | Schema only | `ADD FullName NULL` | unchanged (ignores it) |
| 2 | Code | — | Writes **both** columns; reads `Name` |
| 3 | Data | Backfill `FullName` from `Name` in batches | — |
| 4 | Code | — | Reads `FullName`; still writes both |
| 5 | Schema | `ALTER FullName NOT NULL` | — |
| 6 | Code | — | Stops writing `Name` |
| 7 | Schema | `DROP COLUMN Name` | — |

Seven deployments to rename a column. That is the honest cost of zero downtime, and stating it plainly — rather than pretending there's a shortcut — is exactly the trade-off literacy an architect round is testing.

**The catalogue of operations and their safety:**

| Operation | Safe online? | Note |
|---|---|---|
| Add nullable column | ✅ | The base case |
| Add column with default | ⚠️ | Modern SQL Server/PostgreSQL make this metadata-only; older versions rewrite the table |
| Add `NOT NULL` column | ❌ | Expand/contract: nullable → backfill → constrain |
| Drop column | ❌ until no deployed code reads it | Contract phase only |
| Rename column/table | ❌ | Never atomic across two code versions — always add/backfill/switch/drop |
| Add index | ⚠️ | Use `CREATE INDEX ... WITH (ONLINE = ON)` / PostgreSQL `CONCURRENTLY`; EF won't emit that for you — write it with `migrationBuilder.Sql(...)` |
| Widen a type (`varchar(50)`→`(200)`) | ✅ usually | Metadata-only on most engines |
| Narrow a type | ❌ | Rewrite + possible data loss |
| Add FK / check constraint | ⚠️ | Validate `NOT FOR REPLICATION`/`NOT VALID` then validate separately, to avoid a long table lock |

**And the .NET-specific trap:** EF's generated migration for a column *rename* is frequently `DropColumn` + `AddColumn` — which silently destroys the data. EF can generate `RenameColumn` when it can infer the rename, but it often can't. **Read the migration.** This is the concrete reason for Concept 55's rule.

---

## Concept 60 — Data migrations and backfills

`Up()` is the wrong place for a data backfill, for four reasons:

1. It runs inside the migration's transaction and lock, so a 40-minute update holds both.
2. It isn't restartable — a failure at row 3 million re-runs from zero.
3. It can't be throttled, so it competes with live traffic.
4. It isn't observable — no progress, no metrics, no ability to pause.

**Do this instead:** the migration adds the column (nullable); a separate, idempotent, batched job fills it; a later migration adds the constraint.

```csharp
// The backfill job, not the migration
const int BatchSize = 5_000;
while (!ct.IsCancellationRequested)
{
    var affected = await db.Customers
        .Where(c => c.FullName == null)
        .OrderBy(c => c.Id)
        .Take(BatchSize)
        .ExecuteUpdateAsync(s => s.SetProperty(c => c.FullName, c => c.Name), ct);

    if (affected == 0) break;
    await Task.Delay(_pauseBetweenBatches, ct);   // give the database room to breathe
}
```

Four properties to point out about that loop: it is **batched** (bounded lock duration, bounded log growth), **restartable** (the predicate is the progress marker), **throttled** (the delay), and **set-based** (`ExecuteUpdate`, so no change tracker — Concept 34). Add logging per batch and it's observable too.

Small, safe seed data — reference tables, lookup values — is different and *can* live in the model via `HasData`, which EF turns into migration inserts and diffs across migrations. Keep it genuinely small and genuinely static; `HasData` for anything with a generated key or a real lifecycle gets painful fast.

---

## Concept 61 — Multiple contexts, schemas, and migration ownership

Following Concept 44, once you have more than one `DbContext` over a database:

- Each context needs its **own history table**: `MigrationsHistoryTable("__EFMigrationsHistory", "ordering")`.
- Each context's tables belong in its **own schema** (`HasDefaultSchema("ordering")`), which makes ownership visible and lets you grant permissions per module.
- Every CLI invocation needs `--context`, and the migrations directories should be separate (`--output-dir Migrations/Ordering`).
- EF 11's wildcard helps with the pipeline tedium: `migrations list`, `migrations script`, `database update`, and `database drop` accept `--context "*"` to target all contexts at once.
- EF 11's `.config/dotnet-ef.json` removes the other half — you can set `project`, `startupProject`, `context`, and `framework` once, and override on the command line only when needed.

The governance point, which matters more than the mechanics: **in a modular monolith, schema ownership is the boundary that makes the modularity real.** If the Billing module can read the Ordering module's tables, you have one module with two namespaces. Enforce it with schemas and database permissions if you can; enforce it with review if you can't (Module 21).

---

## Concept 62 — Non-EF-owned schema

Not everything in the database should be EF's. Views, functions, stored procedures, and indexes with options EF can't express (filtered, online, partitioned, columnstore) are all legitimate.

**Mapping in what EF should read:**

```csharp
modelBuilder.Entity<OrderSummaryView>()
    .HasNoKey()                                     // or .HasKey(...) if genuinely unique
    .ToView("vw_OrderSummary", "reporting");

modelBuilder.HasDbFunction(() => AppDbContext.DistanceKm(default, default, default, default))
            .HasName("fn_DistanceKm").HasSchema("geo");
```

Now LINQ can query the view and call the function, and neither is managed by migrations.

**Managing what EF doesn't own:** either put the DDL in a migration via `migrationBuilder.Sql(...)` (so it's versioned with the rest), or manage it with a separate tool (a DACPAC, DbUp, Flyway). The mistake is a third option: nobody manages it, and it exists only in production because someone ran it once.

The rule: **every object in the database has exactly one owner, and the owner is a file in source control.** EF's model, or a SQL script, or an explicit `migrationBuilder.Sql`. Not tribal memory.

---

## Concept 63 — Testing migrations, and detecting drift

Four checks, in increasing order of what they prove:

1. **`dotnet ef migrations has-pending-model-changes`** in CI. Fails the build when the model and the last migration disagree. One line, catches the most common mistake.
2. **Apply migrations to a clean database in CI**, with Testcontainers:
   ```csharp
   await using var sql = new MsSqlBuilder().Build();
   await sql.StartAsync();
   await using var db = new AppDbContext(Options(sql.GetConnectionString()));
   await db.Database.MigrateAsync();          // proves the full chain applies from zero
   ```
3. **Apply migrations to a restored copy of production** (or a production-shaped dataset). This is what catches the migration that's instant on an empty table and takes 40 minutes on 200 million rows — the check most teams skip and most incidents come from.
4. **Drift detection.** Because `migrations add` never looks at the database (Concept 55), out-of-band changes are invisible. Detect them by comparing the deployed schema to the migrated schema — a schema-compare tool, a DACPAC diff, or a scripted check in a scheduled pipeline. In a mature organization, revoke DDL rights from humans in production and drift stops existing; until then, detect it.

For **application tests** that happen to use the database, don't run migrations at all — Concept 70.

---

## Concept 64 — Rollback: forward-only, and what `Down()` is for

EF generates `Down()` and `dotnet ef database update <PreviousMigration>` will run it. In production, plan not to.

Why forward-only:

- **`Down()` cannot restore dropped data.** The schema reverts; the rows don't.
- **It is rarely tested**, so at the moment you need it you're running untested code under pressure.
- **It doesn't compose with deployed code.** Rolling the schema back while the new code is still running on some replicas is a compound failure.
- **With expand/contract (Concept 59), you don't need it.** Every step is individually backward-compatible; recovery means deploying the previous *code*, which still works against the current schema.

So: **roll code back, roll schema forward.** If a migration was genuinely wrong, write a new migration that corrects it. Keep `Down()` for local development, where it's genuinely handy for iterating on a migration you haven't committed yet.

The one place to invest instead: **a tested restore.** For irreversible operations (dropping a column, narrowing a type), the real safety net is a point-in-time restore you have actually practised, not a `Down()` method you haven't.

---

# Part G — The architect's view

---

## Concept 65 — Where the ORM stops

Module 12, Concept 60 stated the position; here's the full argument, which is one of the highest-value things you can say unprompted in a design round.

**EF Core is excellent at:** loading an aggregate by key, changing it, and saving it. Object-graph writes with referential integrity and concurrency control. Schema evolution. Type-safe queries that refactor with your code. Developer velocity on CRUD-shaped work.

**EF Core is a poor fit for:** reporting and analytics queries with complex aggregation, set-based batch operations over millions of rows, queries where you must control the execution plan, hierarchical/recursive queries (CTEs), anything needing engine-specific features EF doesn't surface, and hot read paths where materialization overhead is actually measurable.

**The mature position:** *"EF Core for the write model, Dapper or raw SQL for demanding reads."* That is CQRS at the persistence level without any of the ceremony — no separate database, no event sourcing, no MediatR requirement. Two implementations behind two interfaces, sharing a connection and a transaction when they need to.

```csharp
// Write side: aggregate, invariants, concurrency, events
var order = await _db.Orders.Include(o => o.Lines).FirstAsync(o => o.Id == id, ct);
order.AddLine(sku, qty);                              // domain method
await _db.SaveChangesAsync(ct);

// Read side: exactly the shape the screen needs, no ORM in the way
const string Sql = """
    SELECT o.Id, o.PlacedAtUtc, o.Total, c.Name AS CustomerName,
           (SELECT COUNT(*) FROM OrderLines l WHERE l.OrderId = o.Id) AS LineCount
    FROM Orders o JOIN Customers c ON c.Id = o.CustomerId
    WHERE o.TenantId = @TenantId AND o.Status = @Status
    ORDER BY o.PlacedAtUtc DESC OFFSET @Skip ROWS FETCH NEXT @Take ROWS ONLY
    """;
var rows = await _conn.QueryAsync<OrderListRow>(Sql, new { TenantId, Status, Skip, Take });
```

**Say the cost too**, because that's what makes it a trade-off rather than a preference: two data-access technologies to learn, SQL that doesn't refactor when you rename a column, and a second place to remember tenant filtering — which is precisely why the tenant predicate must be enforced at the database (RLS) and not only in the ORM (Concept 10).

---

## Concept 66 — Repository and unit of work over EF: the honest analysis

The question is a near-certainty in a .NET architecture interview, and the wrong answers are "always" and "never."

**The case against.** `DbSet<T>` *is* a repository (`IQueryable` + `Add`/`Remove`/`Find`). `DbContext` *is* a unit of work (`SaveChanges`). A generic `IRepository<T>` over EF adds a layer that reimplements the same API with less capability — it usually loses `Include`, projection, `AsNoTracking`, split queries, and `ExecuteUpdate` — and its stated benefit, swapping the ORM, is a thing approximately nobody does and which the repository wouldn't survive anyway (the leaky bits are the query semantics, not the method names).

**The case for.** A repository named for the *domain*, not for the *entity*, buys three real things:
1. **It enforces the aggregate boundary.** `IOrderRepository.GetForUpdateAsync(id)` can guarantee the right `Include`s so invariants can be checked. A generic `IRepository<Order>` can't.
2. **It gives the domain a persistence-free vocabulary.** `FindOverdueInvoices()` reads like the business; `Where(i => i.DueDate < x && !i.Paid)` reads like a database.
3. **It's a seam for testing and for the read/write split** (Concept 65) — and, for what it's worth, for the AOT seam of Concept 21.

**The position I'd defend:** *"No generic repository. Yes to a small number of aggregate-shaped repositories on the write side, which return materialized aggregates and never `IQueryable`. On the read side, no repository at all — query objects or handlers that project straight to DTOs, because an abstraction whose only job is to hide `Select` isn't earning anything."*

And the unit of work: **don't wrap `SaveChanges` in an `IUnitOfWork`** unless you have a genuine reason. If each repository takes the same scoped `DbContext`, they already share a unit of work. The failure mode to name is a repository that calls `SaveChanges` internally — now each repository is its own transaction, and a two-aggregate operation can half-commit.

---

## Concept 67 — `IQueryable` leakage and the specification pattern

If a repository returns `IQueryable<T>`, it has abstracted nothing: callers can compose arbitrary queries, execution is deferred past the repository's lifetime (and possibly past the context's), and the "swap the ORM" argument is dead on arrival because the caller is writing LINQ-to-Entities.

Worse, it's a correctness hazard: a caller can compose something untranslatable (Concept 15) or accidentally enumerate outside the scope (Concept 38), and the failure surfaces far from the repository.

**The specification pattern** is the usual reconciliation: encapsulate a query's criteria, includes, ordering, and paging as an object; let the repository apply it.

```csharp
public sealed class OverdueInvoicesSpec : Specification<Invoice>
{
    public OverdueInvoicesSpec(DateOnly asOf, int take)
    {
        Where(i => !i.Paid && i.DueDate < asOf);
        Include(i => i.Customer);
        OrderBy(i => i.DueDate);
        Take(take);
    }
}

var invoices = await _repo.ListAsync(new OverdueInvoicesSpec(today, 100), ct);
```

Honest assessment: specifications are composable, testable (you can assert on the expression), and they keep `IQueryable` inside the infrastructure. They're also a DSL your team has to learn, they rarely cover everything (projections and grouping get awkward), and for a CRUD service they're pure overhead. **Use them when you have genuinely reusable query criteria across several call sites.** Don't adopt them because a template repository has a `Specification` folder.

---

## Concept 68 — EF Core and DDD

Module 22 covers tactical DDD properly. What EF specifically gives you, and what it costs:

**Encapsulated collections.** Expose a read-only view; mutate through methods:

```csharp
public sealed class Order
{
    private readonly List<OrderLine> _lines = [];
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public void AddLine(string sku, int qty)
    {
        if (Status != OrderStatus.Draft) throw new DomainException("Order is not editable.");
        _lines.Add(new OrderLine(sku, qty));
    }
}

// mapping
builder.Metadata.FindNavigation(nameof(Order.Lines))!
       .SetPropertyAccessMode(PropertyAccessMode.Field);
```

**Private setters and backing fields.** EF writes through the field, so `public Guid Id { get; private set; }` and `private string _name;` both work. Materialization uses the field, so your invariants aren't bypassed by EF *and* aren't enforced on load — which is correct: a row already in the database was valid when it was written.

**Value objects as complex types** (Concept 5). `Money`, `Address`, `DateRange` with value semantics.

**Constructors.** EF prefers a parameterless constructor (any accessibility, including `private`), and can bind a constructor whose parameters match mapped properties. With modern C# (Module 16), `required` members and `init` accessors work; positional `record`s work as entity types but are a poor fit for *mutable* aggregates, and an excellent fit for read DTOs.

**Aggregate-sized loads.** The aggregate is the transactional boundary, so load the whole aggregate to change any of it, and cascade-delete within it (Concept 8). Don't load across aggregates — reference other aggregates by **ID**, not by navigation. This is the rule that keeps your object graph from becoming the whole database, and it's the one EF makes easiest to break.

**What it costs:** some mapping friction (shadow properties, field access modes, occasional fluent-API gymnastics), and a temptation to let persistence shape the domain. The senior move is to notice when you're bending the domain for EF's convenience and decide, deliberately, whether that's acceptable.

---

## Concept 69 — Multi-tenancy in EF Core

| Model | Isolation | EF mechanism | Model cache | Cost |
|---|---|---|---|---|
| **Shared schema, discriminator** | Logical | `TenantId` + global query filter + RLS | One model | Cheapest; one bug = a leak |
| **Schema per tenant** | Medium | `HasDefaultSchema` per tenant → **one model per tenant** | One per tenant | Model-cache growth (Concept 3) |
| **Database per tenant** | Strong | One model, tenant → connection string | One model | Most expensive to operate; migrations × N |

The recommendation from Module 12 stands: **pooled (shared schema) by default, with a silo escape hatch.** Concretely:

1. `TenantId` as the leading column of every tenant-scoped index and clustered key (Concept 47).
2. A **named** global query filter for tenancy, generated from a marker interface so it can't be forgotten (Concept 10).
3. **Row-Level Security** in the database as defence in depth, with the tenant set via `SESSION_CONTEXT` in a connection interceptor (Concept 35) — so a raw SQL query or a stray `IgnoreQueryFilters` still can't cross the boundary.
4. Tenant → connection string behind an interface, so the largest tenants can be moved to their own database without an application change.
5. **No tenant state on a pooled `DbContext`** — or if you must, use the pooling-aware factory with an explicit reset (Concept 39). This is where the leak actually happens in practice.
6. Migrations apply per database; with database-per-tenant that's a fan-out job with per-tenant status, retry, and reporting, which is real operational work you should cost before choosing the model.

---

## Concept 70 — Testing strategy

| Approach | What it proves | Verdict |
|---|---|---|
| **InMemory provider** | That your C# compiles and runs | **Don't.** Not a relational database: no FK constraints, no transactions, no SQL translation. Tests pass with queries that throw in production and fail with queries that work. Microsoft's own guidance steers away from it |
| **SQLite in-memory** | Translation happens, constraints exist, it's fast | Useful, with caveats: a different SQL dialect, no `decimal` precision, different date handling. A test that passes here can still fail on SQL Server |
| **Testcontainers** (real engine in Docker) | Everything — real SQL, real constraints, real migrations, real concurrency | **Yes**, for the tests that need a database |
| **Repository mocks** | That the code calls the method you mocked | Only for domain logic that genuinely doesn't touch data |

The practical shape:

```csharp
public sealed class DbFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder().Build();
    public string ConnectionString => _sql.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _sql.StartAsync();
        await using var db = new AppDbContext(Options(ConnectionString));
        await db.Database.MigrateAsync();            // proves the migration chain too
    }
    public Task DisposeAsync() => _sql.DisposeAsync().AsTask();
}
```

Three practices that make this sustainable: **one container per test class collection**, not per test; **isolate with transactions or a respawn/truncate step** between tests rather than recreating the database; and **keep the suite under a minute** (Module 18, Exercise 10) or people will stop running it.

And one high-value trick from Concept 24: assert on the **SQL**, not just the result. `Assert.Contains("IX_Orders_Tenant_Status", plan)` or a snapshot test on `ToQueryString()` catches an accidental N+1 or a lost index hint at review time rather than in production.

---

## Concept 71 — Provider choice

EF's provider model makes swapping databases look easy. It isn't, and knowing why is the point.

- **Relational providers** (SQL Server/Azure SQL, PostgreSQL/Npgsql, SQLite, MySQL, Oracle) share the relational query pipeline but differ in translations, type mappings, and migration capabilities. Code that works on one often works on another; the parts that don't are exactly the parts you care about (JSON support, full-text, vectors, `MERGE`, `RETURNING`, temporal tables, index options).
- **Cosmos** is the big conceptual difference. It's a document database behind a LINQ façade: no cross-partition transactions (EF 11's transactional batches are per partition), no joins across containers, a different cost model (RU/s), and provider-specific features (partition keys, full-text search, `Rrf`, vector search). EF 11 rebuilt its serialization on `System.Text.Json`, which brought performance but also breaking changes: `__jObject` removed, unmapped properties no longer round-tripped, and sync I/O fully removed.
- **In-memory** is a test double, not a database (Concept 70).

**The architectural caution:** EF gives you *portability of skills*, not portability of applications. A production application accumulates provider-specific decisions — index options, JSON mapping, concurrency tokens (`rowversion` vs `xmin`), value generation (identity vs sequence), full-text and vector features — and a migration between engines is a project, not a configuration change. Say that plainly when someone proposes "we'll stay database-agnostic": the abstraction is real but thin, and pretending otherwise produces a lowest-common-denominator data layer that's worse on every engine.

Module 27 goes deeper on Cosmos specifically; the EF-level summary here is: **use the Cosmos provider when you want EF's modelling and change tracking over documents you've already decided to store in Cosmos** — not as a way to avoid learning Cosmos.

---

## Concept 72 — The anti-pattern catalogue reviewers look for

| Anti-pattern | Why it's flagged |
|---|---|
| Generic `IRepository<T>` over `DbSet<T>` | Reimplements EF with less capability; the ORM-swap justification doesn't survive contact (Concept 66) |
| Repository returning `IQueryable<T>` | Abstracts nothing; defers execution past the context's life (Concept 67) |
| Repository calling `SaveChanges` internally | Every repository is its own transaction; two-aggregate operations half-commit |
| Lazy loading in a web API | Invisible N+1, sync I/O, disposal bugs (Concept 17) |
| `db.Update(graph)` on a DTO-derived object | Writes every column of every entity; mass-assignment risk (Concept 28) |
| `DbContext` as a singleton, or captured by one | Unbounded identity map, thread-safety failures, stale reads (Concepts 37–38) |
| Tenant state on a pooled `DbContext` | Cross-tenant leak (Concept 39) |
| `Database.Migrate()` at application startup | Race, no review gate, DDL rights for the web identity (Concept 57) |
| Data backfills inside `Up()` | Long lock, not restartable, not throttled (Concept 60) |
| `EnableSensitiveDataLogging` in production | PII in logs (Concept 24) |
| `FromSqlRaw` with concatenated input | SQL injection; EF 10's analyzer warns (Concept 22) |
| Entities returned directly from API endpoints | Couples schema to contract; serializer walks navigations |
| `ExecuteUpdate` next to tracked entities in one context | Stale tracker, skipped interceptors and events (Concept 34) |
| Business rules in query filters | Invisible behaviour; one `IgnoreQueryFilters` away from being wrong |
| No `CancellationToken` on async EF calls | Work continues after the client is gone (Module 18, Concept 21) |
| InMemory provider as the test database | Proves nothing about SQL, constraints, or transactions (Concept 70) |
| `.Result` / `.Wait()` on a query | Sync-over-async on a request thread (Module 15) |

---

# Putting it together

---

## Worked example 1 — "This endpoint got 10× slower after we added a field. Walk me through it."

**The reasoning I'd narrate:**

"First I'd want to know what 'a field' means, because there are three very different answers. Let me assume I can't ask and I have to diagnose.

I'd start by counting statements per request with command logging. If the count changed, the field added a navigation and somebody added an `Include` — and if it's a *collection* navigation on a query that already had one, that's a cartesian explosion: rows went from 100 to 100 × 20, and the latency is transfer and shaping, not the database. I'd confirm by looking at rows returned versus entities materialized.

If the statement count didn't change, I'd look at the SQL. Three candidates. One: the field is wide — an `nvarchar(max)`, a JSON blob, an embedding — and the query materializes entities, so it's now pulling megabytes it doesn't use. Two: the field broke a covering index; the projection used to be satisfied from the index and now needs a key lookup per row. Three: the field has a value converter and the predicate picked up a `CAST`, so a seek became a scan.

I'd get the actual plan from Query Store to distinguish two and three — `TagWith` on the query makes it findable. Scan versus seek, and the key-lookup operator, settle it immediately.

The fix in every case is the same shape: **project**. If the endpoint returns a list, it should select the six columns it renders, and then the wide field is irrelevant and the index can cover it again. If it genuinely needs the entity — because it writes — then I'd check whether the wide column should be a separate table or a lazily-loaded JSON column rather than part of the row every read pulls.

And the framework-level version of this exact lesson is EF 11's change to vectors: they stopped including `SqlVector<T>` columns in entity `SELECT`s by default, and measured roughly 9× locally and 22× against a remote database. Same problem, same fix, just made a default."

---

## Worked example 2 — "Review this repository."

```csharp
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public OrderRepository(AppDbContext db) => _db = db;

    public IQueryable<Order> GetAll() => _db.Orders;

    public async Task<Order> GetById(int id)
        => await _db.Orders.FirstOrDefaultAsync(o => o.Id == id);

    public async Task Update(Order order)
    {
        _db.Update(order);
        await _db.SaveChangesAsync();
    }

    public async Task<List<Order>> GetOverdue()
    {
        var all = await _db.Orders.Include(o => o.Lines).Include(o => o.Payments).ToListAsync();
        return all.Where(o => o.DueDate < DateTime.Now && !o.Paid).ToList();
    }
}
```

**Eight findings, ordered by severity:**

1. **`GetOverdue` loads the entire table** and filters in memory, with two collection `Include`s producing a cartesian product first. This is the worst line in the file by two orders of magnitude. It should be one `Where` in SQL, and — since nothing here mutates — a projection.
2. **`DateTime.Now` in a filter** is wrong twice: it's local time (should be `UtcNow`, and ideally an injected `TimeProvider` so it's testable — Module 16), and evaluating it inside what *should* be a translated predicate makes the query non-deterministic across a DST boundary.
3. **`Update(order)` marks the whole graph modified** (Concept 28), writing every column of every entity including ones nobody changed — which defeats concurrency tokens and creates mass-assignment risk. Load-then-mutate instead.
4. **`SaveChanges` inside the repository** makes each repository its own transaction (Concept 66). A two-aggregate operation can half-commit.
5. **`GetAll()` returns `IQueryable`**, so this "repository" abstracts nothing and lets callers defer execution past the context's lifetime (Concept 67).
6. **No `CancellationToken` anywhere.** Work continues after the client disconnects.
7. **`GetById` returns `Order` but can return null** — the signature lies. With NRTs on this wouldn't compile cleanly; the honest signature is `Task<Order?>`, or it should throw a domain `NotFoundException`.
8. **No tracking discipline.** `GetOverdue` tracks everything it loads for no reason (Concept 20).

**The rewrite I'd propose:**

```csharp
public sealed class OrderRepository(AppDbContext db) : IOrderRepository
{
    // Write side: returns a full aggregate, tracked, ready to mutate
    public Task<Order?> GetForUpdateAsync(int id, CancellationToken ct) =>
        db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);

    // Read side: projection, no tracking, filtered in SQL
    public Task<List<OverdueOrder>> GetOverdueAsync(DateTimeOffset asOf, CancellationToken ct) =>
        db.Orders
          .Where(o => o.DueDateUtc < asOf && !o.Paid)
          .OrderBy(o => o.DueDateUtc)
          .Select(o => new OverdueOrder(o.Id, o.DueDateUtc, o.Total, o.Lines.Count))
          .ToListAsync(ct);
}
// SaveChanges lives in the application service / command handler — one per unit of work.
```

---

## Worked example 3 — "Design the data access for a multi-tenant B2B SaaS."

**Decisions, with reasons:**

1. **Shared schema, `TenantId` discriminator**, with a per-tenant-database escape hatch behind an `ITenantConnectionResolver`. Rationale: operational cost. 5,000 tenants × migrations × backups × connection pools is an operations programme; one database with a leading index column is a design decision. Large tenants can be silo'd later without an application change (Concept 69).
2. **`TenantId` leads every tenant-scoped index and the clustered key**, because the query filter puts it in every predicate (Concept 47).
3. **A named global query filter**, generated from an `ITenantScoped` marker interface so nobody can forget one, and so the soft-delete filter can be disabled independently of it (Concept 10).
4. **Row-Level Security in the database**, with the tenant pushed into `SESSION_CONTEXT` by a connection interceptor. This is the control that survives a raw SQL query, a stray `IgnoreQueryFilters`, or a reporting tool (Concepts 35, 69).
5. **No tenant state on a pooled context.** Either don't pool, or use the pooling-aware factory with an explicit reset. This is the specific bug that causes cross-tenant data leaks in .NET, and it looks like a performance optimization (Concept 39).
6. **Tenant resolution at the edge, once**, into a scoped `TenantContext` record — not `IHttpContextAccessor` buried in the data layer (Module 18, Concept 45).
7. **Migrations as an idempotent script applied by a pipeline step**, with a migration identity that has DDL rights and an application identity that doesn't (Concept 57).
8. **Rate limiting and quotas partitioned by tenant, not IP** (Module 18, Concept 67), because one tenant's batch job must not consume another's capacity.

**The test that proves it:** a token for tenant A, a request that spoofs tenant B in every place tenancy could be read, and an assertion that zero rows come back — run against a real database with RLS enabled, not the InMemory provider.

---

## Worked example 4 — "Our pods restart every few hours with OOM. It's EF."

**The diagnosis path:**

"I'd take a `gcdump` or a `dotnet-gcdump` from a pod near its peak and look at what's retaining memory (Module 14, Module 17).

If the retention path runs through `StateManager` → `InternalEntityEntry`, it's the change tracker, and there are only four ways that happens. **One:** a `DbContext` with a lifetime longer than a unit of work — singleton, captured, or a background worker with one scope for its whole life (Concepts 37–38). **Two:** a request that materializes far more entities than it needs, so the tracker holds a huge graph for the request's duration — a missing `Where`, a missing projection, or a cartesian explosion (Concepts 16, 18). **Three:** streaming a large result set *with tracking on*, so the identity map grows exactly as fast as the result (Concept 50). **Four:** a context reused across many `SaveChanges` calls in a loop without clearing, so everything ever touched accumulates.

If the retention path doesn't go through the state manager, I'd check the **query cache**: EF uses `IMemoryCache` for query compilation and model building, so a cache key explosion — hand-built expression trees with runtime values as constants, or an `IModelCacheKeyFactory` keyed per tenant — grows memory with no tracked entities involved at all (Concepts 3, 13). The tell is the compiled-query-cache hit-rate counter being far below 100%.

The fixes map one-to-one: shorten the lifetime, project, `AsNoTracking` for streams, `ChangeTracker.Clear()` between batches, parameterize instead of hand-building trees, and take tenancy out of the model.

Then I'd add the control that keeps it fixed: an alert on the active-`DbContext` and cache-hit-rate counters, and a test that fails if a given endpoint materializes more than N entities."

---

## Worked example 5 — "We need to add a NOT NULL column to a 200-million-row table with zero downtime."

**The plan:**

"Seven steps, three deployments of schema and three of code, because old and new code run simultaneously during any rolling deploy.

1. **Migration 1:** add the column as **nullable**, with no default. On modern SQL Server and PostgreSQL adding a nullable column is metadata-only, so it's instant. Adding one with a default *may* rewrite the table on older engines — I'd check the target version rather than assume.
2. **Deploy code that writes the new column** on every insert and update, and still reads the old source of truth. Both versions of the code are now safe: the old one ignores the column, the new one populates it.
3. **Backfill as a batched, restartable, throttled job** — not in `Up()`. `ExecuteUpdate` over a bounded `Take(5000)` ordered by key, with a delay between batches so replication and the log can keep up, and metrics per batch so we can watch it (Concept 60). At 200 million rows this runs for hours; that's expected and it's why it's a job.
4. **Verify**: zero rows where the column is null, and a reconciliation query comparing old and new values.
5. **Migration 2:** `ALTER COLUMN ... NOT NULL`. On SQL Server this is a metadata check if the data is already clean; on PostgreSQL, add a `NOT VALID` check constraint first and `VALIDATE` it separately to avoid a long exclusive lock, then set `NOT NULL`.
6. **Deploy code that reads the new column** as the source of truth.
7. **Migration 3 (later, deliberately):** drop the old column, once no deployed version reads it. This is the contract step, and it's fine to leave it for a week.

The index for the new column goes in with `ONLINE = ON` (SQL Server Enterprise / Azure SQL) or `CREATE INDEX CONCURRENTLY` (PostgreSQL) — and EF won't generate that, so it's `migrationBuilder.Sql(...)` written by hand and reviewed.

Rollback plan: there isn't one, and there doesn't need to be. Every step is backward-compatible, so recovery is deploying the previous code, which still works against the current schema (Concept 64)."

---

## Worked example 6 — "Should we use EF Core or Dapper for this service?"

**The answer that scores:** *"Both, at different boundaries — and the interesting question is where the boundary goes."*

Then the reasoning:

"EF earns its place on the **write** side: an aggregate with several entities, invariants to enforce, optimistic concurrency, cascade rules, and schema evolution. Writing that by hand in Dapper means hand-writing change detection, dependency ordering, and concurrency checks — which is rewriting EF, badly.

Dapper or raw SQL earns its place on the **read** side when the query is shaped by the screen rather than by the model: joins across aggregates, aggregation, window functions, recursive CTEs, or anything where I need to control the plan. EF can express a lot of that now, but the SQL I'd write is clearer and the plan is mine.

I'd also name the cases where EF is the wrong tool outright: bulk loads (`SqlBulkCopy`), reporting (a read replica or a warehouse, Module 12), and any service where NativeAOT is a hard requirement, because EF's AOT support is still experimental and doesn't handle dynamic LINQ.

The cost of running both is real: two technologies, SQL that doesn't refactor, and a second place to remember tenant filtering — which is one more argument for enforcing tenancy in the database with RLS rather than only in the ORM.

For a small service with CRUD-shaped work, I'd start with EF alone and add the second path when a query actually justifies it. Starting with both is premature."

---

## Common questions and what a strong answer contains

**"What happens between `ToListAsync()` and the rows arriving?"** Cache lookup on the expression tree's shape → preprocessing and parameter extraction → navigation expansion → translation to a SQL expression tree → optimization → SQL plus a compiled shaper → connection opened, command executed, reader shaped into objects, identity map consulted if tracking, connection returned (Concepts 12, 13, 25).

**"Why does EF cache queries, and how can that go wrong?"** Translation is expensive, so EF caches by the tree's *shape*. It goes wrong when runtime values end up as `Expression.Constant` nodes — every call is a cache miss, the cache grows, and the database's plan cache fills with near-identical plans. Check the compiled-query cache hit rate (Concepts 13, 46).

**"Explain the N+1 problem in EF Core specifically."** Three shapes: lazy loading through proxies, a query inside a loop, and navigation access without an `Include` (which gives you *wrong* data rather than slow data). Fix hierarchy: project, then `Include`, then explicit load, then split query. Detect by counting statements per request and asserting on it in a test (Concept 19).

**"What is a cartesian explosion and what does `AsSplitQuery` cost?"** Two collection `Include`s multiply rows. `AsSplitQuery` trades one query for N+1 queries, and — the part people miss — gives up single-statement consistency, so a consistent read needs an explicit transaction (Concept 18).

**"`AsNoTracking` — when and why?"** For reads. The saving that matters isn't CPU, it's retention: no identity-map entry, no snapshot, and the graph becomes collectible immediately. Note `AsNoTrackingWithIdentityResolution` for shared references, and note that a projection to a DTO isn't tracked anyway (Concept 20).

**"How does change tracking work?"** Snapshot-based: EF copies original values at materialization and `DetectChanges` walks every tracked entity and property to diff them. Cost is O(entities × properties) and it runs on most public APIs, which is why `Add` in a loop is quadratic. Alternatives are notification entities or proxies (Concepts 25–27).

**"We save and the change isn't persisted, with no exception. Why?"** Almost always a mutable value behind a value converter without a `ValueComparer` — the snapshot holds the same reference, so in-place mutation is invisible to `DetectChanges`. Also possible: the entity isn't tracked, or `AutoDetectChangesEnabled` was left off (Concepts 7, 26).

**"What does `SaveChanges` do?"** Detect changes → apply cascades → generate values → topologically sort by FK dependency → batch → execute inside a transaction → propagate generated values back → accept changes. It does not validate (Concept 30).

**"How does EF batch writes?"** Statements are grouped per round trip; SQL Server defaults to a minimum of 4 and a maximum of 42 because batching is inefficient below 4 and degrades past ~40, while Npgsql defaults to 1,000. Raising it hits the 2,100-parameter limit on SQL Server (Concept 31).

**"How do you implement optimistic concurrency?"** `rowversion` on SQL Server, `xmin` on PostgreSQL, or a business column. EF adds the token to the `WHERE` and throws `DbUpdateConcurrencyException` on zero rows affected. Then the important half: resolution is a business decision — store wins, client wins, or merge — and for counters the better answer is a commutative `ExecuteUpdate` so there's no conflict at all (Concept 33).

**"When would you use `ExecuteUpdate`/`ExecuteDelete`?"** Set-based maintenance — expiring sessions, archiving, backfills, counters. Not for domain state changes, because they bypass the change tracker, concurrency tokens, interceptors, and therefore your outbox and domain events (Concept 34).

**"How would you implement the transactional outbox with EF?"** A `SaveChangesInterceptor` that drains domain events into outbox rows inside the same `SaveChanges`, so state and intent commit atomically; a background worker publishes with at-least-once delivery; consumers are idempotent. And the caveat: bulk paths don't pass through the interceptor (Concepts 35–36).

**"How long should a `DbContext` live?"** As long as one unit of work. In a web request that's usually the request, so scoped — but the principle is the consistency boundary, not the DI default. Longer than that gives you an unbounded identity map, quadratic `DetectChanges`, thread-safety failures, and retention (Concept 37).

**"What does `AddDbContextPool` do, and what's the risk?"** It reuses context instances to avoid re-initializing internal services, with a default pool size of 1024. The risk is that EF only resets state it knows about — your own fields survive into the next request, which in a multi-tenant app is a cross-tenant leak (Concept 39).

**"Context pooling versus connection pooling?"** Orthogonal. The ADO.NET driver pools connections; EF pools context objects. EF opens a connection just before each operation and closes it right after, so a request-scoped context does not hold a connection for the request (Concept 42).

**"`EnableRetryOnFailure` — what's the catch?"** With a user-initiated transaction EF can't retry and throws; you must wrap the whole unit in `strategy.ExecuteAsync(...)`, and that block must be idempotent because it can run twice. EF 11 also applies the execution strategy to `Migrate`/`EnsureCreated` (Concept 43).

**"How do you apply migrations in production?"** An idempotent script or a migration bundle, applied by a separate pipeline step before the new code rolls out, with a migration identity that has DDL rights and an application identity that doesn't. Not `Database.Migrate()` at startup — races, no review gate, and the web app shouldn't be able to drop tables (Concept 57).

**"How do you rename a column with zero downtime?"** You don't rename it. Add nullable → deploy code writing both → batched backfill → deploy code reading the new one → add the constraint → deploy code that stops writing the old one → drop it. Seven steps, and EF's generated migration for a rename is often drop-plus-add, which destroys data — read it (Concepts 55, 59).

**"How do you detect that someone changed the database by hand?"** EF can't: `migrations add` diffs the model against the *snapshot*, never the database. So you need an explicit drift check — a schema compare in a scheduled pipeline — plus `has-pending-model-changes` in CI for the opposite failure (Concepts 55, 63).

**"Repository over EF — yes or no?"** No generic repository. Aggregate-shaped repositories on the write side that return materialized aggregates and never `IQueryable`; no repository on the read side, because an abstraction that only hides `Select` isn't earning its keep (Concepts 66–67).

**"How do you test EF code?"** Testcontainers with the real engine, migrations applied, one container per collection, isolation by transaction or truncate. Not the InMemory provider — it isn't a relational database and will pass queries that throw in production (Concept 70).

**"Where does EF stop?"** Reporting, set-based batch work, plan control, recursive queries, and hot read paths. The mature position is EF for the write model, Dapper or SQL for demanding reads — CQRS at the persistence level, without the ceremony (Concept 65).

**"What's new in EF Core recently?"** EF 10 (LTS to November 2028): complex types as the replacement for owned types, a new parameterized-collection default with padded parameters, named query filters, `ExecuteUpdate` with a non-expression lambda and into JSON, SQL Server 2025's `vector` and `json` types, redaction of inlined constants in logs. EF 11 (GA November 2026): pruned joins and dropped redundant `ORDER BY` keys (~29%/22%), vectors excluded from entity `SELECT`s by default (~9–22×), `VECTOR_SEARCH` with approximate indexes, `JSON_CONTAINS` and JSON indexes, complex types on TPT/TPC, Cosmos rebuilt on `System.Text.Json` with transactional batches, and the migration-ID-in-snapshot merge-conflict trick (Orientation).

**"Can you use EF Core with NativeAOT?"** Not in production yet. NativeAOT support and query precompilation are explicitly experimental, publishing still emits trimming warnings, and dynamic LINQ — a `DbSet` reached through a generic wrapper or base class — isn't supported. Precompiled queries *are* useful without AOT for startup time. If AOT is a hard requirement, put a seam at the data layer (Concept 21).

---

## Mistakes vs. senior signals

| Common mistake | What a senior candidate does instead |
|---|---|
| "EF is slow" | Names which of the three machines is slow — model, query cache, or change tracker — and measures it |
| Describes LINQ-to-SQL as "EF translates your code" | Describes the expression tree, the cache key, and the stages of the pipeline |
| "Always use `AsNoTracking`" | "Project for reads; then tracking is moot. `AsNoTracking` is a retention decision, not a CPU one" |
| Fixes N+1 by adding `Include` | Projects first, and knows `Include` can trade N+1 for a cartesian explosion |
| Recites `AsSplitQuery` as the fix | Names the cost: N+1 round trips and loss of single-statement consistency |
| Treats `DetectChanges` as free | Knows it's O(entities × properties), why `Add` in a loop is quadratic, and what EF 11's `GetEntriesForState` is for |
| Uses `Update(entity)` for edits | Load-then-mutate, so only changed columns are written and invariants are enforced |
| Catches `DbUpdateConcurrencyException` and retries | Resolves deliberately — store wins, client wins, or merge — and prefers commutative updates for counters |
| Puts `SaveChanges` in the repository | One unit of work per operation; `SaveChanges` at the application-service boundary |
| "`DbContext` is scoped" | "A `DbContext` lives as long as its unit of work; scoped is the usual consequence in a web request" |
| Enables context pooling for speed | Knows it resets EF state only, and that tenant state on a pooled context is a cross-tenant leak |
| Adds `EnableRetryOnFailure` and moves on | Knows user transactions require `strategy.ExecuteAsync`, and that the block must be idempotent |
| Runs `Database.Migrate()` at startup | Idempotent script or bundle, applied by a pipeline step, with separate DDL credentials |
| Renames a column in one migration | Expand/contract, seven steps, and reads the generated migration for drop-plus-add |
| Puts a backfill in `Up()` | A batched, restartable, throttled job with metrics |
| Builds a generic `IRepository<T>` | Aggregate-shaped repositories on writes, query objects on reads, no `IQueryable` escaping |
| Tests with the InMemory provider | Testcontainers with the real engine, migrations applied, and assertions on the generated SQL |
| Unaware of current platform state | Knows EF 10 is LTS to Nov 2028, what complex types replaced, EF 11's join pruning and vector defaults, and that AOT is still experimental |

---

## Practice exercises

**Exercise 1 — Read the SQL for ten queries (1 hr).** Write ten queries of increasing complexity — filter, projection, single `Include`, two collection `Include`s, `GroupBy`, a subquery in a projection, `Any` in a predicate, a `Contains` over a list, a left join, keyset pagination. Predict the SQL for each *before* running `ToQueryString()`. Score yourself. The ones you got wrong are your actual gaps.

**Exercise 2 — Cartesian explosion, measured (1 hr).** Seed 200 orders × 20 lines × 3 payments. Run the two-`Include` query and record: rows returned by the database, bytes transferred, entities materialized, elapsed time, allocated bytes. Repeat with `AsSplitQuery` and with a projection. Write down the three-way comparison and the consistency caveat.

**Exercise 3 — N+1 test harness (1 hr).** Write a `DbCommandInterceptor` that counts commands, wire it into a `WebApplicationFactory` fixture, and write a test asserting an endpoint issues at most N statements. Then introduce a lazy-loading proxy and watch the test fail. Keep the harness — this is the one control that stops N+1 recurring.

**Exercise 4 — `DetectChanges` cost curve (1 hr).** Insert 100, 1,000, 10,000 and 50,000 entities using `Add` in a loop, then `AddRange`, then with `AutoDetectChangesEnabled = false`. Plot elapsed time against count for each. Confirm the quadratic shape of the first, and write one paragraph explaining why to a junior engineer.

**Exercise 5 — The silent value-converter bug (45 min).** Map a `List<string>` through a JSON converter without a `ValueComparer`. Load an entity, mutate the list in place, save, reload. Observe the change is gone. Add the comparer and observe it persists. Then explain it in terms of snapshot semantics in three sentences.

**Exercise 6 — Concurrency, three resolutions (1.5 hrs).** Add a `rowversion` token. Write a test that simulates two concurrent edits and implement all three resolutions (store wins, client wins, merge) behind a strategy. Then implement the fourth option — a commutative `ExecuteUpdate` — and write down which you'd use for a product description, a stock count, and an account balance.

**Exercise 7 — The pooled-context tenant leak (1 hr).** Build a two-tenant app with a `TenantId` field on the context, a global query filter, and `AddDbContextPool`. Write an integration test that issues interleaved requests for both tenants and demonstrates the leak. Then fix it two ways — a pooling-aware factory with reset, and moving tenancy out of the context entirely — and compare.

**Exercise 8 — Parameterization and the plan cache (1.5 hrs).** Run `ids.Contains(x.Id)` with collections of 3, 8, 17 and 40 elements under each `ParameterTranslationMode`. Capture the generated SQL for each and count distinct SQL texts. Then query the database's plan cache (`sys.dm_exec_cached_plans` or `pg_stat_statements`) and count plans. Write down which mode you'd default to and why.

**Exercise 9 — Streaming an export (1 hr).** Export 500,000 rows to CSV with `ToListAsync`, then with `AsAsyncEnumerable` + projection + `AsNoTracking`. Measure peak working set, Gen 2 collections, LOH allocations, and time-to-first-byte. Note how long each holds the connection.

**Exercise 10 — Zero-downtime column change (2–3 hrs).** On a table with ~1 million seeded rows, execute the full seven-step expand/contract from Worked Example 5 while a load generator runs continuous reads and writes. Count errors at each step. Then do it "the obvious way" (one migration, `NOT NULL` with a default) and count errors again.

**Exercise 11 — EF + Dapper in one transaction (1.5 hrs).** Write a command handler that loads an aggregate with EF, mutates it, saves, and then runs a Dapper query on the *same connection and transaction*. Prove with a test that a failure after `SaveChanges` rolls back both. Then wrap it in an execution strategy and prove the retry works.

**Exercise 12 — Migration pipeline (2 hrs).** Set up CI that: fails on `has-pending-model-changes`; generates an idempotent script and a bundle as artifacts; applies migrations to a fresh Testcontainers database; and applies them again to prove idempotency. Add a job that diffs the migrated schema against a checked-in baseline to detect drift. Time the whole thing.

---

## Free resources

### Primary sources — the framework itself

| Resource | What it covers | Why read it |
|---|---|---|
| [dotnet/efcore](https://github.com/dotnet/efcore) | The whole ORM, source and issues | The definitive answer to "what does it actually do?"; the issue threads carry the design rationale |
| [src/EFCore/ChangeTracking](https://github.com/dotnet/efcore/tree/main/src/EFCore/ChangeTracking) | `StateManager`, `InternalEntityEntry`, snapshots, identity map | **Read `StateManager` once** and Part C stops being abstract |
| [src/EFCore/Query](https://github.com/dotnet/efcore/tree/main/src/EFCore/Query) | The provider-agnostic query pipeline, navigation expansion, the compiled query cache | Concepts 12–13 from the source |
| [src/EFCore.Relational/Query](https://github.com/dotnet/efcore/tree/main/src/EFCore.Relational/Query) | SQL expression trees, translation, the optimization passes | Where EF 11's join pruning and `CAST` stripping live |
| [src/EFCore.Relational/Migrations](https://github.com/dotnet/efcore/tree/main/src/EFCore.Relational/Migrations) | The migrator, the history repository, the database lock | Concepts 55–58 from the source |
| [dotnet/EntityFramework.Docs](https://github.com/dotnet/EntityFramework.Docs) | The docs, plus **runnable samples for every article** | Clone it; every release-note code block is a working project |
| [efcore issue #13617 — parameterized collections](https://github.com/dotnet/efcore/issues/13617) | The most-voted issue in the repo, and its three successive solutions | The best single case study of an ORM/database trade-off being reasoned about in public |

### Microsoft Learn — modelling

| Resource | What it covers |
|---|---|
| [Creating and configuring a model](https://learn.microsoft.com/en-us/ef/core/modeling/) | Conventions, annotations, fluent API (Concept 2) |
| [**Complex types**](https://learn.microsoft.com/en-us/ef/core/modeling/complex-types) · [Owned entity types](https://learn.microsoft.com/en-us/ef/core/modeling/owned-entities) | The value-vs-reference-semantics distinction that drives Concept 5 |
| [Relationships](https://learn.microsoft.com/en-us/ef/core/modeling/relationships) · [Cascade delete](https://learn.microsoft.com/en-us/ef/core/saving/cascade-delete) | Concept 8, including the four delete behaviours in full |
| [Inheritance](https://learn.microsoft.com/en-us/ef/core/modeling/inheritance) | TPH / TPT / TPC (Concept 9) |
| [Keys](https://learn.microsoft.com/en-us/ef/core/modeling/keys) · [Generated values](https://learn.microsoft.com/en-us/ef/core/modeling/generated-properties) · [Indexes](https://learn.microsoft.com/en-us/ef/core/modeling/indexes) | Concepts 6, 47 |
| [**Value conversions**](https://learn.microsoft.com/en-us/ef/core/modeling/value-conversions) · [**Value comparers**](https://learn.microsoft.com/en-us/ef/core/modeling/value-comparers) | Concept 7 — read the comparers page before you ship a converter |
| [Shadow and indexer properties](https://learn.microsoft.com/en-us/ef/core/modeling/shadow-properties) · [Backing fields](https://learn.microsoft.com/en-us/ef/core/modeling/backing-field) · [Entity constructors](https://learn.microsoft.com/en-us/ef/core/modeling/constructors) | The DDD mapping toolkit (Concept 68) |
| [Sequences](https://learn.microsoft.com/en-us/ef/core/modeling/sequences) · [Table splitting](https://learn.microsoft.com/en-us/ef/core/modeling/table-splitting) | Value generation and the pre-complex-type mapping technique |

### Microsoft Learn — querying

| Resource | What it covers |
|---|---|
| [Querying data](https://learn.microsoft.com/en-us/ef/core/querying/) | The map for Part B |
| [**Tracking vs. no-tracking**](https://learn.microsoft.com/en-us/ef/core/querying/tracking) | Concept 20, including identity resolution |
| [Loading related data](https://learn.microsoft.com/en-us/ef/core/querying/related-data/) · [Eager](https://learn.microsoft.com/en-us/ef/core/querying/related-data/eager) · [Explicit](https://learn.microsoft.com/en-us/ef/core/querying/related-data/explicit) · [Lazy](https://learn.microsoft.com/en-us/ef/core/querying/related-data/lazy) | Concept 17 |
| [**Single vs. split queries**](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries) | Concept 18 — the cartesian-explosion article, with the consistency caveat |
| [Global query filters](https://learn.microsoft.com/en-us/ef/core/querying/filters) | Concept 10, including EF 10's named filters |
| [SQL queries](https://learn.microsoft.com/en-us/ef/core/querying/sql-queries) | `FromSql`, `SqlQuery<T>`, composition rules, injection (Concept 22) |
| [Client vs. server evaluation](https://learn.microsoft.com/en-us/ef/core/querying/client-eval) | Concept 15, and why EF 3.0 made it throw |
| [Pagination](https://learn.microsoft.com/en-us/ef/core/querying/pagination) | Offset vs. keyset, from the team (Concept 23) |
| [Query tags](https://learn.microsoft.com/en-us/ef/core/querying/tags) · [Complex query operators](https://learn.microsoft.com/en-us/ef/core/querying/complex-query-operators) | `TagWith` (Concept 24); joins, `GroupJoin`, `SelectMany` |

### Microsoft Learn — change tracking and saving

| Resource | What it covers |
|---|---|
| [Change tracking overview](https://learn.microsoft.com/en-us/ef/core/change-tracking/) | Part C's official version |
| [**Change detection and notifications**](https://learn.microsoft.com/en-us/ef/core/change-tracking/change-detection) | Concepts 26–27, including the notification strategies |
| [Accessing tracked entities](https://learn.microsoft.com/en-us/ef/core/change-tracking/entity-entries) · [**Debug views**](https://learn.microsoft.com/en-us/ef/core/change-tracking/debug-views) | `EntityEntry`, and the `ChangeTracker.DebugView` you should memorize |
| [Explicitly tracking entities](https://learn.microsoft.com/en-us/ef/core/change-tracking/explicit-tracking) · [Identity resolution](https://learn.microsoft.com/en-us/ef/core/change-tracking/identity-resolution) | Concepts 25, 28–29 |
| [Changing foreign keys and navigations](https://learn.microsoft.com/en-us/ef/core/change-tracking/relationship-changes) | Relationship fixup, the part people find surprising |
| [Saving data](https://learn.microsoft.com/en-us/ef/core/saving/) · [Disconnected entities](https://learn.microsoft.com/en-us/ef/core/saving/disconnected-entities) | Concept 28's `Attach`/`Update`/`TrackGraph` |
| [**Concurrency conflicts**](https://learn.microsoft.com/en-us/ef/core/saving/concurrency) | Concept 33, with the resolution code |
| [Transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions) | Implicit, explicit, ambient, savepoints (Concept 32) |
| [`ExecuteUpdate` and `ExecuteDelete`](https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete) | Concept 34, including the change-tracker caveats |

### Microsoft Learn — `DbContext`, performance, diagnostics

| Resource | What it covers |
|---|---|
| [`DbContext` lifetime, configuration, and initialization](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/) | Concepts 37–40 — the official lifetime guidance |
| [Performance overview](https://learn.microsoft.com/en-us/ef/core/performance/) | The map for Part E |
| [**Efficient querying**](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying) · [**Efficient updating**](https://learn.microsoft.com/en-us/ef/core/performance/efficient-updating) | Concepts 16–20, 31, 34 — including the batch-size numbers |
| [Modeling for performance](https://learn.microsoft.com/en-us/ef/core/performance/modeling-for-performance) | Denormalization, caching columns, inheritance cost |
| [**Advanced performance topics**](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics) | Context pooling, compiled queries, the query cache, thread-safety checks (Concepts 13, 21, 39, 41) |
| [NativeAOT and precompiled queries (experimental)](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries) | Concept 21 — read the limitations section specifically |
| [Connection resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency) | Concept 43, including the execution-strategy/transaction rule |
| [Connection strings](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-strings) | Where pooling is actually configured (Concept 42) |
| [Logging, events, and diagnostics](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/) · [Simple logging](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/simple-logging) · [`Microsoft.Extensions.Logging`](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/extensions-logging) | Concept 24 |
| [**Interceptors**](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/interceptors) | Concept 35 — the outbox hook |
| [Metrics](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/metrics) · [Events](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/events) | The counters in Concept 46, including cache hit rate |

### Microsoft Learn — migrations and schema management

| Resource | What it covers |
|---|---|
| [Migrations overview](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/) | Concept 55 |
| [**Applying migrations**](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying) | Scripts, bundles, `Migrate()`, and **migration locking** (Concepts 57–58) |
| [Managing migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/managing) | `has-pending-model-changes`, squashing, resetting (Concept 63) |
| [**Migrations in team environments**](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/teams) | Concept 56 — snapshot merge conflicts |
| [Custom migration operations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/custom-operations) · [History table](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/history-table) | `migrationBuilder.Sql`, per-context history tables (Concepts 59, 61–62) |
| [Multiple providers](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/providers) · [Data seeding](https://learn.microsoft.com/en-us/ef/core/modeling/data-seeding) | Provider-specific migrations; `HasData` (Concept 60) |
| [`dotnet ef` reference](https://learn.microsoft.com/en-us/ef/core/cli/dotnet) · [PMC reference](https://learn.microsoft.com/en-us/ef/core/cli/powershell) | Every flag, including EF 11's config file and wildcards |

### Microsoft Learn — testing and providers

| Resource | What it covers |
|---|---|
| [**Choosing a testing strategy**](https://learn.microsoft.com/en-us/ef/core/testing/choosing-a-testing-strategy) | Concept 70 — read this before defending the InMemory provider |
| [Testing against the database](https://learn.microsoft.com/en-us/ef/core/testing/testing-with-the-database) · [Without the database](https://learn.microsoft.com/en-us/ef/core/testing/testing-without-the-database) | Both halves, honestly presented |
| [Database providers](https://learn.microsoft.com/en-us/ef/core/providers/) | Concept 71's capability matrix |
| [SQL Server provider](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/) · [**Vector search**](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/vector-search) · [Full-text search](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/full-text-search) · [Temporal tables](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/temporal-tables) | Concepts 52–53 |
| [Cosmos DB provider](https://learn.microsoft.com/en-us/ef/core/providers/cosmos/) | Partition keys, transactional batches, full-text and vector search |
| [SQLite limitations](https://learn.microsoft.com/en-us/ef/core/providers/sqlite/limitations) | Why SQLite is a partial test double — and the `__EFMigrationsLock` gotcha |
| [Npgsql EF Core provider docs](https://www.npgsql.org/efcore/) · [npgsql/efcore.pg](https://github.com/npgsql/efcore.pg) | The best-documented non-Microsoft provider; `xmin`, `jsonb`, arrays |

### Release notes and current state

| Resource | What it covers |
|---|---|
| [**What's new in EF Core 10**](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew) · [Breaking changes](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/breaking-changes) | The production-current LTS |
| [**What's new in EF Core 11**](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-11.0/whatsnew) · [Breaking changes](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-11.0/breaking-changes) | GA November 2026 — read the breaking changes before upgrading |
| [What's new in EF Core 9](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew) · [Breaking changes](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/breaking-changes) | Migration locking, `PendingModelChangesWarning`, the user-transaction warning |
| [What's new in EF Core 8](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-8.0/whatsnew) | Complex types, primitive collections, JSON columns, `HierarchyId` |
| [.NET and EF Core support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core) | LTS/STS dates — know when your version leaves support |
| [dotnet/core release notes](https://github.com/dotnet/core/tree/main/release-notes) | Per-preview EF Core notes; the fastest way to track a feature's arrival |

### Architecture guidance

| Resource | What it covers |
|---|---|
| [.NET microservices: DDD and CQRS patterns](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/) | Microsoft's own EF+DDD guidance (Concept 68) |
| [Infrastructure persistence layer design](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design) | Repository, unit of work, and the aggregate boundary as Microsoft frames it |
| [Architect modern web apps with ASP.NET Core and Azure](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/) | The eShop reference architecture and its data layer |
| [Data partitioning guidance](https://learn.microsoft.com/en-us/azure/architecture/best-practices/data-partitioning) | Ties Module 8 and Module 12 into the multi-tenancy decision |
| [Multitenant SaaS data patterns (Azure SQL)](https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns) | Concept 69's decision table, with operational cost |
| [Row-Level Security (SQL Server)](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security) | The defence-in-depth half of tenancy (Concepts 10, 69) |
| [Query Store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store) · [`pg_stat_statements`](https://www.postgresql.org/docs/current/pgstatstatements.html) | Step 3 of the diagnosis procedure (Concept 46) |

### Blogs, talks, and community

| Resource | What it covers |
|---|---|
| [**Shay Rojansky's blog**](https://www.roji.org/) | EF Core team member; the deepest public writing on EF's query pipeline, parameterization, and Npgsql internals |
| [Arthur Vickers' blog (One Unicorn)](https://blog.oneunicorn.com/) | EF team member; change tracking, identity resolution, and the *why* behind API decisions |
| [Jon P Smith (The Reformed Programmer)](https://www.thereformedprogrammer.net/) | Author of *EF Core in Action*; performance series, DDD-with-EF, soft delete, multi-tenancy |
| [Erik Ejlskov Jensen (ErikEJ)](https://erikej.github.io/) | Tooling, scaffolding, provider internals, migration tips |
| [EF Core Power Tools](https://github.com/ErikEJ/EFCorePowerTools) | Reverse engineering, model diagrams, migration and DbContext visualization |
| [Milan Jovanović — EF Core performance guide](https://www.milanjovanovic.tech/blog/ef-core-performance-guide) | A well-organized practical tour of Part E's material |
| [Code with Mukesh — running migrations in EF Core](https://codewithmukesh.com/blog/running-migrations-efcore/) · [bulk operations](https://codewithmukesh.com/blog/bulk-operations-efcore/) | Concepts 49, 57 with benchmarks and the licensing note |
| [Vladimir Khorikov — Enterprise Craftsmanship](https://enterprisecraftsmanship.com/) | Repository/unit-of-work critique, DDD persistence, testing strategy (Concepts 66–67, 70) |
| [Martin Fowler — Unit of Work](https://martinfowler.com/eaaCatalog/unitOfWork.html) · [Identity Map](https://martinfowler.com/eaaCatalog/identityMap.html) · [Repository](https://martinfowler.com/eaaCatalog/repository.html) · [Data Mapper](https://martinfowler.com/eaaCatalog/dataMapper.html) | The patterns `DbContext` *is* — read these before arguing about repositories |
| [Martin Fowler — Parallel Change](https://martinfowler.com/bliki/ParallelChange.html) | Expand/contract, named (Concept 59) |
| [Evolutionary Database Design](https://martinfowler.com/articles/evodb.html) | The original argument for migrations as a practice |
| [**Use The Index, Luke!**](https://use-the-index-luke.com/) · [No Offset](https://use-the-index-luke.com/no-offset) | The best free resource on indexing and on keyset pagination (Concepts 23, 47) |
| [David Fowler — Async Guidance](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md) | The async rules that Part B assumes |

### Libraries worth knowing (and their trade-offs)

| Resource | What it is | Judgement |
|---|---|---|
| [Dapper](https://github.com/DapperLib/Dapper) | Micro-ORM: SQL in, objects out | The read-side companion in Concept 65 |
| [EFCore.BulkExtensions](https://github.com/borisdj/EFCore.BulkExtensions) | `BulkInsert`/`Update`/`Delete`/`Merge` | Rung 3 of Concept 49 — **check the revenue-based licence before adopting** |
| [Ardalis.Specification](https://github.com/ardalis/Specification) | Specification pattern over EF | Concept 67 — useful when criteria genuinely repeat; overhead when they don't |
| [EntityFramework.Exceptions](https://github.com/Giorgi/EntityFramework.Exceptions) | Turns provider-specific `DbUpdateException`s into typed exceptions (unique constraint, FK, max length) | Removes a pile of error-number string matching |
| [Testcontainers for .NET](https://dotnet.testcontainers.org/) | Real databases in Docker for tests | Concept 70's recommendation |
| [Respawn](https://github.com/jbogard/Respawn) | Fast database reset between tests | The isolation half of a Testcontainers suite |
| [EFCore.NamingConventions](https://github.com/efcore/EFCore.NamingConventions) | snake_case and other naming conventions | Essential on PostgreSQL if you want idiomatic SQL |
| [Entity Framework Plus / Extensions](https://entityframework-extensions.net/) | Commercial bulk and batch extensions | Powerful; commercial licence — price it before you design around it |

### Books (not free, listed for completeness)

- **Entity Framework Core in Action** — Jon P Smith. The best single book on this module; the performance and DDD chapters are the reference.
- **Refactoring Databases: Evolutionary Database Design** — Ambler & Sadalage. Part F's theory, and where expand/contract comes from.
- **SQL Performance Explained** — Markus Winand. The paid companion to *Use The Index, Luke!*
- **Implementing Domain-Driven Design** — Vaughn Vernon. The aggregate-boundary argument behind Concept 68.
- **Designing Data-Intensive Applications** — Kleppmann. Still the distributed-systems companion to everything Phase 3 established.

---

## Quick-recall sheet

| If the interviewer asks about… | Lead with… |
|---|---|
| The three machines | Model (per process), query cache (by tree shape), change tracker (per context) |
| Model building | Conventions → annotations → fluent API; frozen and validated at build time |
| Model cache | Keyed by context type; per-tenant keys mean a model per tenant — memory, not just CPU |
| Compiled models | `dotnet ef dbcontext optimize`; removes model-build time, not query-compile time |
| Complex vs. owned types | Value semantics vs. reference semantics; EF 10 says prefer complex |
| Keys | `int` identity by default; random `Guid` clustered keys fragment; use UUIDv7 if you need client keys |
| Value converters | Mutable values need a `ValueComparer`, or changes are silently lost |
| Delete behaviour | Cascade = "part of"; EF only cascades what's *loaded* |
| Inheritance | TPH default (one table, discriminator); TPT joins; TPC unions and needs cross-table unique keys |
| Query filters | Every query on that type; named in EF 10; pair with RLS for real isolation |
| `IQueryable` | Composition is free; enumeration is the round trip; leaving `IQueryable` leaves the database |
| Query pipeline | Cache → preprocess → navigation expansion → translate → optimize → SQL + shaper → execute |
| Query cache | Keyed by tree shape; runtime values as `Expression.Constant` = cache-key explosion |
| Parameterization | Captured variables parameterize; EF 10 default is one padded parameter per collection element |
| Parameter ceiling | SQL Server 2,100 per command; PostgreSQL 65,535 |
| Client evaluation | Only in the final projection; everything else throws by design since EF 3.0 |
| Projection | The highest-leverage habit: less data, no tracking, fewer joins, a stable contract |
| Loading related data | Eager / explicit / lazy; lazy in a web app = invisible N+1 + sync I/O + disposal bugs |
| Cartesian explosion | Two collection `Include`s multiply rows; `AsSplitQuery` trades rows for round trips *and* consistency |
| N+1 | Lazy loading, a query in a loop, or navigation access without `Include`; count statements per request |
| Tracking modes | Tracking / `AsNoTracking` / `AsNoTrackingWithIdentityResolution`; it's a retention decision |
| Compiled queries | `EF.CompileAsyncQuery` skips the cache lookup; micro-optimization for hot complex queries |
| Precompiled / AOT | `--precompile-queries` helps startup; NativeAOT is **experimental**, no dynamic LINQ |
| Raw SQL | `FromSql` parameterizes every hole; `FromSqlRaw` is yours to sanitize; EF 10 analyzer warns on concatenation |
| Pagination | Offset degrades with depth and shifts under writes; keyset is constant-cost; always add a tiebreaker |
| Seeing the SQL | `ToQueryString()`, `TagWith()`, command logging, interceptors — then the database's plan store |
| State manager | Identity map keyed by PK; one instance per key; it is a GC root |
| `DetectChanges` | O(entities × properties), runs on most public APIs; `Add` in a loop is quadratic |
| Notification entities | `INotifyPropertyChanged`/proxies remove the scan; for long-lived contexts with big graphs |
| Disconnected graphs | `Update()` marks everything modified; prefer load-then-mutate; `TrackGraph` for control |
| Entity states | Added / Modified / Deleted / Unchanged / Detached; Modified is per property |
| `SaveChanges` | Detect → cascade → generate → topological sort → batch → transaction → propagate → accept |
| Batching | SQL Server min 4 / max 42; Npgsql 1,000; `INSERT ... OUTPUT`/`RETURNING` returns generated keys |
| Triggers | Break `OUTPUT`; tell EF with `HasTrigger` or saves fail |
| Transactions | One per `SaveChanges`; explicit for multi-call units; savepoints since EF 7 |
| `TransactionScope` | Default is `Serializable` and no async flow — pass `TransactionScopeAsyncFlowOption.Enabled` |
| Concurrency | `rowversion` / `xmin` / business column → `DbUpdateConcurrencyException`; resolution is a business decision |
| `ExecuteUpdate`/`Delete` | Immediate, set-based, invisible to the tracker — so no tokens, no interceptors, no events |
| Interceptors | Command, connection, transaction, save-changes, materialization; save-changes is the outbox hook |
| Domain events | In-transaction for local consistency; outbox for anything leaving the process |
| Context lifetime | As long as one unit of work; scoped is the consequence, not the principle |
| Background services | A scope per unit of work, from `IServiceScopeFactory` — not one for the worker's life |
| Context pooling | Default pool size 1024; resets EF state only — your fields survive into the next request |
| `IDbContextFactory` | Blazor, workers, parallel queries, anything without an ambient scope |
| Thread safety | One operation at a time; a missing `await` is the usual cause of the second-operation exception |
| Two pools | ADO.NET pools connections; EF pools contexts; connections open late and close early |
| Resiliency | `EnableRetryOnFailure` + user transaction ⇒ `strategy.ExecuteAsync`, and the block must be idempotent |
| Multiple contexts | One per bounded context; own schema, own history table, `--context` on every command |
| Cost model | Round trip and plan dominate; EF overhead is micro-to-low-milliseconds |
| Diagnosis order | Count statements → read the SQL → get the plan → check metrics → trace → benchmark |
| Metrics that matter | Compiled-query cache hit rate, active `DbContext` count, concurrency failures, retries |
| Indexing for EF | Lead with `TenantId`; `IncludeProperties` mirroring your projections; watch converter `CAST`s |
| Parameter sniffing | The flip side of parameterization; `EF.Constant` for skewed low-cardinality predicates |
| Bulk ladder | Batched `SaveChanges` → `ExecuteUpdate` → bulk library (check the licence) → `SqlBulkCopy` + staging |
| Streaming | `AsAsyncEnumerable` + projection + no-tracking; trades memory for connection-hold time |
| Caching | HybridCache over DTOs at the application layer; transparent 2nd-level caches hide staleness |
| JSON columns | For data always read together and rarely filtered; native `json` at compat 170 / `UseAzureSql` |
| Vectors | `SqlVector<float>` + `VECTOR_DISTANCE` (EF 10); `VectorSearch().WithApproximate()` (EF 11) |
| Startup cost | Model build + first query + first connection; warm up after `Build()`, before `Run()` |
| A migration | Up/Down class + model snapshot + a row in `__EFMigrationsHistory` |
| `migrations add` | Diffs model vs. **snapshot**, never the database — so drift is invisible by construction |
| Snapshot conflicts | Never hand-merge; regenerate. EF 11 records the last migration ID to force the conflict |
| Applying migrations | Idempotent script or bundle, separate pipeline step, separate DDL identity |
| Migration locking | EF 9+: `sp_getapplock` / `LOCK TABLE` / `__EFMigrationsLock`; user transactions now warn |
| Pending model changes | EF 9+ `Migrate()` throws; put `has-pending-model-changes` in CI |
| Expand/contract | Add nullable → write both → backfill → read new → constrain → stop writing old → drop |
| Backfills | Batched, restartable, throttled job — never in `Up()` |
| Rollback | Forward-only; `Down()` is for local iteration; invest in a tested restore instead |
| Where the ORM stops | EF for writes, Dapper/SQL for demanding reads, bulk tools for bulk, warehouse for reporting |
| Repository | No generic one; aggregate-shaped on writes; never return `IQueryable` |
| Specifications | Worth it when criteria genuinely repeat across call sites; otherwise ceremony |
| EF + DDD | Backing fields, private setters, complex types as value objects, reference other aggregates by ID |
| Multi-tenancy | Shared schema + leading `TenantId` + named filter + RLS + no tenant state on a pooled context |
| Testing | Testcontainers with migrations; InMemory proves nothing; assert on `ToQueryString()` too |
| Provider portability | Portability of skills, not of applications; a provider switch is a project |
| Anti-patterns | Generic repository, `IQueryable` leakage, lazy loading, `Update(graph)`, startup migration, InMemory tests |

---

## Progress

Module 19 complete — **Phase 4 is finished.** Modules 14–18 gave you the runtime, the concurrency model, the language, the measurement discipline, and the web framework. This module gave you the data layer, which is where most of those either pay off or fail: the change tracker is Module 14's retention problem with a friendly name, the query pipeline is Module 15's async rules under load, the model is Module 16's language features made load-bearing, Part E is Module 17's measurement loop applied, and Part D is Module 18's scoped lifetime with actual consequences.

This module closes the loops it was created to close:

- **Module 12's Concept 60** — pooling, retries, tracking, compiled queries, and "where the ORM stops" — is now the full story: Parts D and E for the mechanics, Concept 65 for the position.
- **Module 12's optimistic concurrency** (Concept 26) became Concept 33, with all three resolution strategies and the commutative-update fourth option.
- **Module 12's migration opinion** — idempotent scripts, not `Database.Migrate()` at startup — became Part F, with the full argument, the locking behaviour EF 9 added, and the expand/contract discipline.
- **Module 12's multi-tenancy design** (Concept 53) became Concept 69, with the model-cache cost of each option and the pooled-context leak that no design document mentions.
- **Module 12's JSON-column point** — that schema flexibility is often a reason to use a JSON column, not a document database — became Concept 52, with EF 10/11's native `json` support and JSON indexes.
- **Module 11's outbox and dual-write problem** got its .NET implementation in Concepts 35–36, as a `SaveChangesInterceptor` that makes the pattern impossible to forget.
- **Module 18's Part E** (scoped lifetimes, captive dependencies, disposal, `IServiceScopeFactory`) is now grounded in the object those rules exist to protect.

Threads left open on purpose:

- **Clean Architecture and layering** — where the composition root and the persistence seam belong, and whether "the domain must not know about EF" survives contact with a real schema — is **Module 20**.
- **Modular monolith vs. microservices**, and the schema-ownership rule from Concept 61 as a module boundary, is **Module 21**.
- **DDD tactical patterns** — aggregates, invariants, value objects, domain events — are **Module 22**, which takes Concept 68 and makes it the subject rather than a mapping exercise.
- **CQRS and MediatR**, and whether the read/write split of Concept 65 deserves a full framework, are **Module 23**.
- **Event sourcing**, the alternative to storing current state at all, is **Module 24**.
- **Polly** — and specifically how it composes with (and must not double up on) the execution strategy of Concept 43 — is **Module 25**.
- **Cosmos DB in depth** — partitioning, RU cost modelling, and consistency levels underneath the EF provider of Concept 71 — is **Module 27**.
- **Observability** — turning the metrics and traces of Concept 46 into SLOs and error budgets — is **Module 28**.

Next in the curriculum: **Phase 5 — .NET Architecture Patterns**, opening with **Module 20 — Clean Architecture and layering**: what it actually buys you, where teams misuse it, and how the persistence decisions you just made either survive the layering or quietly dictate it.
