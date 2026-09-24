# Module 16 — Modern C#
*Phase 4: .NET & C# Technical Mastery · Senior/Architect Interview Prep for .NET & C#*

## Orientation

Here is the sentence to carry through the whole module: **almost every modern C# feature is the compiler taking something you used to enforce by hand — in code review, in tests, in your head — and turning it into something it can generate or check for you; fluency is knowing, for each feature, what it lowers to, what it costs, which invariant it moves into the type system, and where that guarantee stops.**

That reframing matters because the naive picture of "modern C#" is a vocabulary list: records, pattern matching, primary constructors, the `field` keyword, extension members, and — as of this autumn — unions. A mid-level candidate can define each of them. A senior candidate can tell you that a record compares *every instance field*, including the hidden backing field behind a `field ??= …` cache, so equality now depends on whether someone read a property; that a primary-constructor parameter captured by a method *and* copied into a field is two copies of state that will drift; that `with` never runs your constructor, so validation you put there is bypassed; that nullable reference types are erased at runtime, so the promise ends at the deserializer; that adding a case to a closed hierarchy is precisely the breaking change you wanted inside a bounded context and precisely the one you must not ship across a service boundary; and that upgrading from C# 13 to C# 14 changed which overload `array.Reverse()` binds to without a single line of your source changing.

This module is lighter than Modules 14 and 15 by design — those were about what the runtime does to your code; this one is about the code you write, which is what the interviewer actually *sees*. It shows up in three places in a loop: the **coding round** (does your C# read like 2026 or 2016, and did you choose each construct for a reason?), the **code-review round** (can you spot the traps these features introduce?), and the **design round** (can you model a domain so that illegal states cannot be represented, and explain what that costs when the model evolves?). The last one is where C# 15's union types and closed hierarchies change the conversation more than any language release since generics.

**Current platform state (verified September 2026).** C# 14 shipped with .NET 10 in November 2025; .NET 10 is LTS, supported to November 2028. C# 15 ships with .NET 11 in November 2026. .NET 11 RC1 (8 September 2026) carries a go-live licence and makes C# 15 the default language version for `net11.0` projects, stabilizing collection expression arguments, union types, closed class hierarchies, labeled `break`/`continue`, and extension indexers (the RC1 notes also list non-virtual static interface members). The memory-safety redesign — the "unsafe evolution" — remains a preview behind a separate feature flag. .NET 11 is an STS release, now supported for 24 months. So in a late-2026 interview, C# 14 is *production-current* and C# 15 is *just landed*; knowing both, and being able to say which you would adopt when and why, is the signal.

This module has nine jobs:

1. **Give you a frame that outlives the feature list.** The lowering model (what does it compile to?) and a four-way taxonomy of features (ceremony, invariants, performance, compile-time metaprogramming) let you reason about features released after this module was written.
2. **Cover the construction and initialization surface properly.** `init`, `required`, the `field` keyword, primary constructors, and partial members — and the specific ways each one lets state leak or invariants go unenforced.
3. **Take records apart.** What the compiler generates, how equality really works, what `with` does and does not run, the collection-in-a-record bug, `record struct`, and where records belong versus where they quietly break things (EF Core entities, public APIs, cyclic graphs).
4. **Treat pattern matching as a language, not a syntax.** The full pattern catalogue, how the compiler turns it into a decision DAG, what exhaustiveness can and cannot prove, and why the discard arm is a design decision.
5. **Teach C# 15's closed sets from first principles.** Sum types, "make illegal states unrepresentable," closed hierarchies versus unions, what a union actually is at runtime, the expression problem, Result types versus exceptions, and the versioning consequence that is the entire point of the feature.
6. **Cover extension members (C# 14 and 15) at the level of lowering.** What extension blocks compile to, why migration is binary-compatible, what they cannot do, and the design judgement about when a property should not be an extension property.
7. **Make nullability a system, not a set of warnings.** Flow analysis, the attributes, where the compiler's promise ends, and how you adopt it across a large codebase.
8. **Connect the performance-shaped and metaprogramming features to their purpose.** Spans, ref safety, `allows ref struct`, first-class spans, collection expressions, generic math, interpolated string handlers, source generators, interceptors — all of them are the language moving work from runtime to compile time.
9. **Say what fluency actually signals, and how to show it without feature tourism** — in a live coding round, in a code review, and as the architect setting language policy for a fleet of services.

Eight framings to carry through:

1. **Features are lowerings.** Before you judge a feature, ask what it compiles to. Almost everything is a rewrite into C# you could have written by hand, which means its runtime cost is the cost of that hand-written code — and its *reader* cost is new.
2. **Every feature moves an invariant somewhere.** Ask where it moved *to* and where it stops being enforced. The answer is almost always "into the C# compiler," and the compiler is not present at runtime, in reflection, in your deserializer, or in another language.
3. **Guarantees are compile-time unless proven otherwise.** Nullable annotations, `init`, `required`, `closed`, and exhaustiveness are all compiler contracts. Treat them as strong hints inside your codebase and as unverified claims at every boundary.
4. **Value versus identity is the deepest modeling question in this module.** Records, record structs, value objects, strongly-typed IDs, and EF Core entities are all answers to "are two objects with the same data the same thing?"
5. **Closed versus open is the second deepest.** Whether adding a *case* or adding an *operation* should be cheap decides between pattern matching over a closed set and virtual dispatch over an open one. This is the expression problem, and it is a design-round question disguised as a syntax question.
6. **Syntax is cheap to write and expensive to read.** The audience for your code is the next engineer. Consistency across a codebase beats novelty in a single file, which is why the enforcement mechanism for style is `.editorconfig` and analyzers, not taste.
7. **Language version is an architectural dependency.** C# 15 means .NET 11 means an STS runtime; a library that multi-targets `netstandard2.0` cannot use half this module. The language choice is a platform choice.
8. **Fluency is judgement, not vocabulary.** Knowing when *not* to use a feature — no primary constructor on a type with invariants, no record for an entity, no extension property that does I/O, no union across a service boundary without a tolerant reader — is the strongest signal in this module.

| # | Concept | The one-line takeaway |
|---|---|---|
| 1 | C# is a lowering compiler | Almost every feature rewrites into older C#; its runtime cost is the cost of the code you'd have written |
| 2 | Four kinds of feature | Ceremony, invariants, performance, compile-time metaprogramming — each has a different adoption argument |
| 3 | Sugar vs runtime support | Most features need only the compiler (and sometimes a polyfillable attribute); a few need the runtime |
| 4 | The version triangle | SDK, TargetFramework, LangVersion; the default comes from the TFM, and overriding it is "unsupported" |
| 5 | The release map | C# 7 → 15 in one table; C# 15 implies .NET 11 implies STS |
| 6 | How features are born and die | csharplang, LDM notes, speclets, previews; `!!` was cut, unions took years — the *why* is in the notes |
| 7 | Auto-properties | A hidden field plus two methods; properties are the unit of binary compatibility, fields are not |
| 8 | `init` | Settable during initialization only; `modreq` protects it from old compilers; not deep or runtime immutability |
| 9 | `required` | Moves the "must be set" obligation to every construction site; `[SetsRequiredMembers]` is an unchecked promise |
| 10 | The `field` keyword | Semi-auto properties: the compiler's backing field, now reachable from a hand-written accessor |
| 11 | `field` in practice | Initializers bypass the setter; `field ??=` is a race; in a record the hidden field joins equality |
| 12 | Primary constructors | Parameters in scope for the whole type, captured into a hidden **mutable** field only if a member uses them |
| 13 | Primary constructors in practice | Right for dependency capture, wrong for invariant establishment; watch CS9124/CS9107 double storage |
| 14 | Partial members | Methods, properties (C# 13), constructors and events (C# 14): the human/generator contract |
| 15 | Identity vs value | Entities keep identity through change; values *are* their data; mixing them up is a whole bug class |
| 16 | `record class` generates | Equality contract, `Equals`, `GetHashCode`, `==`, `ToString`/`PrintMembers`, `<Clone>$`, copy ctor, `Deconstruct` |
| 17 | Record equality in depth | Every instance field participates; `EqualityContract` keeps base ≠ derived; seal records by default |
| 18 | `with` expressions | Clone + init setters; shallow; never runs the constructor — validate in `init` accessors, not the ctor |
| 19 | The collection-in-a-record bug | `List<T>` fields compare by reference; value equality silently becomes identity equality |
| 20 | `record struct` | Positional properties are **mutable** unless `readonly`; strongly-typed equality; `default` bypasses invariants |
| 21 | Where records don't belong | EF Core entities, aggregates, cyclic graphs, public types you'll evolve |
| 22 | Value objects and strongly-typed IDs | `readonly record struct OrderId(Guid Value)` — cheap to declare, not free to integrate |
| 23 | The pattern catalogue | A sub-language of about a dozen pattern forms, accumulated C# 7 → 11, extended again in C# 15 |
| 24 | `switch` expressions | Arms are tested in order, subsumed arms are errors, and the result must have a type |
| 25 | How patterns compile | A decision DAG: tests are shared and reordered, member reads may be reused — patterns assume purity |
| 26 | Property and positional patterns | Structural tests over members and `Deconstruct`; extended property patterns (C# 10) flatten nesting |
| 27 | List and slice patterns | `[first, .., last]` over anything countable and indexable, including spans and strings |
| 28 | Exhaustiveness | The compiler proves coverage of the input's value space; enums are never closed; failure is a `SwitchExpressionException` |
| 29 | The discard arm | `_ =>` buys silence today by giving up the compiler's help tomorrow |
| 30 | Null patterns | `is null`/`is not null` can't be hijacked by operator overloads; `is { }` tests non-null and binds |
| 31 | Sum types | C# had product types forever and sum types only by encoding; C# 15 closes a twenty-year gap |
| 32 | Make illegal states unrepresentable | Model each state as a type carrying exactly its data; parse, don't validate |
| 33 | `closed` classes (C# 15) | Direct descendants fixed to one assembly; implicitly abstract; not transitive; exhaustive switches |
| 34 | `union` types (C# 15) | A closed set of *existing* types; implicit conversions in; patterns unwrap `Value` |
| 35 | Unions at runtime | A struct holding one `object?`; value-type cases box; `default` is a null union |
| 36 | Custom unions | Basic pattern, non-boxing access pattern, class-based unions, union member providers |
| 37 | Choosing a closed-set shape | enum vs interface vs abstract base vs closed hierarchy vs union — decided by what varies and what you own |
| 38 | The expression problem | Virtual dispatch makes new types cheap; pattern matching over a closed set makes new operations cheap |
| 39 | Result types vs exceptions | Expected domain outcomes in the type; unexpected failures as exceptions; translate both at the edge |
| 40 | Versioning closed sets | Adding a case breaks every exhaustive switch — the feature inside a boundary, a hazard across one |
| 41 | Classic extension methods | A static call in disguise: no virtual dispatch, a null receiver is legal, instance members always win |
| 42 | Extension blocks (C# 14/15) | Instance and static methods, properties, operators, and (C# 15) indexers for types you don't own |
| 43 | How extension members lower | Ordinary static implementation methods plus metadata marker types; binary-compatible with `this` methods |
| 44 | What extensions can't do | No state, no virtuality, no interface implementation; ambiguity across packages is a build break |
| 45 | Extension design judgement | Property syntax promises cheap, pure, repeatable — an extension property that enumerates or does I/O lies |
| 46 | User-defined compound assignment (C# 14) | In-place `+=` for large mutable types; changes aliasing semantics, so only for builder-like types |
| 47 | Nullable reference types | Static analysis over an erased type system: `string?` and `string` are the same runtime type |
| 48 | Flow analysis and `!` | The analysis is deliberately optimistic about fields; `!` changes nothing at runtime |
| 49 | The nullable attributes | `NotNullWhen`, `MemberNotNull`, `DoesNotReturn` and friends teach the analysis your contracts |
| 50 | Where the promise ends | Deserializers, EF Core, configuration binding, reflection, interop — validate at the boundary |
| 51 | Adopting nullability; null-conditional assignment | Warnings-as-errors on new code, bottom-up annotation, `!` as a metric; `a?.B = c` (C# 14) |
| 52 | The ref-safety family | `readonly struct`, `ref struct`, ref fields, `scoped` — escape analysis lets the BCL be zero-copy *and* safe |
| 53 | `allows ref struct` (C# 13) | Generic code over spans; the payoff is alternate lookup — dictionary lookups by span without allocating |
| 54 | First-class spans (C# 14) | Built-in array/span conversions simplify the BCL — and changed overload binding on upgrade |
| 55 | `params` collections (C# 13) | `params ReadOnlySpan<T>` lets the compiler stack-allocate arguments; recompiling picks up the new overloads |
| 56 | Collection expressions | Target-typed construction the compiler optimizes per target; C# 15 adds `with(...)` arguments |
| 57 | Generic math and static abstracts | Static members in interfaces, specialized per value type; in app code mostly `IParsable<T>` and factories |
| 58 | Strings | Raw and UTF-8 literals; interpolated string handlers; and why `$"..."` in a log call is still wrong |
| 59 | Source generators | Moving reflection to compile time; incremental pipelines need value-equatable models |
| 60 | Interceptors | Stable since the .NET 9.0.2xx SDK; call-site substitution for generators; you consume them far more than write them |
| 61 | Analyzers as enforcement | `.editorconfig`, `AnalysisLevel`, banned APIs, architecture tests — policy that runs in CI |
| 62 | AOT and trimming | The reason the language keeps moving work to compile time |
| 63 | Program shape | Top-level statements, file-scoped namespaces, global/implicit usings, file-based apps |
| 64 | Small features worth recognizing | Target-typed `new`, `file` types, alias-any-type, `nameof(List<>)`, lambda upgrades, labeled `break`, ORPA |
| 65 | Interface evolution | Default interface methods for API versioning; static abstract/virtual members for generic contracts |
| 66 | What fluency signals | Currency, cost-awareness, and taste — in that order of how often they're tested |
| 67 | Feature tourism | Using a feature because it exists is the anti-signal; restraint is legible |
| 68 | Modern C# in a live round | Choose constructs that make intent visible and delete bug classes; narrate the choice |
| 69 | The code-review round | A checklist of the traps this module's features introduce |
| 70 | Language policy as an architect | SDK pinning, LangVersion defaults, nullable-as-errors, analyzers, and an upgrade cadence |

---

# Part A — How C# evolves

## Concept 1 — C# is a lowering compiler

The Roslyn compiler works in phases: parse the text into syntax trees, bind names to symbols and types, **lower** high-level constructs into simpler ones, and finally emit IL. Lowering is the phase that matters for this module. It rewrites almost every "modern" construct into a tree of older, simpler constructs before a single IL instruction is produced:

- `foreach` becomes `GetEnumerator()`/`MoveNext()`/`Current` inside a `try`/`finally` that disposes the enumerator.
- `using` becomes `try`/`finally` with a `Dispose` call; `await using` does the same with `DisposeAsync`.
- `lock` becomes `Monitor.Enter`/`Exit` — or, when the target is a `System.Threading.Lock`, `EnterScope()` and a `ref struct` disposal (Module 15, Concept 43).
- `async` methods become a struct state machine and a builder (Module 15, Concept 10); iterators become a state-machine class.
- Lambdas that capture become "display classes" holding the captured variables.
- String interpolation becomes `DefaultInterpolatedStringHandler` calls, `string.Concat`, or a constant.
- Pattern matching becomes a decision graph of type tests, comparisons, and member reads (Concept 25).
- A one-line `record` becomes a class with about a dozen synthesized members (Concept 16).

The IL instruction set has barely changed since generics arrived in .NET 2.0, and that is not an accident: the C# team deliberately prefers features that are pure compiler work, because they ship without a runtime change and they work on every runtime the compiler can target. The consequence is a rule you can apply to any feature, including ones that ship after this module: **a lowered feature costs, at runtime, exactly what the equivalent hand-written code would cost.** A switch expression is not slower than the `if` chain it replaces; a record's `Equals` is not faster than one you'd write carefully; primary constructors do not "save a field" — they either create one or they don't (Concept 12).

What lowering does *not* make free is reading. Every construct the compiler expands is one more thing the next engineer must mentally expand to predict behaviour. That is the real cost you are weighing when you adopt a feature, and it is why "what does this compile to?" is the first question in every section below.

The tool for answering it is **SharpLab** ([sharplab.io](https://sharplab.io/)): paste C#, switch the output to "C#" to see the lowered form, to "IL" to see what's emitted, or to "JIT Asm" to see what the JIT makes of it. ILSpy does the same for compiled assemblies, with an option to show compiler-generated members. In an interview, "I'd check the lowered form — I believe it becomes X" is a strong answer, because it shows you have a model of the compiler rather than a memory of blog posts.

A small demonstration of the habit:

```csharp
// What you write
var greeting = name is { Length: > 0 } n ? $"Hello, {n}" : "Hello";

// Roughly what the compiler produces
string greeting;
if (name != null && name.Length > 0)
{
    var handler = new DefaultInterpolatedStringHandler(7, 1);
    handler.AppendLiteral("Hello, ");
    handler.AppendFormatted(name);
    greeting = handler.ToStringAndClear();
}
else greeting = "Hello";
```

Nothing in the first line costs more than the second. The first line is also shorter, harder to get subtly wrong, and — for a team that reads patterns fluently — easier to read. That last clause is the one that needs a team decision (Concept 70).

---

## Concept 2 — Four kinds of modern feature

A long release-notes list becomes manageable once you notice that modern C# features fall into four categories, and that each category has a *different adoption argument*:

| Category | Examples | What it buys | The question to ask |
|---|---|---|---|
| **Ceremony removal** | File-scoped namespaces, global usings, top-level statements, target-typed `new()`, primary constructors, the `field` keyword, collection expressions, raw string literals | Less noise; the same program in fewer tokens | Does it read more clearly *to the whole team*, and is it applied consistently? |
| **Invariant encoding** | Nullable reference types, `required`, `init`, records' value equality, exhaustive `switch`, `closed` hierarchies, unions, `readonly struct`, `scoped` | Bugs moved from runtime (or code review) to compile time | Where exactly does the guarantee stop being enforced? |
| **Performance enablers** | `Span<T>`/`ref struct`, ref fields, `allows ref struct`, `params ReadOnlySpan<T>`, first-class span conversions, inline arrays, UTF-8 literals, interpolated string handlers, generic math, user-defined compound assignment | Zero-allocation, zero-copy code that is still verifiably safe | Is this path actually hot, and will a reader recognize the idiom? |
| **Compile-time metaprogramming** | Source generators, partial members, interceptors, `[GeneratedRegex]`, `System.Text.Json` source generation, `[LibraryImport]`, `[LoggerMessage]` | Reflection-free, trimming-safe, AOT-compatible code with no runtime discovery cost | Can the reader find and debug the generated code? |

Many features straddle categories — collection expressions remove ceremony *and* let the compiler pick an optimal construction strategy; records remove ceremony *and* encode value equality. But the categorization pays off in two concrete ways.

First, **it tells you how to argue for adoption.** Ceremony features are a matter of taste and consistency, so they belong in `.editorconfig` with a team decision behind them, and nobody should die on those hills. Invariant features are correctness features — nullable reference types with warnings-as-errors is worth mandating across a codebase, because it deletes a bug class. Performance features are local: you want them in the ten hot paths a profiler found, not sprinkled everywhere as a style. Metaprogramming features are infrastructure: someone owns the generator, and everyone else consumes it.

Second, **it tells you what to check in review.** For an invariant feature, the review question is "where does the promise end?" (the deserializer, the database, the other language). For a performance feature, it's "was this measured?" For a metaprogramming feature, it's "what happens when the generator doesn't run, or runs on stale input?"

An interviewer who asks "what's your favourite recent C# feature?" is usually fishing for exactly this: can you say *which kind* of feature it is and what trade it makes, rather than just what the syntax looks like?

---

## Concept 3 — Sugar vs runtime support

Most C# features are compiler-only. Some need the runtime, and the difference decides what you can use in a library that multi-targets, and what an upgrade actually involves.

**Features that need runtime support** (they cannot be polyfilled because the runtime itself must behave differently):

| Feature | C# | Runtime needed | Why |
|---|---|---|---|
| Default interface methods | 8 | .NET Core 3.0+ | The type loader must resolve interface method bodies |
| Covariant return types | 9 | .NET 5+ | Method overriding rules in the type system |
| Static abstract/virtual interface members | 11 | .NET 7+ | Constrained calls to static members, resolved per type argument |
| `ref` fields | 11 | .NET 7+ | The GC and type loader must understand byref-typed fields |
| Inline arrays | 12 | .NET 8+ | `[InlineArray]` changes type layout |
| `allows ref struct` | 13 | .NET 9+ | Generic instantiation over byref-like types |

**Features that need only a type to exist** — usually an attribute or marker the compiler emits or looks for — work on older runtimes if you define the type yourself, which is what the [PolySharp](https://github.com/Sergio0694/PolySharp) source generator automates: `init` (`IsExternalInit`), `required` (`RequiredMemberAttribute`, `CompilerFeatureRequiredAttribute`, `SetsRequiredMembersAttribute`), the nullable attributes, `[CallerArgumentExpression]`, `[ModuleInitializer]`, interpolated-string-handler attributes, `[CollectionBuilder]`, `[OverloadResolutionPriority]`, and the C# 15 union support types (`UnionAttribute` and `IUnion`, which ship in the .NET 11 BCL).

**Features that need a BCL API to be useful** fall in between: `Span<T>` (the `System.Memory` package on `netstandard2.0`), `Index`/`Range`, the `System.Threading.Lock` type (.NET 9), and `SwitchExpressionException` (the compiler falls back to `InvalidOperationException` where it's absent).

Two mechanisms in this list are worth knowing in detail because they show how the compiler protects a contract from *older compilers*:

- An `init` accessor is emitted as a setter whose return type carries a `modreq(IsExternalInit)`. A required modifier is something a compiler must understand to call the method, so a C# 8 compiler — or a language that doesn't know the convention — refuses to call it, instead of silently treating it as a normal setter and breaking the "only during initialization" rule.
- A type with `required` members has its constructors marked with `[CompilerFeatureRequired("RequiredMembers")]` and an error-level `[Obsolete]` carrying a message about unsupported compilers. An old compiler that doesn't understand required members therefore cannot construct the type at all, rather than constructing it without setting the members.

Both are the same design move: when a feature's guarantee depends on every caller being checked, the compiler makes sure callers that *can't* be checked fail loudly.

Why this matters in practice: a shared library targeting `netstandard2.0` and `net10.0` can use records, `init`, `required`, nullable annotations, and pattern matching everywhere (with polyfills), but default interface methods and static abstract members only under `#if NET` guards. And in an interview, "can you use records on .NET Framework 4.8?" has a precise answer: yes, with `LangVersion` raised and `IsExternalInit` polyfilled — an unsupported configuration that works in practice — whereas default interface methods cannot work there at all.

---

## Concept 4 — The version triangle: SDK, TargetFramework, LangVersion

Three separate versions govern what your code means:

- **The SDK** is the compiler you actually run. It is chosen by `global.json` (if present) or by whatever is installed.
- **The TargetFramework** (`net10.0`, `net11.0`, `netstandard2.0`, `net48`) is the BCL surface and runtime you compile against.
- **LangVersion** is the set of language rules the compiler applies.

By default, LangVersion is *derived from the target framework*: `net8.0` → C# 12, `net9.0` → C# 13, `net10.0` → C# 14, `net11.0` → C# 15, and `netstandard2.0` or any .NET Framework target → C# 7.3. Setting `<LangVersion>` higher than the default for your target is officially **unsupported**. It often compiles — the compiler doesn't refuse — but you are now outside the tested matrix: some features will fail for want of a runtime type, some will lower to code paths the older runtime handles differently, and some library behaviour (like the span overloads in Concept 54) was only designed to be correct on the matching BCL. The C# 14 breaking-change notes include exactly such a case: C# 14 against a pre-.NET 10 target changes what `array.Reverse()` binds to, and the mitigation overload only exists in the .NET 10 BCL.

The other values: `latest` (newest released version the SDK supports), `latestMajor`, `preview` (enables preview language features, sometimes alongside `<Features>` flags — the C# 15 memory-safety preview needs both `<LangVersion>preview</LangVersion>` and `<Features>$(Features);updated-memory-safety-rules</Features>`), and `default`.

Three practical rules, all of which surface in architect conversations:

1. **Don't set `LangVersion` unless you have a reason.** Setting it to `latest` in `Directory.Build.props` means your build's language rules change whenever someone installs a newer SDK — the opposite of reproducible.
2. **Pin the SDK with `global.json`** (with a `rollForward` policy such as `latestFeature`) so every developer and CI agent runs the same compiler. A new compiler with the same LangVersion is *mostly* the same language, but compiler releases carry their own breaking-change notes and new warnings.
3. **Treat a TFM upgrade as a language upgrade.** Moving a service from `net10.0` to `net11.0` silently moves it from C# 14 to C# 15. Most of the time that's harmless. Occasionally, it changes binding (Concept 54) or makes a word a keyword (`field` in C# 14, `extension` as a type name, `closed` and `union` as contextual keywords in C# 15). The upgrade PR should be treated as behavioural, with a full test pass, not as a version-string bump.

---

## Concept 5 — The release map: C# 7 → 15

You don't need every feature memorized, but you should be able to place the important ones in time, because "when did this arrive?" is really "which runtime does this require?"

| C# | Ships with | Year | The features that matter in interviews |
|---|---|---|---|
| 7.0–7.3 | .NET Core 2.x / .NET Framework 4.7 | 2017–18 | Tuples and deconstruction, type patterns in `is`/`switch`, local functions, `out var`, ref locals/returns, `in` parameters, `readonly struct`, `ref struct` and `Span<T>` support, `private protected`, `default` literal |
| 8 | .NET Core 3.x | 2019 | Nullable reference types, switch expressions, property/positional/tuple patterns, async streams, indices and ranges, default interface methods, `using` declarations, `??=`, readonly members, static local functions |
| 9 | .NET 5 | 2020 | Records, `init`, top-level statements, relational and logical patterns (`and`/`or`/`not`), target-typed `new()`, covariant returns, function pointers, native-sized integers, module initializers, static lambdas |
| 10 | .NET 6 (LTS) | 2021 | Record structs, global usings, file-scoped namespaces, extended property patterns, interpolated string handlers, lambda natural types, `[CallerArgumentExpression]`, parameterless struct constructors |
| 11 | .NET 7 | 2022 | Raw string literals, `required` members, list patterns, generic math (static abstract members), ref fields and `scoped`, `file`-local types, UTF-8 literals, generic attributes, auto-default structs, `>>>` |
| 12 | .NET 8 (LTS) | 2023 | Primary constructors for all classes and structs, collection expressions, inline arrays, default lambda parameters, alias-any-type, `ref readonly` parameters, `[Experimental]`, interceptors (experimental) |
| 13 | .NET 9 | 2024 | `params` collections, `System.Threading.Lock` recognition, partial properties and indexers, `allows ref struct`, ref struct interfaces, ref locals and ref structs in iterators/async, `\e`, `^` in object initializers, `[OverloadResolutionPriority]`, `field` (preview) |
| 14 | .NET 10 (LTS) | Nov 2025 | Extension members, the `field` keyword, null-conditional assignment, first-class span conversions, `nameof` with unbound generics, modifiers on untyped lambda parameters, partial constructors and events, user-defined compound assignment, file-based app directives |
| 15 | .NET 11 (STS) | Nov 2026 (RC1 Sept 2026) | Union types, closed hierarchies, collection expression arguments, extension indexers, labeled `break`/`continue`; memory-safety redesign in preview |

Two patterns in this table are worth saying out loud in an interview.

**The cadence is annual and coupled.** A new C# ships every November with a new .NET, and since C# 10 the language version has been the .NET version plus four. Even-numbered .NET releases are LTS (three years); odd-numbered releases are STS, which Microsoft extended from 18 to 24 months starting with .NET 9. That extension has a consequence architects should notice: .NET 10 (LTS, released November 2025) and .NET 11 (STS, releasing November 2026) now reach end of support at roughly the same time, in November 2028 — check the support policy page for exact dates. The traditional "LTS-only" argument against adopting an odd-numbered release, and therefore against C# 15's unions, is much weaker than it used to be.

**The themes are visible.** C# 7–8 made value types and spans first-class (the performance arc). C# 8–11 made nullability and immutability expressible (the correctness arc). C# 9–15 made data-oriented modeling first-class: records, then patterns, then list patterns, then closed hierarchies and unions (the modeling arc). C# 12–14 moved work to compile time for Native AOT (the metaprogramming arc). If you can narrate those four arcs in thirty seconds, you've demonstrated that you understand the *direction* of the language, which is more valuable than any individual feature.

---

## Concept 6 — How features are born and die

C# is designed in public. The pieces:

- **[dotnet/csharplang](https://github.com/dotnet/csharplang)** holds proposals, "champion" issues (a proposal a language designer has agreed to push through design), and discussions.
- **Language Design Meeting (LDM) notes** are published in the `meetings/` folder of that repo. They record not just decisions but the alternatives considered and why they lost.
- **Feature specifications ("speclets")** are published on Microsoft Learn under the language reference's *proposals* section. They are the design document for each feature, including breaking-change analysis and open questions.
- **Roslyn's [Language Feature Status](https://github.com/dotnet/roslyn/blob/main/docs/Language%20Feature%20Status.md)** page tracks which features are merged into which compiler branch.
- **Previews** ship behind `<LangVersion>preview</LangVersion>` — sometimes for a full release cycle, as the `field` keyword did in C# 13 before shipping in C# 14, precisely because it was a potential breaking change.
- **The ECMA-334 standard** trails the implementation by several versions; the speclets are the de facto specification for recent features.

Features also die or get reshaped, and the reasons are instructive:

- **`!!` parameter null-checking** appeared in C# 11 previews and was removed after strong community pushback about a terse, easily-missed syntax for runtime checks. `ArgumentNullException.ThrowIfNull(x)` — which uses `[CallerArgumentExpression]` to capture the parameter name — is what survived.
- **"Extension everything" / roles** was designed for years with the ambition of letting a type *implement an interface* through an extension. What shipped in C# 14 is extension members without that type-like capability (Concept 44).
- **Discriminated unions** have been discussed since at least the C# 7 era. What shipped in C# 15 is two complementary features — `union` types and `closed` hierarchies — plus a pattern-based design for custom unions, with closed enums and dictionary expressions announced as planned follow-ons.
- **Primary constructors for classes** were planned for C# 6, pulled, and shipped in C# 12 with different semantics.

Why this is a signal: when an interviewer asks "why aren't primary-constructor parameters `readonly`?" or "why do unions box value types?", the answers live in these documents. For unions, the reference documentation is explicit that the generated form is opinionated — always a struct, always storing `object?`, always boxing value-type cases — and that the *pattern-based* custom-union design exists precisely so you can choose a different representation when that matters (Concept 36). A candidate who has read one or two speclets and a few LDM notes sounds like someone who understands the language as a set of trade-offs made by people, rather than as a fixed artifact.

---

# Part B — Properties, construction, and initialization

## Concept 7 — Auto-properties: a hidden field and two methods

Start with the thing everything in this Part is built on. An auto-property:

```csharp
public string Name { get; set; }
```

lowers to a private field (named something unspeakable like `<Name>k__BackingField`), a `get_Name()` method, a `set_Name(string)` method, and property metadata tying them together. A getter-only auto-property (`{ get; }`) gets a `readonly` backing field that may be assigned only in a constructor or initializer. A property initializer (`= "x";`) assigns the backing field directly, as part of field initialization, before the base constructor runs.

Two consequences matter more than the syntax:

**Properties are the unit of binary compatibility; fields are not.** Callers compiled against a property call `get_Name()`. Callers compiled against a public field read the field directly with `ldfld`. Changing a public field into a property is therefore a *binary* breaking change — every consumer must recompile — whereas changing an auto-property into a full property with logic is not. That, not style, is why public fields are wrong in anything shipped as a library. The JIT inlines trivial accessors, so there is no runtime cost to the rule.

**The backing field is real, and it participates in everything a field participates in**: struct layout and size (Module 14, Concept 13), record equality (Concept 17), serialization by field-based serializers, and — with `[field: SomeAttribute]` — attribute targets. Keep that in mind; it's the root of two traps later in this Part.

---

## Concept 8 — `init`: initialization-time mutability, not immutability

C# 9 added a third accessor, `init`:

```csharp
public sealed class Endpoint
{
    public string Host { get; init; } = "localhost";
    public int Port { get; init; } = 443;
}

var e = new Endpoint { Host = "api.example.com" }; // OK: object initializer
// var f = e with { Port = 8443 };                  // error here: `with` needs a record, a struct, or an anonymous type (Concept 18)
// e.Port = 80;                                     // CS8852: init-only property can only be assigned in an object initializer...
```

An `init` accessor may be called from an object initializer, from a `with` expression, from a constructor of the type, and from another `init` accessor. After that, the compiler refuses assignment. Mechanically it is an ordinary setter protected by a `modreq(IsExternalInit)` so that compilers which don't understand the rule can't call it (Concept 3).

Be precise about what `init` is *not*:

- **It is not runtime immutability.** Reflection can call the setter. So can `System.Text.Json`, which is how init-only DTOs deserialize. So can `Unsafe` tricks. The rule is enforced by the C# compiler at call sites it compiles, and nowhere else.
- **It is not deep immutability.** `public List<string> Tags { get; init; }` prevents replacing the list; it does nothing to stop `Tags.Add(...)`. If the invariant is "this collection doesn't change," the type must say so: `IReadOnlyList<T>` (a promise not to mutate *through this reference*), `ImmutableArray<T>` or `FrozenSet<T>` (a promise that nobody can).
- **It is not a validation hook by default.** An auto-implemented `init` doesn't check anything. With the `field` keyword (Concept 10) it can — and that turns out to be the correct place to validate records (Concept 18).

The one-sentence version for an interview: *`init` controls who may set a property and when; it says nothing about deep immutability or runtime enforcement.*

---

## Concept 9 — `required`: moving the obligation to the construction site

Before C# 11, nullable reference types created an awkward gap. A DTO like this:

```csharp
public sealed class CreateOrder
{
    public string CustomerId { get; init; }   // CS8618: non-nullable property must contain a non-null value when exiting constructor
}
```

warns, because nothing guarantees `CustomerId` is set. The options were a constructor (verbose for DTOs, awkward with serializers) or `= null!` (a lie the compiler believes). C# 11's `required` closes the gap by moving the obligation from the type's constructor to *every construction site*:

```csharp
public sealed class CreateOrder
{
    public required CustomerId Customer { get; init; }
    public required IReadOnlyList<OrderLine> Lines { get; init; }
    public string? Notes { get; init; }
}

var ok  = new CreateOrder { Customer = id, Lines = lines };
var bad = new CreateOrder { Customer = id };   // CS9035: required member 'CreateOrder.Lines' must be set...
```

Mechanics worth knowing:

- The members and the type carry `RequiredMemberAttribute`; constructors carry `[CompilerFeatureRequired("RequiredMembers")]` plus an error-level `[Obsolete]` so that older compilers can't construct the type and bypass the check (Concept 3).
- A constructor marked `[SetsRequiredMembers]` tells the compiler "calling me satisfies all required members." **The compiler does not verify that claim.** It is a promise, and a wrong promise re-opens exactly the hole `required` closed. Use it on constructors that genuinely assign everything, and nowhere else.
- A type with required members can't satisfy a `new()` generic constraint, because `new T()` has no way to set them.
- `System.Text.Json` (since .NET 7) treats `required` members as required during deserialization and throws if they're absent from the payload — so `required` is also a wire-contract statement.

Two design consequences. First, **`required` is a public-API commitment**: adding `required` to an existing member of a published type breaks every caller's source; removing it is safe. Second, **`required` says "must be set," not "must be valid."** It is not a substitute for a constructor or factory that establishes invariants — it's the right tool for data carriers (DTOs, options, messages), and the wrong one for domain objects whose fields must satisfy rules together.

---

## Concept 10 — The `field` keyword (C# 14): semi-auto properties

For twenty years, the moment a property needed one line of logic — a null check, a trim, a change notification — you had to abandon the auto-property, declare a backing field by hand, and write both accessors. The field then leaked into the whole type's scope, where other members could bypass the property. C# 14 fixes this with a contextual keyword:

```csharp
public sealed class Customer
{
    // Validation in the setter; auto-implemented getter
    public string Email
    {
        get;
        set => field = value?.Trim().ToLowerInvariant()
                       ?? throw new ArgumentNullException(nameof(value));
    }

    // Change notification
    public string DisplayName
    {
        get;
        set
        {
            if (field == value) return;
            field = value;
            OnPropertyChanged();
        }
    }

    // Lazy initialization
    public string Pattern { get; init; } = "^.*$";
    public Regex Matcher => field ??= new Regex(Pattern, RegexOptions.Compiled);
}
```

The vocabulary from the specification: a **field-backed property** is either an auto-property or a property that uses `field` in an accessor; the **backing field** is the compiler-synthesized field that `field` denotes. You may now mix an auto accessor (`get;`) with a full accessor (`set => …`), and `field` works in getters, setters, `init` accessors, and expression-bodied properties. It does not work in indexers or event accessors.

Lowering is unremarkable, which is the point: the compiler emits exactly the private field and accessor bodies you would have written, except that the field is unnamed and therefore unreachable from the rest of the type. That encapsulation — only the accessors can touch the storage — is the real improvement over hand-written backing fields, not the saved line. `[field: Attribute]` targets the synthesized field, as it always did for auto-properties.

`field` was a preview in C# 13 and became a full feature in C# 14; the delay existed because it's a potential breaking change (Concept 11).

---

## Concept 11 — `field` in practice: four traps

The feature is small; the traps are where the senior signal lives.

**1. A property initializer bypasses your setter.** The specification is explicit: an initializer on a field-backed property initializes the backing field directly; it does *not* call the setter. That's unavoidable (initializers run before the base constructor, when calling instance methods isn't allowed), and sometimes useful (initializing a view-model property without raising a change event). But it means:

```csharp
public int Hours { get; set => field = value >= 0 ? value : throw new ArgumentOutOfRangeException(nameof(value)); } = -5;
// Compiles, runs, and Hours is -5: the initializer never went through the setter.
```

Assignment *in a constructor*, by contrast, calls the setter. If the setter carries an invariant, initialize in the constructor.

**2. `field ??= Create()` is not thread-safe.** Two threads can both observe `null` and both run `Create()`. For reference types the final write is atomic, so you won't see a torn value — but you'll get duplicate construction, duplicate side effects, and a brief window where two callers hold different instances. That's fine for an idempotent, cheap factory; it's wrong for anything expensive, stateful, or identity-sensitive. Use `Lazy<T>` or `LazyInitializer.EnsureInitialized` (Module 15, Concept 39).

**3. In a record, the hidden field joins equality — and `with` copies stale caches.** Records compare every instance field (Concept 17), and a `field`-backed cache is an instance field:

```csharp
public sealed record Person(string First, string Last)
{
    public string Display => field ??= $"{Last}, {First}";   // don't do this in a record
}

var a = new Person("Ada", "Lovelace");
var b = new Person("Ada", "Lovelace");
_ = a.Display;
Console.WriteLine(a == b);                    // False: a's hidden cache is populated, b's is null

var c = a with { First = "Augusta" };
Console.WriteLine(c.Display);                 // "Lovelace, Ada" — the clone copied the stale cache
```

Equality now depends on whether someone read a property, and `with` produces an object that lies about itself. In a record, compute derived values on demand, or cache them outside the record.

**4. Existing code named `field` changes meaning.** Inside property accessors, `field` used as a primary expression now binds to the keyword. A class with an actual member called `field` that its accessors read unqualified will silently change behaviour on upgrade; the compiler reports a warning where the binding changed, and you fix it with `this.field`, `@field`, or a rename. A local variable named `field` inside an accessor is now an error. This is the reason the feature spent a release in preview.

A fifth point that is a relief rather than a trap: **nullability just works for lazy properties.** The compiler infers the backing field's nullability from the getter — if the getter is "null-resilient" (analysing it with a maybe-null `field` produces no warnings, as with `field ??= …`), the backing field is treated as nullable — so a non-nullable lazy property doesn't trigger CS8618 in the constructor.

And the judgement call: `field` is for *one* property with *one* piece of storage and a small amount of logic. When two properties share an invariant, or a property's logic needs other state, write explicit fields and methods — hiding coupled state inside individual accessors makes the coupling harder to see.

---

## Concept 12 — Primary constructors (C# 12): parameters, not properties

Records have had primary constructors since C# 9. C# 12 extended the syntax to every class and struct — with deliberately *different* semantics, which is where confusion starts:

```csharp
public sealed class OrderService(IOrderRepository repository, TimeProvider clock)
{
    public async Task<Order> PlaceAsync(Cart cart, CancellationToken ct)
    {
        var order = Order.From(cart, clock.GetUtcNow());
        await repository.AddAsync(order, ct);
        return order;
    }
}
```

The rules:

- The parameters are **in scope throughout the type body** — in field and property initializers, and in member bodies.
- If a parameter is used only in initializers, it's an ordinary constructor parameter and nothing is stored.
- If a parameter is used in a member body, the compiler **captures** it into a private, compiler-named field (something like `<repository>P`) and rewrites every use to read that field.
- Captured parameters are **mutable**. A method can write `repository = null;` and every other member sees the change. There is no `readonly` modifier for primary-constructor parameters.
- They are **not properties** and not members: you can't write `this.repository`, and nothing is visible outside the type.
- Every other constructor must chain to the primary one with `: this(...)`.
- There is no constructor body. Validation must happen in initializers.

**Records are different.** A record's positional parameters become public properties (`{ get; init; }` for `record class` and `readonly record struct`, `{ get; set; }` for plain `record struct`), plus a `Deconstruct` method and participation in equality. Same syntax, a different contract — "primary constructor" means "public data shape" in a record and "private dependency capture" in a class.

What it lowers to, roughly:

```csharp
public sealed class OrderService
{
    private IOrderRepository <repository>P;   // not readonly
    private TimeProvider <clock>P;

    public OrderService(IOrderRepository repository, TimeProvider clock)
    {
        <repository>P = repository;
        <clock>P = clock;
    }
    // ... members read <repository>P and <clock>P
}
```

The runtime cost is identical to hand-written `private readonly` fields, minus the `readonly` — which, for reference-typed dependencies, has no measurable performance meaning and only a correctness one.

---

## Concept 13 — Primary constructors in practice

**Where they're right: dependency capture.** A DI-constructed service with three dependencies and no invariants of its own is the ideal case. The noise removed is real (four lines per dependency in the old form), and the mutability risk is small: reassigning a dependency inside a method is rare, easy to spot in review, and flaggable by analyzers. Many teams adopt primary constructors for services and enforce it with the `IDE0290` ("use primary constructor") style rule; others ban them because of the `readonly` gap. Either is defensible. What isn't defensible is a codebase that does both at random.

**Where they're wrong: invariant establishment.** A domain type whose constructor must check that `start <= end`, normalize a currency code, or reject an empty line list needs a constructor *body* or a factory method. Validation squeezed into initializers is possible —

```csharp
public sealed class DateRange(DateOnly start, DateOnly end)
{
    public DateOnly Start { get; } = start <= end ? start : throw new ArgumentException("start > end");
    public DateOnly End { get; } = end;
}
```

— but it's fragile (every new member must remember not to read the raw parameters) and it reads worse than the constructor it replaced. Primary constructors are for *capturing* values, not for *establishing* them.

**The double-storage bug.** This is the one to catch in code review:

```csharp
public class RetryBudget(int maxAttempts)
{
    private int _remaining = maxAttempts;            // copies the parameter into a field...

    public bool TryConsume() => _remaining-- > 0;
    public int Max => maxAttempts;                   // ...and ALSO captures the parameter
    public void Reset() => maxAttempts = _remaining; // now two pieces of state, drifting apart
}
```

The compiler warns — **CS9124**: the parameter is captured into the state of the enclosing type and its value is also used to initialize a field, property, or event. The sibling warning **CS9107** fires when a parameter is both captured *and* passed to a base constructor, which means the base class stores one copy and the derived class another; mutate either and they diverge. Treat both as errors.

**Structs.** A primary constructor on a struct is subject to the usual struct rule: `default(S)` and array elements bypass *every* constructor, so a captured parameter can be zero/null regardless of what the constructor would have done.

**The team decision.** Pick one policy — "primary constructors for services and DTO-like classes, explicit constructors for domain types," say — write it into `.editorconfig` severities and the contributing guide, and move on. The worst outcome is re-litigating it in every pull request.

---

## Concept 14 — Partial members: the contract between you and a generator

`partial` used to mean "this type's declaration is split across files." It now also means "this *member* is declared here and implemented elsewhere," and the "elsewhere" is almost always a source generator (Concept 59).

- **Partial methods** (C# 3, extended in C# 9). The original form — `void`, implicitly private, optional implementation, and calls removed entirely if nothing implements it — still exists. The C# 9 form allows any return type, `out` parameters, and accessibility modifiers, but then an implementation is required.
- **Partial properties and indexers** (C# 13). The *declaring* declaration looks like an auto-property; the *implementing* declaration has accessor bodies (and, with C# 14, can use `field`).
- **Partial constructors and events** (C# 14). Exactly one defining and one implementing declaration. Only the implementing constructor may have a `this(...)`/`base(...)` initializer, and only one part of the type may carry the primary-constructor syntax. The implementing partial event must have `add` and `remove` accessors.

The pattern is always the same: you declare the *shape* and attach an attribute; the generator reads the shape and writes the body.

```csharp
public static partial class Validators
{
    [GeneratedRegex(@"^[A-Z]{3}$", RegexOptions.CultureInvariant)]
    private static partial Regex CurrencyCode { get; }          // .NET 9+ supports properties here
}

public static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} placed for {Amount}")]
    public static partial void OrderPlaced(ILogger logger, OrderId orderId, decimal amount);
}

public partial class ProductViewModel : ObservableObject        // CommunityToolkit.Mvvm
{
    [ObservableProperty]
    public partial string Name { get; set; }                    // generator implements the INPC plumbing
}
```

Why partial properties and constructors mattered enough to add: before them, generators had to invent members next to yours (`[ObservableProperty] private string _name;` generating a `Name` property), which confused navigation, documentation, and nullable analysis. With partial members, *your* declaration is the public surface, and the generator only fills in the body. That keeps the API you review identical to the API you ship — a small change that makes generated code far less surprising to read.

---

# Part C — Records and value semantics

## Concept 15 — Identity vs value: the question underneath records

Before any syntax: two objects contain the same data. Are they the same thing?

- **Entities have identity.** A customer who changes their name and address is still the same customer. Equality is "same identity" — usually a key — and the data can change freely. Orders, accounts, users, aggregates.
- **Values are their data.** `Money(10, "EUR")` equals any other `Money(10, "EUR")`. There's no "which one." An address, a date range, a coordinate, a currency code, a message, a cache key. Values are naturally immutable: "changing" a value means making a different value.

Getting this wrong in either direction is a bug class:

- **Identity where value was meant**: deduplication that doesn't dedupe, `Dictionary` lookups that miss because the key is a fresh instance, test assertions that compare references, cache keys that never hit.
- **Value where identity was meant**: two distinct customers with the same name collapsing into one `HashSet` entry, an ORM treating two tracked entities as the same object, "equal" aggregates that are actually different business objects.

C#'s defaults split the difference awkwardly. Classes get reference equality. Structs get field-wise value equality from `ValueType.Equals` — which, when the fields aren't bitwise-comparable, uses reflection and boxes (Module 14, Concept 17), so it's correct but slow. For twenty years, a correct value type as a class meant hand-writing `Equals`, `GetHashCode`, `IEquatable<T>`, `==`, and `!=`, keeping them in sync as fields were added, and getting the inheritance case subtly wrong. Records exist to make value semantics cheap to *declare* and correct by *construction*.

---

## Concept 16 — What `record class` generates

```csharp
public record Money(decimal Amount, string Currency);
```

is shorthand for `record class` and produces, approximately:

```csharp
public class Money : IEquatable<Money>
{
    public Money(decimal Amount, string Currency) { this.Amount = Amount; this.Currency = Currency; }
    protected Money(Money original)                          // copy constructor, used by `with`
    { Amount = original.Amount; Currency = original.Currency; }

    public decimal Amount   { get; init; }
    public string  Currency { get; init; }

    protected virtual Type EqualityContract => typeof(Money);

    public virtual bool Equals(Money? other) =>
        ReferenceEquals(this, other) ||
        (other is not null
         && EqualityContract == other.EqualityContract
         && EqualityComparer<decimal>.Default.Equals(<Amount>k__BackingField, other.<Amount>k__BackingField)
         && EqualityComparer<string>.Default.Equals(<Currency>k__BackingField, other.<Currency>k__BackingField));

    public override bool Equals(object? obj) => Equals(obj as Money);
    public override int GetHashCode() => /* combines EqualityContract and every field */;
    public static bool operator ==(Money? left, Money? right) => ReferenceEquals(left, right) || (left?.Equals(right) ?? false);
    public static bool operator !=(Money? left, Money? right) => !(left == right);

    public override string ToString() { /* "Money { Amount = 10, Currency = EUR }" */ }
    protected virtual bool PrintMembers(StringBuilder builder) { /* "Amount = 10, Currency = EUR" */ }

    public virtual Money <Clone>$() => new Money(this);     // unspeakable; used by `with`
    public void Deconstruct(out decimal Amount, out string Currency) { Amount = this.Amount; Currency = this.Currency; }
}
```

You can replace most of this by declaring the member yourself: your own `Equals(Money? other)`, `GetHashCode()`, `PrintMembers`, copy constructor, or (since C# 10) a `sealed override ToString()` to stop derived records regenerating it. You can't declare `==`/`!=` — they always delegate to `Equals`. If you declare `Equals(R?)` without `GetHashCode()`, the compiler warns, because you almost certainly broke the hash contract.

A `sealed record` omits the virtual machinery it doesn't need, which lets the JIT devirtualize equality calls. Two defaults follow: **seal records unless you intend inheritance**, and remember that non-positional properties declared in the body (`public string? Note { get; init; }`) participate in equality and `ToString` exactly like positional ones — "positional" controls the constructor and `Deconstruct`, not equality.

---

## Concept 17 — Record equality in depth

Five facts, each of which is an interview question in disguise.

**1. Every instance field participates — not just the properties you see.** The synthesized `Equals` compares each instance field declared in the record: positional backing fields, backing fields of body properties, private fields you declared, `field`-keyword backing fields, and the delegate field behind a field-like event. There is no attribute to exclude one. That's what made the cached-`Display` example in Concept 11 unequal, and it's why a record holding a lazily populated cache, a mutable counter, or an event is almost always a design error. If a field shouldn't participate in equality, the type probably isn't a value — or the field belongs somewhere else.

**2. `EqualityContract` makes a base record never equal a derived one.**

```csharp
public record Person(string Name);
public record Employee(string Name, string Team) : Person(Name);

Person p = new Person("Ada");
Person e = new Employee("Ada", "Platform");
Console.WriteLine(p == e);   // False, and e == p is also False
```

This fixes a classic flaw in hand-written equality across inheritance, where `base.Equals(derived)` could return true while `derived.Equals(base)` returned false — a symmetry violation that corrupts hash-based collections. The price is that "is-a" never implies "equals," which is the right answer for values.

**3. Hash stability is your problem.** Anything used as a dictionary key or `HashSet` element must not change its hash while it's in there. Positional `record class` properties are `init`-only, which protects you against direct mutation, but a `record struct` with default `{ get; set; }` properties (Concept 20), a body property with `set`, or a mutable collection inside the record all let the hash drift. The collection stops finding its own elements.

**4. `==` means value equality**, which surprises readers who expect reference comparison for class types. When you genuinely mean identity, say `ReferenceEquals(a, b)`.

**5. Equality recurses.** A record containing another record compares it with its `Equals`, and so on down the graph. A cyclic graph — a parent holding children that hold the parent — makes `Equals`, `GetHashCode`, and `ToString` recurse until the stack overflows (and a stack overflow cannot be caught; Module 14, Concept 9). Records are for trees of values, not graphs of objects.

---

## Concept 18 — `with` expressions: clone, then init

```csharp
var eur = new Money(10m, "EUR");
var usd = eur with { Currency = "USD" };
```

lowers to: call the virtual `<Clone>$()` (which calls the copy constructor), then call the `init` accessor of each property named in the braces on the clone. Because `<Clone>$` is virtual, `with` on a `Person`-typed variable holding an `Employee` produces an `Employee` — the runtime type is preserved.

Three properties of this lowering are traps:

**It's shallow.** Reference-typed fields are copied as references. A `with` on a record containing a `List<T>` gives you two records sharing one list.

**It never runs the constructor.** The copy constructor copies fields; the named `init` accessors run; nothing else does. So validation placed in a constructor is bypassed:

```csharp
public sealed record Percentage
{
    public Percentage(decimal value)
    {
        if (value is < 0 or > 100) throw new ArgumentOutOfRangeException(nameof(value));
        Value = value;
    }
    public decimal Value { get; init; }
}

var p   = new Percentage(40);
var bad = p with { Value = 250 };   // no exception: `with` never calls the constructor
```

(A positional record's primary constructor has no body at all, so there's nowhere to put the check in the first place.) The idiomatic answer since C# 14 is to validate in the `init` accessor, which runs on *both* paths:

```csharp
public sealed record Email
{
    public Email(string value) => Value = value;                // constructor assignment calls the init accessor

    public string Value
    {
        get;
        init => field = value.Contains('@')
            ? value.Trim().ToLowerInvariant()
            : throw new ArgumentException($"'{value}' is not an email address.", nameof(value));
    }
}

var a = new Email("Ada@Example.com");     // validated
var b = a with { Value = "garbage" };     // also validated: `with` calls the init accessor → throws
```

Note what isn't used: positional syntax, and a property initializer. A positional record that redeclares `Value` would have to initialize it with `= Value;`, and a property initializer assigns the backing field directly, *bypassing* the accessor (Concept 11) — so construction wouldn't validate while `with` would. For validated value objects, write the constructor explicitly and validate in `init`.

**It isn't available for arbitrary classes.** `with` works on records, on any struct (C# 10), and on anonymous types. A non-record class has no `<Clone>$`, so the compiler has nothing to call.

---

## Concept 19 — The collection-in-a-record bug

This is the single most common record defect, and it's worth being able to explain in one breath:

```csharp
public sealed record Order(OrderId Id, List<OrderLine> Lines);

var a = new Order(id, [new("SKU-1", 2)]);
var b = new Order(id, [new("SKU-1", 2)]);
Console.WriteLine(a == b);            // False
```

`List<T>` doesn't override `Equals`, so `EqualityComparer<List<OrderLine>>.Default` compares references. The record's value equality silently degrades to identity equality for that member — and, worse, the "immutable" record now holds a mutable list anyone can `Add` to, changing its hash code while it sits in a dictionary. The same applies to arrays, `ImmutableArray<T>` (its `Equals` compares the underlying array reference), `Dictionary<K,V>`, and most other collections. `ToString` is unhelpful too: `Lines = System.Collections.Generic.List`1[OrderLine]`.

The fixes, in order of preference:

1. **Ask whether the type needs value equality at all.** An `Order` with lines is probably an entity or an aggregate (Concept 21). If nobody compares two of them, a record buys you `with` and `ToString`, not correctness.
2. **Use a collection type with value semantics.** Write — or borrow — a small `EquatableArray<T>` / `ValueList<T>` wrapper that implements `IEquatable<>` with `SequenceEqual` and a combined hash. This is exactly what incremental source generators use for their pipeline models (Concept 59), because generator caching depends on value equality.
3. **Override `Equals(Order? other)` and `GetHashCode()`** in the record to compare the sequence. This works, but you're now maintaining hand-written equality again, which removes much of the point of the record.

Whatever you choose, expose the collection as `IReadOnlyList<T>` or an immutable type, not `List<T>`.

---

## Concept 20 — `record struct` and `readonly record struct`

C# 10 added value-type records:

```csharp
public record struct Point(int X, int Y);           // X, Y are { get; set; } — mutable!
public readonly record struct Range(int Start, int End); // Start, End are { get; init; }, fields readonly
```

What's different from `record class`:

- **Positional properties on a plain `record struct` are mutable** (`get; set;`). This asymmetry with `record class` surprises people. Default to `readonly record struct` unless you have a specific reason for mutability — mutable structs cause lost updates through copies (Module 14, Concept 16).
- **No inheritance, so no `EqualityContract`.** Equality is generated and strongly typed — `IEquatable<T>` with per-field `EqualityComparer<T>.Default` calls — which is much faster than the reflection-based `ValueType.Equals` fallback that plain structs get.
- **`with` works**, as it does for every struct: it copies the value and applies the setters.
- **`default` is always valid.** `default(Range)` is `(0, 0)` regardless of any constructor, and array elements start as `default`. You cannot establish an invariant a struct's default value violates. For a value object like `Email` whose `Value` must not be null, `default(Email)` has a null `Value` despite the non-nullable annotation — which is one of the stronger arguments for making such value objects classes.
- **Size still matters.** The struct-vs-class decision procedure from Module 14 (Concept 47) applies unchanged: small (on the order of 16 bytes or so), immutable, frequently created, often in collections. A `readonly record struct` with eight fields that gets passed around by value is a copying cost, not an optimization.

The sweet spot for `readonly record struct` is small, identity-free, frequently created values: strongly-typed IDs, composite dictionary keys, coordinates, money amounts in hot paths, and the tuple-like results of internal methods where a named type reads better than `(int, int)`.

---

## Concept 21 — Where records belong, and where they don't

**Records shine for:**

- **Messages, commands, and events** (Module 11). They're values by nature, immutable once published, and `with` makes versioned evolution ergonomic in tests.
- **DTOs and API contracts.** Value equality makes tests trivial; `ToString` makes logs readable; `required` plus `init` makes the contract explicit.
- **Value objects** in DDD (Module 22): `Money`, `Address`, `DateRange` — with validation in `init` accessors (Concept 18).
- **Composite keys** for dictionaries and caches: `readonly record struct CacheKey(TenantId Tenant, string Sku)`.
- **Configuration snapshots, query results, and test data.**

**Records are wrong for:**

- **EF Core entities.** Entities have identity (Concept 15). Value equality conflicts with how an ORM reasons about "the same row": navigation collections are commonly `HashSet<T>`, which consults `Equals`; two distinct entities with equal data collapse; and `with` produces an *untracked copy*, so `order with { Status = Paid }` followed by `SaveChanges()` saves nothing. EF Core's own guidance is to give entities reference or key-based identity. (Records are fine for *owned* or *complex* value types inside an entity — EF Core 8's complex types exist exactly for value objects without identity.)
- **Aggregates with behaviour.** An aggregate root is mutable by design, guards its invariants in methods, and is identified by its key. A record's public `init` surface and `with` are an open door around those methods.
- **Cyclic object graphs.** Equality, hashing, and `ToString` recurse (Concept 17).
- **Types whose equality should be a subset of their fields** — "same if the ID matches." That's an entity; write key-based equality on a class.
- **Public library types you expect to evolve.** Adding a positional parameter changes the primary constructor and the `Deconstruct` signature — a *binary* breaking change for every consumer. Adding any property changes equality semantics — a *behavioural* break that no compiler will flag. Non-positional `init` properties with `required` where needed are a more evolvable public shape than positional parameters.

The interview-grade summary: *records are for values; entities are for things with identity; a record is the wrong tool whenever "equal data" doesn't mean "same thing."*

---

## Concept 22 — Value objects and strongly-typed IDs

The cheapest domain-modeling upgrade in modern C# is refusing to pass bare primitives for identifiers:

```csharp
public readonly record struct OrderId(Guid Value);
public readonly record struct CustomerId(Guid Value);

public Task<Order?> GetAsync(OrderId id, CancellationToken ct);   // can't pass a CustomerId by mistake
```

`GetAsync(customerId)` is now a compile error rather than a production incident, and the declaration costs one line. The cost is not in the declaration; it's in the **integration surface**, and a senior answer names it:

- **JSON.** `System.Text.Json` serializes `OrderId` as `{"Value":"…"}` by default. For a clean wire format you need a `JsonConverter` (or a generator that writes one).
- **EF Core.** A value converter per ID type — `HasConversion(id => id.Value, v => new OrderId(v))` — or a convention in `ConfigureConventions` applied to all ID types.
- **Routing and binding.** Minimal APIs and MVC bind route and query values through a static `TryParse`, and since .NET 7 through `IParsable<T>` (Concept 57), so the ID type should implement it.
- **`default`.** `default(OrderId)` wraps `Guid.Empty`. A struct can't prevent it, so "empty ID" checks remain your job at boundaries.
- **Generation.** `Guid.CreateVersion7()` (.NET 9) gives time-ordered GUIDs that index better than random v4 values in B-tree clustered indexes (Module 12).

Libraries such as [Vogen](https://github.com/SteveDunn/Vogen) and [StronglyTypedId](https://github.com/andrewlock/StronglyTypedId) generate the converters, parsers, and validation for you — a good example of the compile-time metaprogramming category (Concept 2) paying for an invariant-encoding feature. Worked example 5 builds the whole thing end to end.

The judgement call: strongly-typed IDs are almost always worth it for aggregate and entity identifiers in a domain model, and rarely worth it for every integer in a codebase. The test is whether *mixing two of them up* is a plausible bug with real consequences.

---

# Part D — Pattern matching

## Concept 23 — Patterns are a sub-language

A pattern is a *test of a value's shape* that can also *bind* parts of the value to new variables. C# accumulated its pattern language over five releases, and it now composes like a small language of its own:

| Pattern | Example | Tests | Since |
|---|---|---|---|
| Declaration / type | `x is Circle c`, `Circle => …` | Runtime type (and binds) | C# 7 / 9 |
| Constant | `x is 42`, `x is null`, `"GET" => …` | Equality with a constant | C# 7 |
| `var` | `x is var v` | Always matches; binds | C# 7 |
| Discard | `_ => …` | Always matches | C# 8 |
| Property | `x is { Status: Paid, Total: > 100 }` | Members, recursively | C# 8 |
| Positional | `p is (0, 0)`, `Point(var x, _)` | `Deconstruct` outputs (or tuple elements) | C# 8 |
| Tuple | `(a, b) switch { (true, false) => … }` | Several values at once | C# 8 |
| Relational | `> 0`, `<= 100` | Comparison with a constant | C# 9 |
| Logical | `not null`, `>= 0 and < 10`, `'a' or 'b'` | Combinators | C# 9 |
| Parenthesized | `not (A or B)` | Grouping | C# 9 |
| Extended property | `{ Customer.Address.Country: "RS" }` | Nested members without nesting braces | C# 10 |
| List | `[1, 2, 3]`, `[_, _, ..]` | Count and elements | C# 11 |
| Slice | `[first, .. var rest]` | A sub-range, optionally bound | C# 11 |
| Span-on-constant-string | `span is "GET"` | `ReadOnlySpan<char>` against a string constant | C# 11 |
| Union / closed-hierarchy exhaustiveness | `Dog d => …` over a `Pet` union | Case types; compiler knows the full set | C# 15 |

Patterns appear in four places: `is` expressions, `switch` expressions, `switch` statements (`case … when …:`), and — for property patterns over extension properties, since C# 14 — anywhere those three appear.

The compositional core is small: *type tests*, *constant/relational tests*, *member access* (property, positional, list), and *combinators* (`and`, `or`, `not`). Everything else is a combination. If you think of patterns that way, rather than as a list of syntaxes, you can read any pattern by decomposing it.

---

## Concept 24 — `switch` expressions vs statements; order and subsumption

A switch expression is an *expression*: every arm yields a value, the whole thing has a type, and it must handle every input or warn (Concept 28).

```csharp
decimal ShippingFor(Order order) => order switch
{
    { Total: >= 100m }                              => 0m,
    { Destination.Country: "RS" or "HR" or "SI" }   => 4.90m,
    { Destination.Country: _, Weight: > 20 }        => 29m,
    _                                               => 12m,
};
```

Three rules:

1. **Arms are *semantically* tested top to bottom; the first match wins.** (How the compiler *implements* that is a different question — Concept 25.)
2. **An arm that can never be reached is an error, not a warning.** If a previous arm subsumes it, you get CS8510 ("the pattern has already been handled by a previous arm"). Reordering specific-before-general is the fix, and the error is a genuine safety feature: it catches the "general case accidentally placed first" bug at compile time.
3. **The arms must have a best common type**, or the switch must be target-typed.

Switch *statements* are still the right tool when arms perform different side effects of different shapes, when you need `break`/`goto case`, or when an arm is several statements long. The statement form gained patterns and `when` guards in C# 7, but it doesn't produce exhaustiveness warnings the way the expression form does. A heuristic that works in practice: if every arm computes a value, use the expression; if every arm *does* something, use the statement — and if an arm is more than a line or two, extract a method either way.

---

## Concept 25 — How patterns compile: the decision DAG

The compiler doesn't emit your arms as a sequence of `if` statements. It gathers every test from every arm, builds a **decision DAG** (a directed acyclic graph of tests and bindings), shares tests common to several arms, orders tests to minimize work, and emits the graph.

What that means concretely:

- **A type test shared by five arms is performed once.** `Circle { Radius: > 10 }` and `Circle { Radius: <= 10 }` do one `isinst Circle` between them.
- **Integral constants can become a jump table** (the IL `switch` instruction) and relational patterns on integers become a tree of comparisons — effectively a binary search over the ranges.
- **String constants are not tested with a chain of `string.Equals`.** Depending on the case count and compiler version, you get a hash-based jump table or a dispatch on length and a distinguishing character, then a final equality check.
- **Member reads are reused.** If three arms test `order.Status`, the getter may be called once and its result reused across the graph.

That last point is the one with a correctness consequence. **Patterns assume the members they read are pure**: the order in which the compiler evaluates subpatterns and the number of times it reads a property are not something you should depend on. A property getter with side effects — logging, lazy loading, incrementing a counter, hitting the database via a lazy-loading proxy — inside a pattern is a bug even if it happens to work today. This is the same "properties must be cheap and side-effect-free" principle that reappears for extension properties (Concept 45).

The performance consequence is the reassuring one: a well-structured switch expression is as fast as the best hand-written branching you would have produced, and often better, because the compiler considers all arms at once. When a switch over a type hierarchy is hot and has many arms, the costs you're paying are the type tests themselves (`isinst` is a pointer comparison for sealed types, a hierarchy walk otherwise) — which is one more reason to seal leaf types.

---

## Concept 26 — Property, positional, and extended property patterns

**Property patterns** test members recursively and compose with everything else:

```csharp
if (request is { Method: "POST", Headers.ContentType: "application/json", Body.Length: > 0 and <= 1_048_576 })
    { /* ... */ }
```

`{ }` alone means "not null," and `{ } x` means "not null, bind as `x`." **Extended property patterns** (C# 10) let you write `Headers.ContentType:` instead of `Headers: { ContentType: … }`. Since C# 14, extension properties participate in property patterns too.

**Positional patterns** match against the outputs of a `Deconstruct` method (which records generate) or tuple elements:

```csharp
string Quadrant(Point p) => p switch
{
    (0, 0)                => "origin",
    (> 0, > 0)            => "I",
    (< 0, > 0)            => "II",
    (< 0, < 0)            => "III",
    (> 0, < 0)            => "IV",
    (_, 0) or (0, _)      => "axis",
};
```

**Tuple patterns** switch on several values at once, which is the cleanest way to write a state-transition table:

```csharp
OrderState Next(OrderState state, OrderEvent evt) => (state, evt) switch
{
    (OrderState.Draft,   OrderEvent.Submit)  => OrderState.Pending,
    (OrderState.Pending, OrderEvent.Pay)     => OrderState.Paid,
    (OrderState.Pending, OrderEvent.Cancel)  => OrderState.Cancelled,
    (OrderState.Paid,    OrderEvent.Ship)    => OrderState.Shipped,
    _ => throw new InvalidOperationException($"{evt} is not valid in {state}"),
};
```

The judgement: positional patterns are compact but *positional* — `(> 0, < 0)` means nothing without knowing the order of the deconstruction. For types with more than two or three components, or where the reader won't know the order, property patterns (`{ X: > 0, Y: < 0 }`) are longer and far clearer. A good rule: positional for tuples and genuinely coordinate-like types, property patterns for everything else.

---

## Concept 27 — List and slice patterns

C# 11 added patterns over sequences:

```csharp
string Describe(ReadOnlySpan<string> args) => args switch
{
    []                              => "no arguments",
    ["--help" or "-h"]              => "help",
    ["run", var target]             => $"run {target}",
    ["run", var target, .. var rest]=> $"run {target} with {rest.Length} options",
    [.., "--verbose"]               => "something, verbosely",
    _                               => "unrecognized",
};
```

The requirements are structural rather than interface-based. A list pattern works on any type that is **countable** (an accessible `Length` or `Count`) and **indexable** (an `int` or `Index` indexer). A slice pattern that binds (`.. var rest`) additionally needs a `Range` indexer or a `Slice(int, int)` method. Arrays, `List<T>`, `Span<T>`, `ReadOnlySpan<T>`, `ImmutableArray<T>`, and `string` all qualify. `IEnumerable<T>` does not — it has no count or indexer, and enumerating it inside a pattern would be exactly the hidden-cost behaviour patterns are designed to avoid.

The compiler lowers `[first, .., last]` to a length check plus index reads (`s[0]`, `s[^1]`), so on arrays and spans it's allocation-free. Binding a slice on a `string` or an array allocates the slice (a new string or array); on a span it doesn't. List patterns are excellent for command-line and protocol parsing, token matching, and small structural checks. They're a poor fit for large collections where what you actually mean is a query.

---

## Concept 28 — Exhaustiveness: what the compiler can prove

A switch expression that doesn't handle every possible input produces **CS8509** ("the switch expression does not handle all possible values of its input type"), usually with an example of an unhandled value. If an unhandled value arrives at runtime, the generated code throws **`SwitchExpressionException`** (or `InvalidOperationException` where that type isn't available).

What "every possible input" means is decided by the input type's *value space*, and this is the part candidates get wrong:

- **`bool`** has two values; both arms make it exhaustive.
- **Integral types** are exhaustive only if the relational patterns cover the full range — `< 0`, `0`, `> 0` does it for `int`.
- **Enums are never closed.** An enum's value space is its underlying integer type; `(OrderState)42` is a perfectly legal value. Covering every named member still leaves a gap, and the compiler says so with **CS8524** ("does not handle some values of its input type … involving an unnamed enum value"). This is not pedantry: enums arrive from databases, JSON, and older service versions carrying values your code has never seen.
- **Class hierarchies were never closed — until C# 15.** Before `closed`, the compiler could not assume that `Circle`, `Square`, and `Triangle` were the only subclasses of an abstract `Shape`, because another assembly can add one. A switch over them always needed a discard arm. With a `closed` base (Concept 33) or a `union` (Concept 34), the compiler knows the full set, and handling every case is exhaustive with no default arm.
- **Nullable inputs** add `null` to the value space; a switch over `Shape?` must handle `null` too.

So the answer to "does C# have exhaustive pattern matching?" changed this year. Before C# 15: yes for `bool`, integers and tuples of them, and for nothing a domain model is made of. From C# 15: also for closed hierarchies and unions — which is what made the feature worth waiting for.

---

## Concept 29 — The discard arm is a design decision

Given how exhaustiveness works, a `_ =>` arm is not neutral boilerplate. It is a statement: "every value I haven't named is handled *this way*, now and forever."

```csharp
string Label(PaymentState s) => s switch
{
    PaymentState.Pending   => "Awaiting payment",
    PaymentState.Captured  => "Paid",
    _                      => "Unknown",      // silences CS8524 today...
};
// ...and when someone adds PaymentState.Refunded next quarter, this happily returns "Unknown"
// in every screen and report, and no compiler anywhere tells you.
```

The trade is explicit:

- **With a discard**, you get silence now and a silent wrong answer when the set grows.
- **Without one** (on a closed set), you get a compile-time error or warning at *every* switch when the set grows — the compiler becomes the to-do list for the change.
- **With a throwing discard** (`_ => throw new UnreachableException()` — `UnreachableException` is .NET 7+), you get a loud runtime failure instead of a silent wrong answer, which is the right choice for enums, where the compiler cannot help you.

The senior default is: **no discard on closed sets; a throwing discard on enums; a value-producing discard only where "everything else" is a genuine, stable business category** (for example, "all countries not listed pay standard shipping"). Treating warnings as errors (`CS8509`, `CS8524`) makes the whole policy enforceable.

At service boundaries, the calculus reverses: a message from a newer producer may legitimately contain a case your consumer doesn't know yet, and crashing is worse than degrading. That's a tolerant-reader problem, and Concept 40 addresses it.

---

## Concept 30 — Null patterns

```csharp
if (customer is null) return;
if (customer is not null) { … }
if (order is { Customer: { } c }) Notify(c);   // non-null, and bind
```

Why `is null` rather than `== null`: `==` can be user-defined, and a type can overload it to do something other than a reference check. `is null` always compiles to a reference comparison (or `HasValue` for nullable value types); it can't be hijacked. Historically this mattered for Unity's `UnityEngine.Object`, which overloads `==` to report destroyed objects as null; in ordinary code it's mostly about consistency and intent. Either way, patterns participate fully in nullable flow analysis: after `if (x is not null)`, `x` is known not-null (Concept 48).

Two smaller points that show up in reviews:

- **`is { }` versus `is not null`.** They're equivalent tests; `is { } x` is useful when you also want to bind, especially on a member chain. Pick one style for plain checks and let `.editorconfig` enforce it.
- **Null in switch expressions goes first or is subsumed.** A `null` arm must come before any arm that would match null, and property patterns never match null. For union types, `null` has a specific meaning — "the union holds no value" — covered in Concept 35.

---

# Part E — Closed sets: closed hierarchies and unions (C# 15)

## Concept 31 — Sum types: the twenty-year gap

Type theory has two basic ways of combining types:

- A **product type** holds *this and that*: a record `Point(int X, int Y)` holds an `X` **and** a `Y`. Its number of possible values is the product of its parts'. C# has had product types since 1.0 — classes, structs, tuples, records.
- A **sum type** holds *this or that*: a payment result is `Captured` **or** `Declined` **or** `RequiresAction`, each with its own data. Its number of possible values is the *sum* of its cases'. F#, Rust, Swift, Kotlin, Scala, TypeScript, and Haskell all have sum types natively — as discriminated unions, enums with payloads, or sealed hierarchies.

Before C# 15, C# developers encoded sum types, each encoding with a flaw:

| Encoding | Flaw |
|---|---|
| An enum plus nullable fields for per-case data (`Status`, `DeclineReason?`, `RedirectUrl?`) | Nothing stops `Status = Captured` with a `DeclineReason` set; invalid combinations are representable |
| An abstract base class plus subclasses | Open: any assembly can add a case, so the compiler can't check exhaustiveness |
| An abstract base with a `private` constructor and **nested** sealed subclasses | Genuinely closed (only nested types can call the constructor) but awkward, and the compiler still didn't reason about it for exhaustiveness |
| A marker interface | Open, and any type anywhere can implement it |
| A library type such as [OneOf](https://github.com/mcintyre321/OneOf)`<T0, T1, T2>` | Positional cases (`T0`, `T1`) instead of names, a `Match` method instead of `switch`, and no language-level exhaustiveness |
| Source generators such as [Dunet](https://github.com/domn1995/dunet) | Better ergonomics, still not the compiler's own concept |

C# 15 adds two complementary language features — `closed` classes for hierarchies you design, and `union` types for composing existing types — and together they give C# real sum types with compiler-checked exhaustiveness. Closed enums are announced as a planned third shape of the same idea.

---

## Concept 32 — Make illegal states unrepresentable

The phrase comes from Yaron Minsky's writing on OCaml at Jane Street, and it's the design principle that makes sum types matter. Consider a connection modelled the usual way:

```csharp
public sealed class Connection
{
    public ConnectionState State { get; set; }      // Disconnected, Connecting, Connected, Failed
    public DateTimeOffset? ConnectedAt { get; set; } // only meaningful when Connected
    public string? SessionId { get; set; }           // only meaningful when Connected
    public int Attempt { get; set; }                 // only meaningful when Connecting
    public Exception? LastError { get; set; }        // only meaningful when Failed
}
```

Five properties produce hundreds of combinations, of which four are valid. Every consumer must know which fields are meaningful in which state, every method must defend against the invalid combinations, and every test must cover them. The type is lying about the domain: it claims all those combinations exist.

Model each state as a type that carries exactly its data:

```csharp
public closed record ConnectionState;
public sealed record Disconnected : ConnectionState;
public sealed record Connecting(int Attempt) : ConnectionState;
public sealed record Connected(string SessionId, DateTimeOffset Since) : ConnectionState;
public sealed record Failed(Exception Error, int Attempts) : ConnectionState;
```

Now `Connected` without a session ID cannot be constructed, `Failed` cannot lack an error, and a switch over `ConnectionState` that handles four cases is provably complete. The invalid states haven't been *validated away*; they've been *made unrepresentable*, so there is nothing to validate.

The companion principle is Alexis King's **"parse, don't validate"**: at the boundary, convert unstructured input (a JSON payload, a string, a database row) into a type that can only hold valid values, and let the rest of the program trust the type. A method that takes `Connected` instead of `ConnectionState` doesn't need to check the state — the type system already did. This is where sum types pay off beyond the switch statement: they shrink the set of things every function must defend against.

---

## Concept 33 — `closed` classes (C# 15)

```csharp
public closed record class JobStatus;
public sealed record class Queued : JobStatus;
public sealed record class Running(int PercentComplete) : JobStatus;
public sealed record class Completed(TimeSpan Elapsed) : JobStatus;
public sealed record class Failed(string Error) : JobStatus;

public static string Describe(JobStatus status) => status switch
{
    Queued                 => "waiting to start",
    Running(var percent)   => $"{percent}% complete",
    Completed(var elapsed) => $"finished in {elapsed.TotalSeconds:F1}s",
    Failed(var error)      => $"failed: {error}",
    // exhaustive — no default arm, no warning
};
```

The rules, from the language reference:

- **Direct subtypes must be declared in the same assembly** (and module) as the closed class. Another assembly deriving directly from `JobStatus` gets a compile error. Same-assembly is the natural boundary because it's the unit the compiler sees completely.
- **A closed class is implicitly `abstract`.** It can't be combined with `sealed`, `static`, or an explicit `abstract`.
- **Closedness is not transitive.** A non-sealed descendant of a closed class is an ordinary open class: another assembly can derive from `Failed` above unless `Failed` is `sealed` or itself `closed`. To extend exhaustiveness down a multi-level hierarchy, mark the intermediate levels `closed` too — and seal the leaves.
- **Generic descendants must use all their type parameters in the base-class specification.** `record Leaf<T>(T Value) : Tree<T>` is fine; `record Constant<U>(U Value) : Tree<int>` is an error, because the compiler couldn't enumerate the descendants of `Tree<int>`.
- **A type parameter constrained to a closed type is treated as that closed type** for exhaustiveness, so generic methods over `where T : JobStatus` get exhaustive switches too.
- **Nullable inputs still need a `null` arm.**
- `closed` is a **contextual keyword**; existing identifiers named `closed` keep working (use `@closed` where the modifier would also be valid).

Two properties make `closed` the right choice over a union in many domain models. Cases **share a base type**, so they can share members — a `JobStatus.IsTerminal` property, a common `Timestamp`, an abstract method each case implements. And the cases are **your types**, designed together, with inheritance available when it helps.

One caution to say out loud in a design discussion: `closed` is a **compile-time** contract. The feature specification includes a mechanism for blocking subtyping from other languages and older compilers, so it is designed to hold across the ecosystem rather than only in C# — but runtime-generated subclasses (mocking and proxy libraries emitting IL) are a different matter, and like `init` and nullability, the thing it guarantees is what compiled code can express, not what reflection can do. Verify the edge cases you depend on against the speclet.

---

## Concept 34 — `union` types (C# 15)

A closed hierarchy closes a set of types *you design to be related*. A union closes a set of *existing* types that need not be related at all:

```csharp
public sealed record Cat(string Name);
public sealed record Dog(string Name);
public sealed record Bird(string Name);

public union Pet(Cat, Dog, Bird);

Pet pet = new Dog("Rex");          // implicit conversion from each case type

string name = pet switch
{
    Dog d  => d.Name,
    Cat c  => c.Name,
    Bird b => b.Name,              // exhaustive: the compiler knows the three case types
};
```

The rules worth knowing:

- **Case types can be any type that converts to `object`**: classes, structs, interfaces, type parameters, nullable types, other unions, and types you don't own — `public union IntOrString(int, string);` is legal.
- **Conversions go one way, implicitly**: from each case type into the union, via a compiler-generated constructor. A user-defined implicit conversion for the same type wins over the union conversion; if two case types are equally applicable to a value, the conversion is ambiguous and is a compile error.
- **Patterns unwrap.** Patterns against a union apply to its contents (the `Value` property), not to the union itself — which is why `Dog d =>` works. Three patterns are exceptions and apply to the union value itself: the discard `_`, `var`, and `not`. A consequence that surprises people: `pet is Pet` typically does *not* match, because it tests the contents against `Pet`.
- **A union can have a body** — methods, computed properties, even members that switch over `this` — but no instance fields, auto-properties, or field-like events (it has no storage of its own beyond the value), and no public single-parameter constructors (those are the generated case constructors).

```csharp
public sealed record Meters(double Value);
public sealed record Feet(double Value);

public union Length(Meters, Feet)
{
    public double TotalMeters => this switch
    {
        Meters m => m.Value,
        Feet f   => f.Value * 0.3048,
        _        => throw new InvalidOperationException("The Length has no value."), // default(Length) holds nothing
    };
}
```

The discard in that last example isn't sloppiness; it handles the empty union, which is the subject of the next concept.

---

## Concept 35 — What a union is at runtime

This is where the senior-level questions are. The compiler lowers `public union Pet(Cat, Dog, Bird);` to, in effect:

```csharp
[System.Runtime.CompilerServices.Union]
public struct Pet : System.Runtime.CompilerServices.IUnion
{
    public Pet(Cat value)  => Value = value;
    public Pet(Dog value)  => Value = value;
    public Pet(Bird value) => Value = value;
    public object? Value { get; }
}
```

`UnionAttribute` and `IUnion` ship in the .NET 11 BCL. Everything interesting follows from that shape:

- **A union is a struct holding one `object?` reference.** Passing one around costs a pointer-sized copy; it adds no allocation of its own for reference-type cases.
- **Value-type cases are boxed.** `IntOrString x = 42;` allocates a boxed `int` to store in `Value`. The documentation is explicit that the generated form is opinionated — always a struct, always `object?`, always boxing value types. In a hot path over value-type cases, that's an allocation per value (Module 14, Concept 14), and it's the main reason custom unions exist (Concept 36).
- **`default(Pet)` holds nothing.** Its `Value` is `null`. Uninitialized fields, array elements, and `default` all produce an empty union. The compiler tracks this through nullability: if a union value might be default — not definitely assigned from a case — you must handle `null` in the switch to avoid a warning.
- **`Pet?` is `Nullable<Pet>`**, a nullable struct wrapping the union. A `null` pattern against it matches both "no `Pet`" and "a `Pet` holding nothing."
- **Pattern matching goes through `Value`**, which is a runtime type test on an `object?` — cheap for reference types, and a box-then-unbox round trip for value types.
- **`IUnion` lets you detect unions at runtime** (`value is IUnion { Value: null }`), and Roslyn exposes union case types to analyzers — but note that treating a union *as* `IUnion` boxes the union struct itself.

The practical boundary that matters most in architecture work: **serialization**. In .NET 11, `System.Text.Json` supports unions natively, and the design choice it made is instructive. It writes a union as its **active case only** — no envelope, no `$type`, no discriminator: `IntOrString` serializes as `42` or `"hello"`. Reading works automatically when the cases have different JSON token types (a `bool` versus a `string`), but when two cases are both JSON objects (`Cat` versus `Dog`) the payload is ambiguous, and you must opt into a classifier — `[JsonUnion(TypeClassifier = typeof(JsonUnionTypeStructuralClassifier))]` guesses from property names, which costs a scan of the object and makes classification depend on field names that may change. ASP.NET Core inherits exactly this behaviour wherever it uses STJ (request and response bodies, SignalR's JSON protocol, Blazor interop), and does **not** support unions in route values, query strings, headers, or form fields, because a bare string token gives no way to choose a case. The .NET team's own guidance draws the line where this module does: use a union to describe an *existing* discriminator-free contract (Kubernetes' `maxUnavailable` being `2` or `"25%"`); for a *new* polymorphic contract you control, use a closed hierarchy with a discriminator — `[JsonPolymorphic(InferClosedTypePolymorphism = true)]` on a `closed` base lets STJ infer the derived types without registering each one — because the discriminator becomes part of your documented contract rather than a guess the reader makes.

---

## Concept 36 — Custom unions: choosing your own representation

Because the generated form is fixed, C# 15 also defines unions by *pattern*: any class or struct marked `[Union]` that follows the **basic union pattern** — one or more public single-parameter constructors (each parameter type is a case type) and a public `object? Value` getter — gets union conversions, pattern matching, and exhaustiveness. The compiler assumes you uphold the behavioural rules: `Value` only ever holds `null` or a case-type value, a union created from a case keeps that case, and so on.

Three extensions of the pattern cover the cases the default can't:

**The non-boxing access pattern.** Add a `bool HasValue` property and a `bool TryGetValue(out T value)` method per case, and the compiler calls those instead of type-testing `Value` — so value-type cases never box during matching:

```csharp
[System.Runtime.CompilerServices.Union]
public readonly struct IntOrBool : System.Runtime.CompilerServices.IUnion
{
    private readonly int _int;
    private readonly bool _bool;
    private readonly byte _tag;                 // 0 = none, 1 = int, 2 = bool

    public IntOrBool(int? value)  { if (value is int i)  { _int = i;  _tag = 1; } }
    public IntOrBool(bool? value) { if (value is bool b) { _bool = b; _tag = 2; } }

    public object? Value => _tag switch { 1 => _int, 2 => _bool, _ => null };  // boxes only if someone asks
    public bool HasValue => _tag != 0;
    public bool TryGetValue(out int value)  { value = _int;  return _tag == 1; }
    public bool TryGetValue(out bool value) { value = _bool; return _tag == 2; }
}
```

Type patterns call `TryGetValue`; null patterns call `HasValue`; each is independently optional.

**Class-based unions.** A `[Union]` *class* gives reference semantics and inheritance. For class unions, a `null` pattern matches both a null reference and a null `Value`.

**Union member providers.** A nested `IUnionMembers` interface declaring static `Create` methods (one per case) plus `Value`, and optionally `TryGetValue`/`HasValue`, lets the union use private constructors or factory logic — useful for adapting a type whose public constructors you don't want to be union conversions.

When would you actually write one? Three cases: a hot path over value-type cases where boxing shows up in a profile; adapting an existing result type (a library's `Result<T>`, or a OneOf-style type) so it participates in exhaustiveness; and interop or layout requirements the default can't meet. Otherwise, the one-line declaration is the right default — and the fact that this escape hatch exists is itself a good answer to "isn't boxing a problem?"

---

## Concept 37 — Choosing a closed-set shape

You now have five ways to say "one of these," and the choice is a design decision you should be able to defend:

| Shape | Use when | Exhaustive? | Per-case data? | Shared members? | Cases you don't own? |
|---|---|---|---|---|---|
| `enum` | A fixed set of labels with no per-case data; flags; database-friendly codes | **No** — any integer is a value | No | No | — |
| `interface` | The set of implementations is *open* and should stay open (plugins, providers, strategies) | No | Yes | Contract only | Yes |
| Abstract base class (open) | Open set with shared implementation | No | Yes | Yes | No |
| `closed` hierarchy | A closed set of *related* cases you design together, possibly with shared behaviour | **Yes** | Yes | Yes | No |
| `union` | A closed set of *existing, possibly unrelated* types, including primitives and types from other libraries | **Yes** | Via the case types | Via a union body | **Yes** |

Two heuristics settle most cases. **If the cases are your domain's states or outcomes, designed together, use a closed hierarchy** — it gives you names, shared members, and records per case. **If the cases already exist and you're composing them** — `union Result(Order, ValidationProblem, NotFound)` over types that live in different layers, or `union Input(string, Stream, ReadOnlyMemory<byte>)` — **use a union**.

Enums stay the right tool for plain labels, especially ones persisted as integers or strings — but with a throwing discard in every switch (Concept 29), because the compiler cannot close them for you.

---

## Concept 38 — The expression problem: pattern matching vs virtual dispatch

This is the concept that turns a syntax question into an architecture question. Philip Wadler named it in 1998: you have a set of *types* (cases) and a set of *operations* over them. Can you add both new types and new operations without editing existing code and without losing static type safety?

The two classic designs make opposite trade-offs:

| | Add a new **case** | Add a new **operation** |
|---|---|---|
| **Virtual dispatch** (an interface or abstract method per operation; each case implements it) | Easy — one new class | Hard — edit every existing class |
| **Pattern matching over a closed set** (cases are data; each operation is a function with a switch) | Hard — edit every switch | Easy — one new function |

With exhaustiveness checking, the "hard" side of pattern matching becomes *mechanical*: when you add a case, the compiler lists every switch that must handle it. Without exhaustiveness — the C# situation until this year — that side was hard *and* unsafe, which is why object-oriented dispatch was the default answer in C# for two decades.

So the decision procedure is: **ask which axis grows.**

- **Cases are stable, operations proliferate** → closed set plus pattern matching. Workflow and order states, payment outcomes, protocol messages, AST nodes, command results, validation errors. You'll keep adding reports, projections, transitions, and handlers; the states rarely change.
- **Cases proliferate, operations are stable** → interfaces plus virtual dispatch, typically wired through DI. Payment providers, notification channels, storage backends, pricing strategies, plugins. You'll keep adding implementations; the operations (`Charge`, `Send`, `Save`) rarely change.

The **Visitor pattern** is object-orientation's workaround for wanting the pattern-matching side: double dispatch that lets you add operations over a closed hierarchy. With closed hierarchies and exhaustive switch expressions, a visitor in C# is now usually ceremony you can delete.

Saying "this is the expression problem — which axis do we expect to grow?" in a design round is one of the highest-value sentences in this module. It converts a stylistic debate into a question about the domain, which is exactly the move interviewers are looking for.

---

## Concept 39 — Result types vs exceptions: an architectural stance

Unions make the "Result type" debate concrete, and a senior candidate needs a position that isn't dogma.

**Exceptions** are for the unexpected and the unrecoverable-at-this-level: programming errors (`ArgumentException`, `InvalidOperationException`), infrastructure failures (a timeout, a dropped connection, a full disk), and violated invariants. They unwind the stack to whoever can actually handle them — often a middleware that turns them into a 500 and a log entry. They carry a stack trace, which is precisely what you want for things that shouldn't happen. They're also expensive relative to a return value (microseconds, not nanoseconds — though .NET 9's reworked exception handling made throwing substantially faster), which is irrelevant for rare failures and very relevant for a validation path running at thousands of requests per second.

**Result types** are for *expected* outcomes that the caller must handle: validation failed, not found, insufficient funds, already exists, conflict, requires additional authentication. These aren't exceptional — they're part of the operation's contract — and a union makes the compiler enforce that the caller considers each one:

```csharp
public sealed record Placed(OrderId Id);
public sealed record OutOfStock(string Sku);
public sealed record PaymentDeclined(string Reason);

public union PlaceOrderResult(Placed, OutOfStock, PaymentDeclined, ValidationErrors);
```

The anti-patterns on both sides are worth naming:

- **Result everywhere**: wrapping every database call and HTTP request in a `Result<T, Error>`, which re-implements exceptions badly, loses stack traces, and forces every layer to thread errors it can't handle. Scott Wlaschin, who popularized "railway-oriented programming," later wrote a piece titled *Against Railway-Oriented Programming* warning against exactly this.
- **Exceptions for control flow**: throwing `NotFoundException` from a repository and catching it in a controller to return a 404, on a path that happens constantly.

The stance that holds up: **domain outcomes in the type, infrastructure failures as exceptions, and translation at the edge.** In ASP.NET Core, the edge is the endpoint, and minimal APIs already have a union-shaped return type — `Results<Ok<T>, NotFound, ValidationProblem>` (since .NET 7) — so the mapping is a switch expression from your domain union to `TypedResults`. Worked example 3 builds it.

Libraries like ErrorOr, FluentResults, and OneOf filled this gap for years. With C# 15 unions, the language covers the core case; a library earns its place only if it adds something specific (error aggregation, fluent composition) that your team actually uses.

---

## Concept 40 — Versioning closed sets: the breaking change is the feature

Adding a case to a closed hierarchy or a union makes every exhaustive switch over it non-exhaustive. Inside one codebase, that is exactly what you want: the compiler produces a to-do list of every place that must handle `Refunded`, and with warnings as errors, the build won't pass until they're all handled. It's the strongest argument for closed sets.

Across a boundary, the same property is a hazard, in two distinct ways:

- **Source compatibility.** If a *published library* exposes a closed set, adding a case breaks every consumer's build the moment they upgrade (under warnings-as-errors) — a breaking change that must be versioned as such.
- **Binary compatibility.** A consumer compiled against the old version and *not recompiled* has no idea the new case exists. When one arrives at runtime, its switch expression throws `SwitchExpressionException`. The compiler's help only applies to code it compiles; already-deployed code gets a crash.

For distributed systems, that second point is decisive. When service A publishes an event whose payload is a closed set and service B consumes it, A will add a case before B is updated — that's how independent deployment works (Module 11). So:

- **Inside a bounded context or a single deployable**: closed sets and exhaustive switches, no discards. Let the compiler find every change.
- **At the wire boundary**: a *tolerant reader*. The deserialized contract includes an explicit "unknown" case (or a raw fallback), consumers handle it deliberately — log, park in a dead-letter queue, degrade — and the anti-corruption layer maps from the wire's open set to the domain's closed set.
- **In a public library**: treat adding a case as a major-version change, or don't expose the closed set publicly and give consumers a stable abstraction instead.

This is the versioning version of the expression problem, and being able to say "closed inside, open at the edges" is the architect-level answer to "should we use unions for our event contracts?"

---

# Part F — Extension members (C# 14 and 15)

## Concept 41 — Classic extension methods: a static call in disguise

Extension methods arrived in C# 3 to make LINQ possible:

```csharp
public static class StringExtensions
{
    public static bool IsBlank(this string? s) => string.IsNullOrWhiteSpace(s);
}

bool b = input.IsBlank();          // compiles to: StringExtensions.IsBlank(input)
```

Everything about their behaviour follows from that lowering:

- **They're static calls.** No virtual dispatch, no polymorphism. The method is chosen by the *static* type of the receiver at compile time. An extension on `Animal` is not overridden by an extension on `Dog`, and if the variable is typed `Animal`, the `Animal` version runs even when the object is a `Dog`.
- **A null receiver is legal.** `((string?)null).IsBlank()` calls the method with `null`; there's no `NullReferenceException` at the call site. That's useful (`IsBlank` above) and surprising (every other method call on null throws).
- **Instance members always win.** If the type gains an instance method with a compatible signature, it silently takes over at the next recompile. Library authors adding a member can therefore change which code runs in a consumer.
- **Lookup is by imported namespace**, innermost first. Two extensions with the same signature in two imported namespaces are ambiguous — a compile error that can appear simply because you added a NuGet package.
- **They're marked with `[Extension]`** on the method and the class, which is how other compilers and tools recognize them.
- **`ref this`** (C# 7.2) lets an extension on a struct mutate the caller's value rather than a copy.

Hold on to "a static call chosen at compile time"; it's the key to what the C# 14 generalization can and cannot do.

---

## Concept 42 — Extension blocks (C# 14, extended in C# 15)

For fifteen years, "extension" meant "extension *method*." You couldn't add a property, a static member, or an operator to a type you didn't own. C# 14 introduces **extension blocks**, declared inside a top-level, non-generic static class:

```csharp
public static class SequenceExtensions
{
    // Instance extension members: the receiver is named
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();                        // extension property
        public IEnumerable<T> WhereNot(Func<T, bool> predicate)      // extension method
            => source.Where(x => !predicate(x));
    }

    // Static extension members: receiver type only, no name
    extension<T>(IEnumerable<T>)
    {
        public static IEnumerable<T> Empty => [];                     // appears as IEnumerable<T>.Empty
        public static IEnumerable<T> operator +(IEnumerable<T> left, IEnumerable<T> right)
            => left.Concat(right);                                   // extension operator
    }
}

// C# 15: extension indexers
public static class BitExtensions
{
    extension(ref ulong bits)
    {
        public bool this[int index]
        {
            get => (bits & (1UL << index)) != 0;
            set => bits = value ? bits | (1UL << index) : bits & ~(1UL << index);
        }
    }
}
```

Usage reads as if the members were declared on the type: `orders.IsEmpty`, `IEnumerable<int>.Empty`, `a + b` on two sequences, `flags[3] = true`.

The rules that shape design:

- **The receiver** is declared once per block — `extension(Receiver name)` for instance members, `extension(Receiver)` when the block only holds static members. Its name is in scope in every instance member of the block.
- **Type parameters used by the receiver go on the block**; type parameters specific to one member go on that member; the same parameter can't appear in both places.
- **The receiver can be `ref`, `ref readonly`, or `in`** when it's known to be a value type — the bit-manipulation indexer above mutates the caller's `ulong` in place.
- **Member kinds**: methods, properties, and operators in C# 14; indexers in C# 15. Indexers require a named receiver, since indexers are always instance members.
- **Nullable annotations and attributes work on the receiver**, so `extension([NotNullWhen(false)] string? s) { public bool IsNullOrEmpty => … }` teaches flow analysis the contract.
- **Where extension properties participate**: object initializers, `with` expressions, and property patterns (`x is { IsEmpty: false }`), alongside the places extension methods always participated (`GetEnumerator` for `foreach`, `Deconstruct`, `GetAwaiter`, collection-initializer `Add`). They do *not* satisfy `Count`/`Length` for list patterns or implicit indexers.

---

## Concept 43 — How extension members lower, and why migration is safe

For instance extension *methods*, the compiler emits exactly what a classic `this`-parameter method would: a static method on the containing class, with the receiver prepended as the first parameter and `[Extension]` applied. The two forms are **binary- and source-compatible** — callers can't tell them apart — which means a library can convert its existing extension methods to blocks without breaking any consumer.

For the new member kinds, the compiler emits ordinary static **implementation methods** with speakable names:

```text
extension<T>(IEnumerable<T> source) { public bool IsEmpty => … }
    ↓
public static bool get_IsEmpty<T>(IEnumerable<T> source) { … }
```

alongside metadata that preserves the *shape* of the extension block for other compilers, tools, and documentation: a nested **extension grouping type** per CLR-level receiver signature, an **extension marker type** carrying the exact C# receiver (nullability, names, attributes), and skeleton member declarations pointing back to the marker. The names of these types are **content-based** — derived from the receiver signature rather than declaration order — so XML documentation IDs and public-API tracking stay stable when you reorder blocks.

Two practical consequences:

- **Disambiguation is always possible.** If two imported classes both define `IsEmpty` for the same receiver, `source.IsEmpty` is ambiguous, but you can call the implementation method directly: `SequenceExtensions.get_IsEmpty(source)`. The same works for static extension members and operators.
- **Call sites cost exactly a static call.** `orders.IsEmpty` compiles to a direct call to `get_IsEmpty`, which the JIT can inline like any other static method. There is no wrapper object, no interface, no allocation.

---

## Concept 44 — What extensions cannot do

The limits all follow from "a static call chosen at compile time, with no storage":

- **No state.** You cannot declare fields in an extension block, so there are no extension *auto*-properties — an auto-property needs a backing field, and there's nowhere to put one per receiver instance. Every extension property is computed. (If you truly need per-instance attached state, `ConditionalWeakTable<TKey, TValue>` exists — Module 14, Concept 42 — but reaching for it is usually a sign the design is fighting you.)
- **No `init` accessors**, because there's no object-initialization phase an extension participates in with storage.
- **No virtual dispatch.** `abstract`, `virtual`, `override`, `sealed`, `new`, `protected`, `partial`, and `readonly` aren't allowed on extension members. An extension property on a base type is not "overridden" by one on a derived type.
- **No interface implementation.** An extension cannot make an existing type *implement* an interface. This is the part of the long-discussed "roles"/"extension types" design that did not ship; extension members let you *call* things as if they were members, not *satisfy contracts* with them.
- **Instance members still win.** An extension property named like a real property of the receiver type is never chosen.
- **Ambiguity is a build break.** Two packages both adding `IsEmpty` to `IEnumerable<T>` in namespaces you import will stop your code compiling. The more popular the receiver type, the higher that risk — `IEnumerable<T>`, `string`, and `object` are the worst places to put generically named extensions.

The interview framing: *extension members add syntax, not semantics — they make static helpers read like members, and they cannot give a type new state or new behavioural identity.*

---

## Concept 45 — Extension design judgement

The language reference's own showcase declares an extension property `Median` on `IEnumerable<int>` that sorts the sequence on every access. It's a fine syntax demo and a useful design warning, because **property syntax makes promises**: that reading it is cheap, has no side effects, and returns the same value twice in a row. The Framework Design Guidelines have said this about properties for twenty years, and extension properties inherit the rule with interest, because on an interface like `IEnumerable<T>` the receiver might be:

- a lazy LINQ pipeline, so `IsEmpty` re-executes the whole query;
- an EF Core `IQueryable<T>` — hopefully with a translatable `Any()`, possibly not;
- a one-shot stream that can't be enumerated twice.

`if (!orders.IsEmpty) foreach (var o in orders) …` enumerates twice, and the property syntax hides it where a method call (`orders.Any()`) would at least look like work. So:

**Good uses**
- Cheap derived facts on types you don't own: `span.IsAscii`, `timeSpan.IsNegative`, `uri.IsLoopbackHost`.
- Static factories and constants where callers will look for them: `IEnumerable<T>.Empty`, `TimeSpan.FromBusinessDays(…)` on a type you don't control.
- Operators on types you don't own when the algebra is genuinely natural (vector or unit arithmetic).
- Layering: mapping extensions like `order.ToResponse()` in an API layer keep the domain type free of transport concerns.
- Indexers (C# 15) where the receiver has a real notion of position that its author didn't expose.

**Smells**
- **I/O or expensive work behind property syntax** (`order.Customer` that queries the database).
- **Extensions on your own types** to add core behaviour — that scatters a type's behaviour across files and namespaces; put it on the type. (Layering, as above, is the legitimate exception.)
- **Generic names on ubiquitous receivers** (`string.Value`, `object.ToJson`) — collision and IntelliSense pollution.
- **Global usings of extension namespaces**, which make every extension visible everywhere and turn every package upgrade into a potential ambiguity.

The rule of thumb: an extension should read like a member the type's author *would have written if they'd thought of it*. If the type's author would have made it a method, you should too.

---

## Concept 46 — User-defined compound assignment (C# 14)

Until C# 14, `a += b` always meant `a = a + b` for user-defined types: call the static `operator +`, which returns a *new* instance, then assign it. For small immutable values that's exactly right. For large mutable types — a matrix, a tensor, a buffer, an accumulator — it allocates a fresh instance per `+=` in a loop.

C# 14 lets a type define compound assignment as an **instance** operator that mutates in place:

```csharp
public sealed class Accumulator
{
    private readonly double[] _values;
    public Accumulator(int size) => _values = new double[size];

    public static Accumulator operator +(Accumulator a, Accumulator b)   // still available: returns a new instance
    { var r = new Accumulator(a._values.Length); /* ... */ return r; }

    public void operator +=(Accumulator other)                           // C# 14: in-place, no allocation
    {
        for (int i = 0; i < _values.Length; i++) _values[i] += other._values[i];
    }
}
```

When both exist, `a += b` uses the instance form; otherwise it falls back to the old `a = a + b` expansion. The feature also covers the instance forms of `++` and `--`, has `checked` variants, and — combined with extension operators — lets you add operators to types you don't own.

The caution is semantic, not syntactic: **in-place `+=` changes aliasing behaviour.** With the old expansion, `var b = a; a += x;` left `b` pointing at the old value. With an in-place operator, `b` sees the mutation, because it's the same object. That's the expected behaviour for a builder or an accumulator and a nasty surprise for anything a reader thinks of as a value. Use it for types that are obviously mutable containers (numeric buffers, builders); keep static operators only for value-like types.

---

# Part G — Nullability as a system

## Concept 47 — Nullable reference types: analysis over an erased type system

C# 8's nullable reference types (NRT) did something unusual: they changed the meaning of existing syntax. With nullable enabled, `string` means "not null" and `string?` means "maybe null." But nothing changes at runtime:

- **`string?` and `string` are the same runtime type.** The annotation is recorded in metadata via `[Nullable]` and `[NullableContext]` attributes, which other C# compilations read. No checks are inserted, no exceptions are thrown, reflection sees `System.String` either way.
- **Contrast with `int?`**, which *is* a different runtime type — `Nullable<int>`, a struct with a `HasValue` flag.
- **The feature has two independent switches**: the *annotation* context (does `?` mean anything here?) and the *warning* context (do I get diagnostics?), controlled by `<Nullable>` in the project or `#nullable enable|disable|restore [annotations|warnings]` in source. Code compiled with nullable disabled is *oblivious*, and the analysis trusts it completely.
- **Generics** add a subtlety: unconstrained `T?` (C# 9+) means "the default value of `T` might be returned" — `null` for a reference type, `default` for a value type — not "`Nullable<T>`."

So NRT is a **static analysis with an annotation language**, not a type-system guarantee. That's the right mental model for every other concept in this Part.

---

## Concept 48 — Flow analysis and the `!` operator

The compiler tracks a *null state* for every variable and member access — *not-null* or *maybe-null* — and updates it as control flows:

```csharp
string? name = GetName();       // maybe-null
if (name is null) return;       // after this, not-null
Console.WriteLine(name.Length); // fine

if (string.IsNullOrEmpty(name)) return;   // also works, thanks to [NotNullWhen(false)] (Concept 49)
```

The analysis is deliberately **optimistic** in places where being strict would drown real code in warnings, and knowing where it's unsound is a senior-level signal:

- **Fields aren't invalidated by method calls.** After checking `_customer is not null`, calling `DoSomething()` — which might set `_customer = null` — doesn't reset the state. Multithreaded code gets no help at all.
- **Arrays of non-nullable references start full of nulls.** `new string[10]` has ten nulls in a `string[]`.
- **Structs' `default`** fills reference fields with null regardless of annotations (Concept 20).
- **Oblivious code** — older libraries, code compiled without nullable — is trusted.
- **Reflection, deserialization, and interop** create objects the analysis never saw (Concept 50).

The **null-forgiving operator `!`** (`name!.Length`) tells the analysis "trust me, this isn't null." It generates no code; it only suppresses the diagnostic. It's appropriate where you have knowledge the compiler can't — a test after `Assert.NotNull`, a framework that guarantees initialization — and inappropriate as a way to make warnings go away. `= null!` and `= default!` on fields are the most common misuse: they tell the compiler a non-nullable field is initialized when it isn't. Since C# 11, `required` (Concept 9) is almost always the honest replacement.

A useful team practice: treat the count of `!` in a codebase as a metric. It should be small, and each one should be explainable.

---

## Concept 49 — The nullable attributes: teaching the analysis your contracts

Annotations on types (`string?`) can't express conditional contracts — "the out parameter is non-null when the method returns true." The `System.Diagnostics.CodeAnalysis` attributes can:

| Attribute | Meaning | Typical use |
|---|---|---|
| `[NotNullWhen(true)]` / `[NotNullWhen(false)]` | The argument is non-null if the method returns true/false | `TryGetValue(key, [NotNullWhen(true)] out TValue? value)`, `IsNullOrEmpty([NotNullWhen(false)] string? s)` |
| `[MaybeNullWhen(false)]` | The out value may be null (default) when the method returns false | Generic `TryGet<T>` where `T` may be non-nullable |
| `[NotNullIfNotNull(nameof(input))]` | The return is non-null if that argument was non-null | `Normalize(string? input)` |
| `[MemberNotNull(nameof(_field))]` | After this method returns, the named members are non-null | An `Initialize()` helper called from constructors |
| `[MemberNotNullWhen(true, nameof(Value))]` | …when it returns true | `HasValue` properties |
| `[DoesNotReturn]` | This method never returns normally | `ThrowHelper.ThrowNotFound()` |
| `[DoesNotReturnIf(false)]` | Doesn't return if the argument is false | Custom `Assert(bool condition)` |
| `[AllowNull]` / `[DisallowNull]` | Preconditions on inputs that differ from the declared type | A property setter that accepts null and substitutes a default |
| `[MaybeNull]` / `[NotNull]` | Postconditions on outputs that differ from the declared type | Generic methods returning `default(T)`; argument validators |

```csharp
public bool TryFind(OrderId id, [NotNullWhen(true)] out Order? order)
{
    order = _orders.GetValueOrDefault(id);
    return order is not null;
}

if (repo.TryFind(id, out var order))
    Console.WriteLine(order.Total);    // no warning: the attribute told the analysis
```

These attributes are the difference between a codebase where nullability *works* and one full of `!`. Every `Try*` method, every guard helper, and every lazy-initialization helper in shared code should carry them. They're also polyfillable (Concept 3), so libraries targeting `netstandard2.0` can use them.

---

## Concept 50 — Where the compiler's promise ends

Because NRT is compile-time analysis, every place where objects are created or filled *outside* compiled C# is a boundary where the promise stops. The important ones:

- **Deserialization.** `System.Text.Json` historically created objects with `null` in non-nullable properties when the JSON omitted them or sent `null`. Two controls exist: `required` members (and `[JsonRequired]`) make missing properties an error; and since .NET 9, `JsonSerializerOptions.RespectNullableAnnotations` makes the serializer honour non-nullable annotations. Know which your services use.
- **EF Core** uses NRT as *model configuration*: with nullable enabled, a non-nullable `string` property maps to a `NOT NULL` column. That's valuable — and it means flipping `<Nullable>enable</Nullable>` on an existing EF model can generate a migration that changes column nullability. Review the migration.
- **Configuration binding.** An options class with non-nullable properties binds happily from configuration that's missing a key, leaving `null`. Validate with data annotations or a validator, and call `ValidateOnStart()` so a bad deployment fails at startup rather than on first use.
- **Reflection, `Activator`, mappers, and interop** construct objects the analysis never sees.
- **Oblivious libraries** return values the analysis assumes are fine.

The principle that ties this Part back to Part E: **parse at the boundary, trust inside.** Validate — or better, *convert into a type that can only hold valid values* — at the edges where data enters, and let the annotations mean something within the core. A codebase that validates the same non-null property in five layers has usually failed to decide where its boundary is.

---

## Concept 51 — Adopting nullability, and the null-operator family

**Adoption** in a large existing codebase is an engineering project, not a flag flip:

1. **New projects**: `<Nullable>enable</Nullable>` and `<WarningsAsErrors>nullable</WarningsAsErrors>` from day one. There's no good argument for anything else.
2. **Existing code, bottom-up**: annotate leaf libraries first (utilities, domain types), then the layers that depend on them, so each layer consumes already-annotated APIs.
3. **File by file** with `#nullable enable` at the top, or project-wide with warnings disabled at first (`annotations` context only) so annotations can land without a wall of warnings.
4. **Ratchet**: once a project is clean, make nullable warnings errors so it stays clean. Track `!` usage.
5. **Never** end in the state of "nullable enabled, three thousand warnings, all ignored" — it's worse than disabled, because the annotations now lie.

**The null-operator family**, for completeness, since interviewers like to probe it:

| Operator | Meaning | Since |
|---|---|---|
| `a?.B`, `a?[i]` | Null-conditional access: `null` if `a` is null | C# 6 |
| `a ?? b` | Null-coalescing | C# 2 |
| `a ??= b` | Assign if null | C# 8 |
| `a!` | Null-forgiving (analysis only) | C# 8 |
| `x is null` / `is not null` | Null patterns (Concept 30) | C# 7 / 9 |
| `a?.B = c`, `a?.Total += x` | **Null-conditional assignment** — the right side is evaluated only if `a` isn't null | **C# 14** |

C# 14's null-conditional assignment replaces the `if (customer is not null) customer.Order = GetOrder();` idiom with `customer?.Order = GetOrder();` — note that `GetOrder()` isn't called when `customer` is null. It works with compound assignment but not with `++`/`--`.

And the **guard helpers** that replaced the abandoned `!!` syntax (Concept 6): `ArgumentNullException.ThrowIfNull(arg)` (.NET 6), `ArgumentException.ThrowIfNullOrEmpty` and `ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegative`/`ThrowIfZero`/`ThrowIfGreaterThan` (.NET 8), and `ObjectDisposedException.ThrowIf`. They use `[CallerArgumentExpression]` to capture the argument's name, and they carry the nullable attributes that tell flow analysis the argument is non-null afterwards.

---

# Part H — Performance-shaped language features

## Concept 52 — The ref-safety family, from the language side

Module 14 covered the runtime mechanics of `readonly struct` and defensive copies (Concept 16), `Span<T>` and the ref-struct rules (Concept 19), and ref fields and `scoped` (Concept 20). Here is the language-design view, which is what makes them hang together.

Every feature in this family exists for one reason: **to let the BCL expose zero-copy, zero-allocation APIs that are still memory-safe**, with the safety proof done by the compiler rather than by the programmer.

| Feature | C# | What it lets you say | What the compiler proves |
|---|---|---|---|
| `in` parameters, `ref readonly` returns | 7.2 | "Pass this large struct by reference, but don't let the callee modify it" | No writes through the reference |
| `readonly struct`, `readonly` members | 7.2 / 8 | "This type (or method) never mutates `this`" | No defensive copies needed |
| `ref struct` | 7.2 | "This type may hold references into the stack" | It never escapes to the heap: no boxing, no fields in classes, no capture in lambdas |
| `ref` fields, `scoped`, `[UnscopedRef]` | 11 | "This ref struct holds a reference to something" (how `Span<T>` is now written) | The reference never outlives what it points to |
| `ref readonly` parameters | 12 | "Take an lvalue by reference, read-only" | Callers pass variables, not temporaries |
| Ref locals and ref structs in `async`/iterators | 13 | Use spans in async methods | They don't live across an `await` or `yield` |
| `allows ref struct` | 13 | Generic code over ref structs | Concept 53 |

In application code you mostly **consume** this family — you call span-based APIs, and you use `ReadOnlySpan<char>` for parsing — and you rarely **author** ref structs. That's the right balance, and saying so is itself a signal: authoring ref structs is for library and infrastructure code with a measured need.

Two newer directions worth knowing. C# 13's relaxation that allows ref structs as *locals* in async methods removed the most common reason people copied spans into arrays. And C# 15's memory-safety preview starts redefining `unsafe` itself: pointer *types* and `&`/`fixed` no longer need an unsafe context, while *dereferencing* still does; `unsafe` on a member becomes a *caller obligation* ("requires-unsafe"); and a `safe` keyword marks audited boundaries. The direction is clear — `unsafe` is becoming an auditable contract that flows to callers, rather than a syntax marker for "pointers live here" — and it's explicitly a preview that may change before C# 16.

---

## Concept 53 — `allows ref struct` (C# 13) and the alternate-lookup payoff

Before C# 13, a generic type parameter could never be a ref struct: generic code might box `T`, store it in a field, or capture it in a lambda, all of which would break ref safety. So `Func<ReadOnlySpan<char>, int>` was illegal, and any API that wanted to be generic over "a string or a span" had to be written twice.

C# 13 adds an **anti-constraint**: `where T : allows ref struct` says "`T` may be a ref struct, and I promise my generic code obeys ref-struct rules for `T`." The compiler then checks the generic code against those rules. Alongside it, **ref structs may implement interfaces** — though they still can't be *converted* to the interface (that would box), so the interface is usable only through a generic parameter constrained with `allows ref struct`.

The flagship payoff arrived in .NET 9: **alternate lookup**. A `Dictionary<string, TValue>` can be queried with a `ReadOnlySpan<char>` key, without allocating a string:

```csharp
var counts = new Dictionary<string, int>(StringComparer.Ordinal);
var lookup = counts.GetAlternateLookup<ReadOnlySpan<char>>();

ReadOnlySpan<char> text = line.AsSpan();
foreach (Range r in text.Split(' '))             // .NET 9 span splitting: no substrings
{
    ReadOnlySpan<char> word = text[r];
    lookup[word] = lookup.TryGetValue(word, out int n) ? n + 1 : 1;
    // a string is allocated only the first time a new word is inserted
}
```

This works because the comparer implements `IAlternateEqualityComparer<ReadOnlySpan<char>, string>` — an interface whose `TAlternate` parameter is declared `allows ref struct`. The same mechanism exists on `HashSet<T>`, `ConcurrentDictionary`, and the frozen collections. For a parser, tokenizer, or header lookup running millions of times, it removes the single largest source of garbage: the substring you only needed for a lookup.

This is a good example to use in an interview when asked "what's the point of these low-level features?" — it connects a language feature nobody writes directly to a concrete allocation you can eliminate in a hot path.

---

## Concept 54 — First-class spans (C# 14): simpler BCL, changed binding

Before C# 14, the conversions between `T[]`, `Span<T>`, `ReadOnlySpan<T>`, and `string`→`ReadOnlySpan<char>` were *user-defined* implicit operators on the span types. User-defined conversions don't apply to extension-method receivers and don't participate in generic type inference, so the BCL had to ship array, `Span<T>`, and `ReadOnlySpan<T>` overloads of the same helper, and `array.StartsWith(1)` didn't compile against a span-based extension.

C# 14 makes those conversions **built-in language conversions**. Now one `ReadOnlySpan<T>` overload serves arrays, spans, and (for `char`) strings; span-based extension methods apply to arrays; and inference works through them. That's a real simplification of the platform.

It is also the clearest recent example of a **language upgrade changing which method binds, with no source change**:

- **`array.Reverse()`** used to bind to LINQ's `Enumerable.Reverse` (returns a reversed sequence). With first-class spans, `MemoryExtensions.Reverse(Span<T>)` became applicable and preferred — and it reverses *in place* and returns `void`. .NET 10 added an array-specific `Enumerable.Reverse(T[])` overload so existing code keeps its meaning; projects using C# 14 against an *older* target framework (an unsupported configuration) still hit the break.
- **Covariant arrays**: if a `string[]` is passed where the static type is `object[]`, and a `Span<T>` overload now wins over an `IEnumerable<T>` overload, the span constructor throws `ArrayTypeMismatchException` at runtime. Overload resolution prefers `ReadOnlySpan<T>` over `Span<T>` precisely to reduce this.
- **Expression trees**: `array.Contains(x)` inside an `Expression<Func<…>>` now binds to `MemoryExtensions.Contains`, which involves a ref struct. The expression interpreter can't handle that, and LINQ providers had to add translation for it — EF Core rewrites span-based calls back to `Enumerable` equivalents, and other query providers had to follow.

The lesson isn't about spans. It's that **overload resolution is part of your program's semantics, and the language owns overload resolution.** That's why a compiler upgrade is a behavioural change deserving a full test pass, why the compiler breaking-changes page is required reading before an upgrade, and why library authors now have `[OverloadResolutionPriority]` (C# 13) to steer binding deliberately.

---

## Concept 55 — `params` collections (C# 13)

`params` used to mean "a `T[]` allocated at every call." C# 13 lets `params` apply to any collection type that collection expressions can build — including `ReadOnlySpan<T>`, `Span<T>`, `IEnumerable<T>`, `List<T>`, and interfaces:

```csharp
public static string JoinPath(params ReadOnlySpan<string> parts) { /* ... */ }

JoinPath("usr", "local", "bin");    // arguments can be placed in an inline array on the stack — no heap allocation
```

For `params ReadOnlySpan<T>`, the compiler can store the arguments in a stack-allocated inline array (Module 14, Concept 49), so a variadic call allocates nothing. .NET 9 added `params ReadOnlySpan<T>` overloads across the BCL — `string.Concat`, `string.Join`, `Path.Combine`, `Task.WhenAll`, and many more — and overload resolution prefers them. The consequence is the pleasant mirror image of Concept 54: **recompiling existing code against .NET 9 or later silently removed allocations** at many call sites, because the same source now binds to the span overload.

The API-design guidance for library authors: if you'd have written `params T[]`, write `params ReadOnlySpan<T>` (and keep the array overload only for binary compatibility), unless you need to retain the arguments beyond the call — a span can't be stored.

---

## Concept 56 — Collection expressions: one syntax, many lowerings

C# 12's collection expressions give every collection one construction syntax:

```csharp
int[] a = [1, 2, 3];
List<int> b = [.. a, 4, 5];
ReadOnlySpan<byte> header = [0x50, 0x4B, 0x03, 0x04];
ImmutableArray<string> names = ["ada", "grace"];
IReadOnlyList<Order> none = [];
HashSet<string> tags = [with(StringComparer.OrdinalIgnoreCase), "a", "A"];   // C# 15: constructor arguments
List<Order> reserved = [with(capacity: expected), .. seed];                   // C# 15
```

The key word is **target-typed**: a collection expression has no natural type (`var x = [1, 2];` is an error), and the compiler chooses a construction strategy based on the target:

- **Arrays**: allocate the exact length and fill.
- **`Span<T>` / `ReadOnlySpan<T>`**: constant data can come straight from the assembly's static data; otherwise an inline array on the stack. No heap allocation.
- **`List<T>`**: pre-sized to the known count when the compiler can compute it, then filled directly.
- **`IEnumerable<T>`, `IReadOnlyCollection<T>`, `IReadOnlyList<T>`**: the compiler may use an array or a compiler-synthesized read-only type — you're promised only the interface, so **don't cast the result back to `List<T>`** or depend on the concrete type.
- **`ICollection<T>`, `IList<T>`** (mutable interfaces): a `List<T>`.
- **Empty `[]`**: a cached empty instance where possible (`Array.Empty<T>()` for arrays and read-only interfaces).
- **Custom and immutable types**: via `[CollectionBuilder]`, which names a static factory taking a `ReadOnlySpan<T>` — which is how `ImmutableArray<T>` and friends support the syntax without an intermediate list.
- **Spreads (`..x`)**: enumerate `x`; if `x` is countable, the compiler pre-sizes the result.

C# 15's **`with(...)` element**, written first, forwards arguments to the target's constructor or factory — capacity, comparers, and similar. It exists largely as groundwork for the planned *dictionary expressions*, where specifying a comparer is routine.

One review point: IDE analyzers suggest converting existing initializations to collection expressions, and most of those conversions are pure wins. Be deliberate where the target is an *interface* and something downstream depends on the concrete type, or where the old code intentionally returned a mutable `List<T>` through an `IEnumerable<T>` signature that callers then cast.

---

## Concept 57 — Generic math and static abstract interface members

C# 11 and .NET 7 added `static abstract` (and `static virtual`) members to interfaces. An interface can now require that implementing *types* provide static methods, properties, and operators, and generic code calls them through the type parameter:

```csharp
static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    T total = T.Zero;
    foreach (T v in values) total += v;
    return total;
}

Sum<int>([1, 2, 3]);         // specialized code for int: no boxing, operators inlined
Sum<decimal>([1.5m, 2.5m]);
```

This required runtime support (Concept 3): the call to `T.Zero` or `+` is a *constrained call* resolved per type argument, and because the JIT generates specialized code for each value-type instantiation (Module 14, Concept 5), `Sum<int>` compiles down to a plain integer loop. The BCL's numeric types implement a family of interfaces — `INumber<T>`, `IAdditionOperators<TSelf, TOther, TResult>`, `IBinaryInteger<T>`, `IFloatingPoint<T>` — and parsing and formatting got the same treatment: `IParsable<T>`, `ISpanParsable<T>`, `IUtf8SpanFormattable`.

In **application code**, you'll rarely write numeric generics. You *will* use the mechanism in two places:

- **`IParsable<T>` on your own types** — strongly-typed IDs, money, codes — so framework binding (minimal API routes, query strings) and generic parsing utilities can construct them (Concept 22, worked example 5).
- **Static abstract factories** in generic infrastructure: `interface IMessage<TSelf> where TSelf : IMessage<TSelf> { static abstract string Topic { get; } }` lets a generic publisher find a message type's topic without reflection or attributes.

The self-referencing `TSelf : IFoo<TSelf>` constraint looks odd the first time; it's how an interface says "the implementing type itself" in a static context. The cost is complexity — compiler errors in heavily constrained generic code are hard to read — which is why this belongs in libraries and infrastructure, not in everyday business logic.

---

## Concept 58 — Strings: literals, handlers, and logging

**Raw string literals** (C# 11) — `"""…"""` — contain anything, including quotes and backslashes, without escaping, and strip the indentation of the closing delimiter. They're ideal for JSON, SQL, regex, and test fixtures. For interpolation, the number of `$` sets how many braces open a hole: in `$$"""{ "id": {{id}} }"""`, single braces are literal.

**UTF-8 string literals** (C# 11) — `"HTTP/1.1"u8` — produce a `ReadOnlySpan<byte>` pointing at constant data in the assembly, with no allocation and no transcoding at runtime. Useful wherever you write bytes to a pipe or compare protocol tokens.

**Interpolated string handlers** (C# 10) changed what `$"..."` compiles to. Instead of `string.Format` (which boxes value types and parses a format string at runtime), the compiler emits calls to `DefaultInterpolatedStringHandler` — appending literals and formatting values through `ISpanFormattable` into a pooled buffer. More importantly, APIs can declare their *own* handler type, and the handler's constructor can decline to format at all:

- `Debug.Assert(condition, $"…")` doesn't format the message when the condition is true.
- `StringBuilder.Append($"…")` appends directly into the builder with no intermediate string.
- `destination.TryWrite($"…", out int written)` formats straight into a `Span<char>`.

**Which brings us to the logging trap**, one of the most common review comments in .NET codebases:

```csharp
logger.LogInformation($"Order {orderId} placed by {customerId}");   // CA2254
```

The allocation is the lesser problem. The real damage is that the interpolation **destroys structured logging**: the logging provider receives a finished string, not a template with named properties, so `orderId` and `customerId` never reach your log backend as queryable fields, and every message is unique text (bad for grouping and cardinality). The fix is a message template — `logger.LogInformation("Order {OrderId} placed by {CustomerId}", orderId, customerId)` — or, better for hot paths, a **`[LoggerMessage]` source-generated method** (Concept 14), which is strongly typed, checks `IsEnabled` before doing any work, and avoids boxing the arguments.

---

# Part I — Compile-time metaprogramming

## Concept 59 — Source generators: moving reflection to compile time

A source generator is a compiler plug-in that inspects your code during compilation and **adds** new source files to it. It cannot modify your code (interceptors, Concept 60, are the narrow exception). Generators replace runtime reflection — discovering types, reading attributes, emitting IL, building serializers — with code generated at build time that the compiler checks, the JIT optimizes, and the trimmer understands.

The BCL now leans on them heavily:

| Generator | Replaces |
|---|---|
| `System.Text.Json` source generation (`JsonSerializerContext`) | Reflection-based serializer metadata |
| `[GeneratedRegex]` | Runtime regex compilation (`RegexOptions.Compiled` emitting IL) |
| `[LoggerMessage]` | `LoggerMessage.Define` boilerplate, boxing, template parsing |
| `[LibraryImport]` | Runtime-generated P/Invoke marshalling stubs |
| Configuration binding generator | Reflection-based `IConfiguration.Bind`/`Get<T>` |
| Options validation generator | Reflection-based data-annotation validation |
| ASP.NET Core Request Delegate Generator | Runtime-compiled minimal-API delegates |
| `[GeneratedComInterface]` | Runtime COM interop stubs |

Writing one well requires understanding the **incremental** model. An `IIncrementalGenerator` declares a pipeline — "find classes with this attribute, extract a model, generate code from the model" — and the compiler caches each stage, re-running downstream stages only when an upstream output *changes*. "Changes" is decided by **equality of the models**. This is where Part C comes back:

- Pipeline models should be small, immutable **value types** — records are the natural choice.
- A record containing an array or `List<T>` compares by reference (Concept 19), so the cache never hits and the generator re-runs on every keystroke in the IDE. The fix is an equatable collection wrapper, which is why `EquatableArray<T>` appears in so many generator codebases.
- Models must never hold `ISymbol`, `SyntaxNode`, or `Compilation` objects — they're large, they don't compare meaningfully across compilations, and holding them keeps old compilations alive.
- `ForAttributeWithMetadataName` is the much more efficient entry point for attribute-driven generators than general syntax scanning.

The older `ISourceGenerator` API is deprecated in favour of incremental generators. The review question for any generator a team adopts: *can a developer find, read, and step through the generated code?* (Setting `EmitCompilerGeneratedFiles` writes it to disk; IDEs show it under the analyzer node.)

---

## Concept 60 — Interceptors

Generators can only *add* code, which leaves a gap: a library can generate a fast, reflection-free implementation, but it can't make *your existing call* — `app.MapGet(...)`, `config.Get<MyOptions>()` — use it. **Interceptors** close that gap. A generated method carries an `[InterceptsLocation]` attribute identifying a specific call site (a version number plus an opaque encoded checksum-and-position), and the compiler substitutes a call to the interceptor at that location during lowering.

Status and rules worth knowing:

- Interceptors shipped experimentally in .NET 8 and have been a **stable feature since the .NET 9.0.2xx SDK**.
- A project opts in by listing allowed namespaces in `<InterceptorsNamespaces>` (the older `<InterceptorsPreviewNamespaces>` is an alias). Well-behaved generators only put interceptors in namespaces they own.
- They can intercept only ordinary method calls — not constructors, properties, operators, or delegate invocations.
- Roslyn provides APIs so generators compute locations correctly and analyzers can tell whether a call is being intercepted.

Who uses them: the ASP.NET Core Request Delegate Generator and the configuration-binding generator (both central to Native AOT), and EF Core's experimental query precompilation. **You will consume interceptors far more often than you write one**, and the honest interview position is that writing interceptors is for framework authors: they change the meaning of code at a distance, which is exactly the property you don't want in application code. What you should know as a consumer is that when a generator is enabled, the call you see in source may not be the call that executes — which matters when you're debugging.

---

## Concept 61 — Analyzers as the enforcement layer

Everything in this module that is a *team decision* — which features to use, which styles to prefer, which traps to ban — needs an enforcement mechanism, or it decays into review-comment folklore. In .NET, that mechanism is analyzers running in the build:

- **`.editorconfig`** sets severities for code-style rules (`dotnet_diagnostic.IDE0290.severity = warning` to require primary constructors, `= none` to disable the suggestion) and naming rules. It is the style contract, checked into the repository.
- **`<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>`** makes IDE-style rules run in CI builds, not just in the editor.
- **`<AnalysisLevel>latest-recommended</AnalysisLevel>`** (or a pinned version) chooses the set of quality rules enabled by default.
- **`<TreatWarningsAsErrors>`**, or the narrower `<WarningsAsErrors>nullable;CS8509;CS8524</WarningsAsErrors>`, turns the correctness-relevant diagnostics from this module into build failures.
- **The banned-API analyzer** (`Microsoft.CodeAnalysis.BannedApiAnalyzers`) with a `BannedSymbols.txt` forbids specific APIs — `DateTime.Now` in favour of `TimeProvider` (Module 15, Concept 69), synchronous `Task.Wait`, a deprecated internal library.
- **Third-party packs** such as Roslynator and Meziantou.Analyzer add hundreds of rules, several of which catch exactly the traps in this module.
- **Architecture tests** (NetArchTest, ArchUnitNET) enforce layering rules — "the domain assembly doesn't reference EF Core" — as unit tests (Module 20).
- **Custom analyzers**, for rules specific to your codebase, are cheaper to write than most teams assume.

The architect-level point: **code review is the most expensive place to enforce a rule and the least reliable.** Every rule that can be an analyzer should be one, leaving human review for design.

---

## Concept 62 — AOT and trimming: why the language is moving this way

Native AOT compiles an application ahead of time into a single native executable with no JIT; the trimmer removes unreferenced code to shrink it (Module 14, Concepts 7 and 64; Module 17 in depth). Both need to see, *at build time*, every type and member the program will use. Reflection breaks that: `Type.GetType(name)`, `Activator.CreateInstance`, reflection-based serializers, and runtime IL emission discover code the tools can't see or can't compile.

That single constraint explains the direction of the last several C# releases:

- **Source generators** (Concept 59) replace reflection-based discovery.
- **Interceptors** (Concept 60) let generators redirect existing calls to AOT-safe implementations.
- **Partial properties and constructors** (Concept 14) make generator output fit cleanly into user types.
- **Static abstract interface members** (Concept 57) replace attribute-plus-reflection conventions ("find a static `Topic` property") with compile-time contracts.
- **Annotations** — `[RequiresUnreferencedCode]`, `[RequiresDynamicCode]`, `[DynamicallyAccessedMembers]` — let libraries declare their reflection needs, and `<IsAotCompatible>true</IsAotCompatible>` turns on the analyzers that flag violations.

When an interviewer asks "why does modern .NET generate so much code?", this is the answer: *startup time, memory, deployment size, and AOT compatibility all require that the program be knowable at build time, and the language has been steadily providing the tools to make it so.*

---

# Part J — Program shape, small features, and what fluency signals

The last part of the module has two halves. Concepts 63–65 finish the feature survey with the pieces that shape *every* file you write — program structure, the small features that make code read as current, and interface evolution. Concepts 66–70 step back and ask the interview question directly: what does fluency in all of this actually signal, how do you show it without showing off, and what does an architect *decide* about the language on behalf of a team?

## Concept 63 — Program shape: the ceremony that disappeared

Four features, spread over C# 9, C# 10, and .NET 10, changed what a C# file looks like before you've written a line of logic. None of them changes semantics; all of them change what a reader sees first.

**Top-level statements (C# 9).** One file per project may contain statements outside any type. The compiler wraps them in a synthesized class — named `Program` — with an entry-point method whose signature it infers from the body: `await` anywhere makes it `async Task`, a `return 1;` makes it return `int`, and `args` is in scope as an implicit `string[]`. Local functions declared among the statements are local functions of that synthesized method; types must be declared after the statements (or in other files).

```csharp
var builder = WebApplication.CreateBuilder(args);   // `args` is implicit
var app = builder.Build();
app.MapGet("/", () => "ok");
await app.RunAsync();                               // → the entry point becomes async Task
```

Two consequences matter in practice. First, the synthesized `Program` class is `internal`, which historically broke integration tests using `WebApplicationFactory<Program>` — the fix was a hand-written `public partial class Program { }` or `InternalsVisibleTo`. Since .NET 10, ASP.NET Core ships a source generator that emits that declaration for apps using top-level statements, so on current targets the workaround is unnecessary; on older targets you still write it. Second, top-level statements are for *entry points*: a composition root, a script, a small tool. They are not a place to grow business logic, and a 400-line `Program.cs` is a smell regardless of syntax.

**File-scoped namespaces (C# 10).** `namespace Orders.Domain;` applies to the whole file and removes one level of indentation from every line in it. There is no semantic difference from the block form; the only constraint is one namespace per file, which is what almost everyone wanted anyway. In a modern codebase, the block form is the thing a reviewer notices — and the right response is an `.editorconfig` rule (`csharp_style_namespace_declarations = file_scoped:warning`), not a comment.

**Global usings and implicit usings (C# 10 / .NET 6).** `global using System.Text.Json;` in any file applies to every file in the compilation. `<ImplicitUsings>enable</ImplicitUsings>` makes the SDK generate a file of global usings for you (look in `obj/…/<Project>.GlobalUsings.g.cs`); the set depends on the SDK — the Web SDK adds the ASP.NET Core namespaces. You can add and remove entries declaratively:

```xml
<ItemGroup>
  <Using Include="System.Collections.Immutable" />
  <Using Include="Orders.Domain.OrderId" Alias="OrderId" />
  <Using Remove="System.Net.Http" />
</ItemGroup>
```

The trade-off is discoverability. Global usings make individual files shorter and make it less obvious where a name comes from — and an ambiguity between two globally imported namespaces (two `Result` types, say) is now a whole-project problem. The team rule that works: implicit usings on, a single `GlobalUsings.cs` (or the `<Using>` items) for project-wide additions, and no scattering of `global using` across random files.

**File-based apps (.NET 10).** `dotnet run app.cs` builds and runs a single C# file with no project file. File-level directives stand in for the `.csproj`:

```csharp
#!/usr/bin/env dotnet
#:package Humanizer@2.14.1
#:property LangVersion=preview
#:sdk Microsoft.NET.Sdk.Web

using Humanizer;
Console.WriteLine(TimeSpan.FromMinutes(90).Humanize());
```

The supported directives are `#:package`, `#:sdk`, `#:property`, `#:project` (reference another project), and — from .NET 11 Preview 3 and the 10.0.300 SDK — `#:include` for pulling in more files. A shebang line makes the file directly executable on Unix-like systems. `dotnet build`, `dotnet publish`, and `dotnet pack` work on the file too; file-based apps publish as Native AOT by default (opt out with `#:property PublishAot=false`). When a script outgrows the format, `dotnet project convert app.cs` produces an ordinary project.

The architectural relevance is small but real: file-based apps give C# a credible scripting story for build tooling, migration scripts, one-off data fixes, and operational runbooks — territory that used to go to PowerShell or Python by default. For an interview it is also a practical tool: a single file you can run with no ceremony is the fastest way to test a language-semantics question ("does `with` run my constructor?") in front of someone, or to prepare an example beforehand.

---

## Concept 64 — Small features worth recognizing

These don't deserve a concept each, but each is either a readability win you should use by default or a construct you must be able to *read* without hesitation when an interviewer's code contains it.

| Feature | Version | What it is | The judgement |
|---|---|---|---|
| Target-typed `new()` | C# 9 | `List<Order> orders = new();` — the type comes from the target | Use when the type is visible on the same line (fields, declarations with an explicit type). `var x = new Foo()` and `Foo x = new()` are both fine; pick one per codebase via `.editorconfig`. Avoid in arguments, where the type is invisible: `Process(new())` tells a reader nothing |
| `static` lambdas | C# 9 | `static x => x * 2` — the lambda may not capture | A cheap guarantee that a hot-path lambda allocates no closure (Module 14, Concept 51). Worth it in libraries and hot loops; noise elsewhere |
| Lambda natural types, attributes, explicit return types | C# 10 | `var parse = (string s) => int.Parse(s);` infers `Func<string, int>` | Mostly matters for minimal APIs, where a lambda *is* an endpoint and its attributes and return type drive metadata |
| Lambda default parameters and `params` | C# 12 | `var greet = (string name = "world") => $"hi {name}";` | Rare in application code; recognize it |
| Lambda parameter modifiers without types | C# 14 | Given `delegate bool TryParse<T>(string s, out T r);`, write `TryParse<int> p = (text, out result) => int.TryParse(text, out result);` | Removes the old rule that any modifier (`ref`, `out`, `in`, `scoped`) forced you to spell every parameter type |
| `file`-local types | C# 11 | `file sealed class Helper` — visible only in its source file | Designed for source generators, which must emit helper types without colliding with yours. In hand-written code, a narrower-than-`private` scope for a file's private helper type |
| Generic attributes | C# 11 | `[ValueObject<Guid>]` instead of `[ValueObject(typeof(Guid))]` | Type-safe attribute arguments; you'll see it in generator-driven libraries (Vogen, worked example 5) |
| `nameof` on parameters in attributes | C# 11 | `[return: NotNullIfNotNull(nameof(input))]` | Makes the nullable attributes (Concept 49) refactor-safe |
| Alias any type | C# 12 | `using Point = (int X, int Y);` and `using unsafe Ptr = int*;` | Useful for naming a tuple shape used across one file. If the alias leaks into many files, it wanted to be a `record struct` |
| `ref readonly` parameters | C# 12 | A by-reference parameter that can't be written through, warning when callers pass an rvalue | Library-author feature for large structs; complements `in` (Module 14, Concept 13) |
| `\e` escape; `^` in object initializers | C# 13 | `"\e[1m"` for ANSI escape; `new Countdown { Buffer = { [^1] = 0 } }` | Recognize them |
| `[OverloadResolutionPriority]` | C# 13 | Library authors rank overloads so a new, better one wins without breaking callers | The tool the BCL uses to steer binding after changes like Concept 54's. You'll almost never apply it; you should know why it exists |
| `nameof` on unbound generics | C# 14 | `nameof(List<>)` → `"List"` | Removes the old `nameof(List<int>)` workaround in logging and diagnostics |
| Labeled `break` and `continue` | C# 15 | `outer: for (...) { for (...) { if (done) break outer; } }` | Replaces the boolean-flag and `goto` workarounds for nested loops. The label must sit directly on the loop or `switch` it names, and `continue` can only target a loop. Use sparingly — when three levels of nesting need it, extracting a method is often still clearer |

The labeled-jump example is worth seeing once in full, because it's the newest item in the table and an interviewer on a C# 15 codebase may use it:

```csharp
// before: a flag, checked at every level
bool found = false;
for (int x = 0; x < width && !found; x++)
    for (int y = 0; y < height; y++)
        if (grid[x, y] == target) { found = true; break; }

// C# 15: the jump names its target
search:
for (int x = 0; x < width; x++)
{
    for (int y = 0; y < height; y++)
    {
        if (grid[x, y] == target)
            break search;          // leaves both loops
    }
}
```

Two meta-points about this table. First, **most of these are recognition features, not adoption decisions** — the win from knowing them is that code using them doesn't slow you down in a code-reading round. Second, several exist *for tools*: `file` types for generators, generic attributes for generator configuration, `[OverloadResolutionPriority]` for the BCL. That pattern — language features whose primary customer is a library or a generator — is Concept 62's direction showing up at small scale.

---

## Concept 65 — Interface evolution: default members and static contracts

Interfaces gained two very different capabilities after C# 7, and both are frequently misdescribed.

**Default interface methods (C# 8).** An interface member can have a body; implementing types that don't provide the member inherit the default:

```csharp
public interface IClock
{
    DateTimeOffset UtcNow { get; }

    // added in v2 — existing implementers keep compiling and keep working
    DateOnly Today => DateOnly.FromDateTime(UtcNow.UtcDateTime);
}
```

What they are *for* is **API versioning**: adding a member to a published interface without breaking every implementer. Before C# 8, adding a member to an interface was a breaking change for any third party implementing it, which is why the Framework Design Guidelines long recommended abstract base classes for extensibility points you expected to evolve.

What they are *not* is traits or mixins, and the traps follow from that:

- **They need runtime support.** DIMs are one of the few features in this module the compiler cannot lower to older IL — the runtime must resolve interface calls to interface-provided bodies. They're unavailable on .NET Framework and `netstandard2.0` (Concept 3).
- **The default is only reachable through the interface.** `clock.Today` on a variable typed as the concrete class doesn't compile unless the class declares its own `Today`. The member belongs to the interface, not the implementer.
- **No instance state.** Interfaces still can't declare instance fields, so a default body can only compose other interface members.
- **Diamonds are resolved by "most specific implementation."** If two interfaces a class implements both provide a default for the same inherited member and neither is more specific, the class must implement it — and where the ambiguity only appears at runtime (a type loaded against a newer interface version), the call throws.
- **`ref struct` implementers must implement every member explicitly** — they can't use a default, because calling the default would require boxing the `ref struct` into the interface (C# 13 allowed `ref struct`s to implement interfaces at all).

**Static abstract and static virtual members (C# 11).** Concept 57 covered them as the engine of generic math; the design point here is that they give interfaces a **compile-time contract over static members** — factories, parsers, constants, operators — resolved through a type parameter at compile time, with no runtime dispatch. `IParsable<TSelf>` is the example you'll use most, and it's exactly what lets framework code construct *your* types without reflection (worked example 5). One consequence to know: an interface with static abstract members can't be used as an ordinary type argument (the compiler reports CS8920), because there'd be no implementation to call.

**C# 15.** The .NET 11 RC1 notes also list a change to non-virtual static interface members; read the C# 15 what's-new page for its exact scope before relying on it in code.

The senior-level summary: **default members solve a versioning problem for interface authors; static abstracts solve a genericity problem for algorithm authors.** Neither is a reason to put behaviour in interfaces that belonged in a class.

---

## Concept 66 — What fluency signals

Step back from the features. When an interviewer probes modern C#, what are they actually trying to learn? Three things, in roughly this order of how often they're tested:

1. **Currency.** Do you know what the language is *now*? Not every feature — but whether you write `required` properties and switch expressions or null-check boilerplate and `if`/`else` ladders, whether you know which version is current and which is LTS, whether you've heard of the `field` keyword and unions. Currency is a proxy for something the interviewer cares about more: *does this person keep learning, or did they stop in 2018?* It's the cheapest signal to send and the most damaging to be missing.
2. **Cost-awareness.** Do you know what features *cost*? That a record compares every field, that `with` skips the constructor, that a union boxes value types, that a switch over a property assumes the getter is pure, that `$"..."` in a log call allocates even when the level is disabled, that upgrading the compiler can change overload binding. This is what separates senior from mid-level, and it's why this module keeps asking "what does it lower to?"
3. **Taste.** Do you choose well? A primary constructor for dependency capture but not for a type with invariants; a closed hierarchy for a domain state machine but a tolerant reader at the wire; a record for a DTO but never for an EF entity. Taste is judgement under trade-offs, which is the thing the whole interview loop is ultimately measuring.

A useful self-test: for any feature in this module, can you give **one sentence on what it's for, one sentence on what it costs or where it stops, and one situation where you wouldn't use it?** If yes, you're fluent in it. If you can only give the first sentence, you've memorized it.

And one honesty rule that is itself a signal: if C# 15 unions are two months old and you haven't used them in production, say so — "I've worked through the spec and built experiments, here's what I'd watch for" is a stronger answer than bluffing production experience with a feature that only just shipped. Interviewers calibrate on whether your confidence tracks your knowledge.

---

## Concept 67 — Feature tourism: the anti-signal

The failure mode of a candidate who has read a "what's new" post is **feature tourism**: using a construct because it exists, not because it makes this code better. It's legible to an experienced reviewer, and it reads as insecurity rather than skill. The common forms:

- **Pattern-matching gymnastics.** A switch expression with nested property patterns four levels deep, `and`/`or`/`not` combinators, and relational patterns, replacing three readable `if` statements. Patterns are wonderful for *dispatch on shape*; they're a poor way to write arbitrary boolean logic.
- **Records everywhere.** Entities, services, and mutable models declared as records because "records are modern," dragging value equality into places where identity was the point (Concept 21).
- **Primary constructors on everything.** Including types with invariants, where the captured parameter is mutable and unvalidated (Concept 13) — often introduced wholesale by accepting an IDE refactoring (IDE0290) across a solution.
- **Extension everything.** Business logic hung off `string` and `int` as extension properties, so `"ORD-42".Order` performs a database lookup (Concept 45).
- **Operator soup.** `a?.B?.C ?? d?.E ?? throw new …` chains where every `?.` is a place a null silently became "skip," hiding the question of which of those nulls are actually legitimate.
- **Tuples as public API.** `(bool, string?, int)` return types in public methods, where a named record would document the contract and survive a new field.
- **Clever LINQ.** A single twelve-operator query that a loop would have expressed in eight obvious lines, with a closure allocation per element on a hot path.

The heuristic that separates use from tourism: **a feature earns its place if it deletes a bug class or removes noise the reader would otherwise have to parse; if it adds a concept the reader must learn and removes nothing, it's tourism.** A switch expression over a closed set deletes the "forgot a case" bug class. A `required` property deletes the "forgot to set it" bug class. `ArgumentNullException.ThrowIfNull(x)` removes three lines of noise. A four-level nested pattern replacing clear conditionals deletes nothing.

The second heuristic: **consistency beats novelty.** A codebase where half the files use one style and half use another costs more to read than one that is uniformly a version behind. That's why the enforcement belongs in `.editorconfig` and analyzers (Concept 61) and why "I'd match the codebase's existing conventions and propose a change through the team's style config" is a strong answer in a review round.

Restraint is legible. An interviewer who sees you choose the plain construct deliberately — and say why — learns more about your judgement than from any number of new features.

---
## Concept 68 — Modern C# in a live coding round

Module 36 covers the coding round in full; this concept is only about what the *language* contributes to it. Your DS&A is strong, so the risk in a senior coding round isn't the algorithm — it's that the code reads as dated, or as clever, or that you spend minutes fighting syntax. Five practices cover most of it.

**1. Check the environment in the first minute.** Online editors often lag the current SDK. Ask which .NET version the pad runs, or print `Environment.Version`. If it's older, write to that version without comment — complaining about a missing feature is a small negative; silently adapting is a small positive.

**2. Reach for the constructs that delete bug classes, and narrate each choice in one sentence.** The narration is where the signal is:

```csharp
// "Point is a value — two with the same coordinates are the same point — and it's
//  8 bytes, so a readonly record struct: value equality makes it a dictionary key for free."
public readonly record struct Point(int X, int Y);

// "Shipping is a mapping from a small closed set of conditions, so a switch expression:
//  the order of arms is the business rule, and it reads top to bottom."
static decimal Shipping(Order order) => order switch
{
    { Total: >= 100m }                     => 0m,
    { Destination.Country: not "RS" }      => 25m,
    { Items.Count: > 10 }                  => 5m,
    _                                      => 10m,   // a real default: every other order
};

// "A static readonly table, so the directions aren't re-allocated per call."
private static readonly (int Dx, int Dy)[] Directions = [(0, 1), (1, 0), (0, -1), (-1, 0)];

// "Guard clauses up front, with the BCL helpers — the parameter name comes for free."
static IReadOnlyList<Point> Neighbours(Point p, int width, int height)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(width);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(height);

    List<Point> result = [];
    foreach (var (dx, dy) in Directions)
    {
        var n = new Point(p.X + dx, p.Y + dy);
        if (n.X >= 0 && n.X < width && n.Y >= 0 && n.Y < height) result.Add(n);
    }
    return result;
}
```

Each of those comments is a five-second sentence that demonstrates currency, cost-awareness, and taste at once. Note the discard arm in `Shipping` is *legitimate*: the input isn't a closed set, and "everything else costs 10" is the rule (Concept 29).

**3. Let records carry your tests.** Value equality turns expected-value assertions into one line — `Assert.Equal(new Point(1, 2), result)` — and a couple of quick tests in a live round are a strong senior signal (Module 36).

**4. Know the BCL's modern toolbox for algorithmic work**, so you don't hand-roll it: `PriorityQueue<TElement, TPriority>` (.NET 6) for Dijkstra and scheduling, `Comparer<T>.Create` for custom orderings, `Random.Shared` for thread-safe randomness, `CollectionsMarshal.GetValueRefOrAddDefault` if the interviewer pushes on dictionary performance, `MemoryExtensions` and span slicing for parsing without substring allocations, `StringComparison.Ordinal` and `CultureInfo.InvariantCulture` wherever culture could change behaviour, and `checked` arithmetic when overflow is a plausible edge case.

**5. Talk about features you won't type.** In a timed round, you rarely have time to declare a closed hierarchy or a union properly. Say it instead: *"In production I'd model these outcomes as a closed hierarchy so the compiler finds every switch when we add a case; for time I'll use an enum and a default arm that throws."* That's the design judgement, delivered for free, without spending ten minutes on it.

What not to do: rewrite working code into a newer idiom mid-round, reach for a construct you'd have to look up, or use a feature the interviewer then has to ask about — unless you can explain it crisply, in which case that question is an opportunity rather than a cost.

---

## Concept 69 — The code-review round: a checklist

A code-review round hands you a pull request and asks what you'd say. Almost every trap in this module appears in such exercises, because they're exactly the bugs that pass tests and compile cleanly. Here is the checklist, feature by feature:

| Construct | Look for | Why it matters |
|---|---|---|
| Record with a collection property | `List<T>`, arrays, `Dictionary` inside a record | Equality compares references; two "equal" records aren't; `with` shares the list (Concept 19) |
| Record with a lazily cached property | `field ??=` or a private cache field | The cache joins equality and `with` copies it stale (Concept 11) |
| Validation in a record's constructor | Checks that `with` will bypass | `with` clones and runs `init` accessors, never the constructor (Concept 18) |
| Record used as an EF Core entity | `record` on anything with a key that changes state | Value equality breaks change tracking and identity (Concept 21; Module 19) |
| Primary constructor + field of the same parameter | `private readonly X _x = x;` and a method also using `x` | Two copies of state; CS9124 warns; a mutation of one drifts from the other (Concept 13) |
| Primary constructor on a type with invariants | Parameters used directly by members, never validated | The captured parameter is a mutable, unvalidated hidden field (Concept 12) |
| `required` with `[SetsRequiredMembers]` | A constructor carrying the attribute | An unchecked promise; if the constructor doesn't actually set everything, the compiler won't notice (Concept 9) |
| `init` described as immutability | Mutable collections or nested mutable objects behind `init` | `init` is shallow and compile-time only (Concept 8) |
| Switch with a discard arm over a closed set | `_ =>` where the input is an enum-like closed set | Silences the compiler's help when a case is added (Concept 29) |
| Switch over an enum | No handling of undefined values | Enums aren't closed; `(Status)42` is legal (Concept 28) |
| Pattern on an impure property | `x switch { { Next: … } … }` where a getter has side effects | The decision DAG may read it once, twice, or in a different order (Concept 25) |
| Closed set in a message contract | A union or closed hierarchy deserialized from another service | Binary-incompatible consumers crash on a new case; needs a tolerant reader (Concept 40) |
| Union with value-type cases on a hot path | `union Metric(int, double, long)` in a tight loop | Every value boxes (Concepts 35–36) |
| Extension property doing work | `.IsActive` that queries, `.Count` that enumerates | Property syntax promises cheap and repeatable (Concept 45) |
| The `!` operator | `!` on values from deserializers, config, or reflection | Silences the analysis exactly where the promise ends (Concepts 48, 50) |
| `#nullable disable` or `#pragma warning disable CS86xx` | New suppressions in a PR | Each one is technical debt with an interest rate (Concept 51) |
| Interpolated log messages | `logger.LogInformation($"User {id} logged in")` | Loses structured properties and formats even when disabled (Concept 58) |
| `params T[]` in a new hot API | New variadic signatures | `params ReadOnlySpan<T>` avoids the array allocation (Concept 55) |
| Reflection in a new component | `Activator.CreateInstance`, `Type.GetType(string)` | Blocks trimming and AOT; is there a generator? (Concepts 59, 62) |
| `LangVersion` in a project file | Especially `latest` or `preview` | Unsupported or unstable language on the shipped runtime (Concept 4) |
| Tuple-returning public API | `public (bool, string?) TryX(...)` | Unnamed contract that can't evolve; wants a record (Concept 67) |
| Mixed styles | Block and file-scoped namespaces, `var` and explicit types, in one PR | Consistency is an `.editorconfig` problem, not a review comment (Concept 61) |

Two points about *how* to deliver a review in an interview. **Order by severity**: correctness first (equality bugs, bypassed validation, races, crash-on-new-case), then boundary robustness (nullability lies), then performance (boxing, logging allocations), then style — and say that you're ordering them. And **phrase findings as questions where intent is ambiguous**: "Is `Tags` meant to participate in equality? If so, it needs to be an immutable collection with structural comparison" is more useful, and more senior, than "records with lists are wrong." The last row of the table is its own signal: noticing that a class of comments should be an analyzer rule rather than a review comment is how an architect reviews code.

---

## Concept 70 — Language policy as an architect

At the architect level, the language question shifts from *"do I know this feature?"* to *"what have I decided about the language for forty services and twelve teams, and how is that decision enforced?"* A complete answer has six parts.

**1. Pin the compiler.** A `global.json` at the repository root chooses the SDK — and therefore the compiler — so developers and CI build with the same one:

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

`rollForward` decides how strictly: `latestPatch` for maximum reproducibility, `latestFeature` to pick up feature bands automatically. Bump it deliberately — via Renovate or Dependabot, as a reviewed PR — because a compiler upgrade is a behavioural change (Concept 54).

**2. Centralize the build contract.** A `Directory.Build.props` at the root applies to every project beneath it:

```xml
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <!-- correctness diagnostics fail the build everywhere -->
    <WarningsAsErrors>$(WarningsAsErrors);nullable;CS8509;CS8524</WarningsAsErrors>
    <!-- no LangVersion: the target framework chooses it (Concept 4) -->
  </PropertyGroup>
  <PropertyGroup Condition="'$(ContinuousIntegrationBuild)' == 'true'">
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

Pair it with `Directory.Packages.props` (Central Package Management) so analyzer packages are declared once — `GlobalPackageReference` applies them to every project — and with a root `.editorconfig` that records the style decisions: file-scoped namespaces, `var` policy, and explicit severities for the suggestions you don't want (setting IDE0290, "use primary constructor," to `none` in domain projects is a common and defensible choice after Concept 13).

**3. Decide the upgrade cadence — and notice what changed.** A new C# ships every November with a new .NET. The traditional policy was "LTS to LTS," because STS releases were supported for less time. Since the support policy change, STS releases get **24 months** — so .NET 11 (STS, November 2026) and .NET 10 (LTS, November 2025) both reach end of support in **November 2028**. Adopting .NET 11 no longer shortens your support window; the real costs of tracking every release are upgrade labour and ecosystem lag (third-party libraries, container base images, and hosting platforms such as Azure Functions adding support). The two defensible policies are:

- **Rolling currency for actively developed services**: upgrade each service within a few months of every release. Each upgrade is small, the fleet never drifts far, and C# 15's closed sets become available as soon as the domain needs them.
- **LTS-to-LTS for low-change systems**: fewer, larger upgrades, accepted deliberately, with the next LTS (.NET 12, expected November 2027) as the planned target.

Either is fine; *having no policy* is the anti-pattern, because it produces a fleet spread across five runtimes, three of them out of support.

**4. Separate application policy from library policy.** Applications target one current TFM and get the default language version. Libraries shared across teams multi-target (`netstandard2.0;net8.0;net10.0` or similar), and therefore constrain themselves to features that lower on their lowest target, with polyfills (PolySharp) for the attribute-only features and `#if` for the rest (Concept 3). Public library surfaces deserve `Microsoft.CodeAnalysis.PublicApiAnalyzers`, which tracks every public signature in a checked-in file so API changes are explicit in review — and closed sets in public APIs are versioned as breaking changes (Concept 40).

**5. Make the upgrade itself a process.** For each annual upgrade: read the compiler breaking-changes document and the .NET breaking-changes pages for the release; bump `global.json` and the TFM on a branch; build with warnings as errors and triage every new diagnostic; run the full test suite *including* integration tests, because binding changes surface there; deploy to a canary; and only then roll across the fleet. Preview language features — `<LangVersion>preview</LangVersion>`, or the C# 15 memory-safety rules behind their feature flag — are for spikes and experiments, never for production code.

**6. Write it down and measure it.** The policy is an ADR (Module 31): SDK and TFM targets, cadence, the analyzer baseline, the warnings-as-errors set, and who owns the next upgrade. Then trend the handful of numbers that show whether it's working: nullable warnings, `!` count, `#pragma warning disable` and `[SuppressMessage]` counts, and the spread of TFMs across the fleet. A falling suppression count is the evidence that a language policy is being adopted rather than tolerated.

This is the answer to the architect-flavoured version of every question in this module. "Would you use primary constructors?" becomes "here's the rule we'd set, where it applies, and how the build enforces it."

---
# Putting it together

Six worked examples, each shaped like a question you might actually get: a modeling question, a code review, an API-edge mapping, an upgrade diagnosis, an integration exercise, and a design-round question about contracts between services.

## Worked example 1 — "Model an order's lifecycle: draft, placed, paid, shipped, cancelled."

**The shape most codebases start with:**

```csharp
public enum OrderStatus { Draft, Placed, Paid, Shipped, Cancelled }

public class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    public DateTimeOffset? PlacedAt { get; set; }
    public string? PaymentReference { get; set; }
    public DateTimeOffset? PaidAt { get; set; }
    public string? TrackingNumber { get; set; }
    public string? CancellationReason { get; set; }
    public List<OrderLine> Lines { get; set; } = new();
}
```

Say the problem as arithmetic, because it lands: five statuses times five independently nullable fields is 5 × 2⁵ = **160 representable states**, of which about five are legal. Every consumer re-derives the invariants — "if `Status == Shipped` then `TrackingNumber` isn't null" — usually with a `!`, and every public setter is a way to reach one of the 155 illegal states. The type documents nothing; the invariants live in people's heads (Concept 32).

**The modern shape:** the order is an *entity* — it keeps its identity while its data changes — so it stays a class. Its *state* is a value, and each state carries exactly the data that exists in that state:

```csharp
public readonly record struct PaymentReference(string Value);
public readonly record struct TrackingNumber(string Value);

public closed record class OrderState;
public sealed record class Draft : OrderState;
public sealed record class Placed(DateTimeOffset PlacedAt) : OrderState;
public sealed record class Paid(DateTimeOffset PlacedAt, PaymentReference Payment, DateTimeOffset PaidAt) : OrderState;
public sealed record class Shipped(PaymentReference Payment, TrackingNumber Tracking, DateTimeOffset ShippedAt) : OrderState;
public sealed record class Cancelled(DateTimeOffset CancelledAt, string Reason, PaymentReference? RefundDue) : OrderState;

public sealed class Order                                  // entity: identity, not value (Concept 15)
{
    private readonly List<OrderLine> _lines = [];

    public Order(OrderId id) => Id = id;

    public OrderId Id { get; }
    public OrderState State { get; private set; } = new Draft();
    public IReadOnlyList<OrderLine> Lines => _lines;

    public void AddLine(OrderLine line)
    {
        if (State is not Draft) throw InvalidTransition(nameof(AddLine));
        _lines.Add(line);
    }

    // Transition: a discard that throws is fail-safe — a new state can't be placed until someone decides it can.
    public void Place(TimeProvider clock) => State = State switch
    {
        Draft when _lines.Count > 0 => new Placed(clock.GetUtcNow()),
        _                           => throw InvalidTransition(nameof(Place)),
    };

    public void RecordPayment(PaymentReference payment, TimeProvider clock) => State = State switch
    {
        Placed p => new Paid(p.PlacedAt, payment, clock.GetUtcNow()),
        _        => throw InvalidTransition(nameof(RecordPayment)),
    };

    public void Ship(TrackingNumber tracking, TimeProvider clock) => State = State switch
    {
        Paid p => new Shipped(p.Payment, tracking, clock.GetUtcNow()),
        _      => throw InvalidTransition(nameof(Ship)),
    };

    // Transition deliberately exhaustive: refund semantics differ per state, so a new state must get a decision.
    public void Cancel(string reason, TimeProvider clock) => State = State switch
    {
        Draft or Placed      => new Cancelled(clock.GetUtcNow(), reason, RefundDue: null),
        Paid p               => new Cancelled(clock.GetUtcNow(), reason, RefundDue: p.Payment),
        Shipped or Cancelled => throw InvalidTransition(nameof(Cancel)),
    };

    private InvalidOperationException InvalidTransition(string action) =>
        new($"Cannot {action} an order in state {State.GetType().Name}.");
}

// Projection: exhaustive, no discard — adding a state breaks the build here, which is the point.
public static string StatusLabel(OrderState state) => state switch
{
    Draft       => "Draft",
    Placed      => "Awaiting payment",
    Paid        => "Preparing shipment",
    Shipped s   => $"Shipped ({s.Tracking.Value})",
    Cancelled c => c.RefundDue is null ? "Cancelled" : "Cancelled — refund pending",
};
```

**How to narrate it.** Six points, in roughly this order:

1. **The representable-state count dropped from 160 to 5.** A `Shipped` value cannot exist without a tracking number; no consumer needs `!`; illegal combinations are compile errors rather than validation code.
2. **Entity versus value is explicit.** `Order` is a class with an identity (`OrderId`); its state is a closed record hierarchy of values. Records are right for the state (two `Placed` states with the same timestamp *are* the same state) and wrong for the order (Concept 21).
3. **The discard policy is a design decision, stated per function.** *Projections* (`StatusLabel`, DTO mapping, UI) are exhaustive with no discard, so a new state produces a to-do list at compile time. *Transitions* can use a discard that throws — fail-safe, because a new state can't be placed or shipped until someone adds an arm — or be exhaustive, like `Cancel`, when every state genuinely needs its own decision. Saying this out loud is a strong senior signal (Concept 29).
4. **Guards don't count toward exhaustiveness.** `Draft when _lines.Count > 0` doesn't cover `Draft`, because the compiler can't reason about the guard — so `Place` needs its fallback arm regardless.
5. **Invalid transitions throw; expected rejections would be results.** Calling `Ship` on a `Draft` is a programming error — the UI or the application layer should have checked — so an exception is right. A business rejection that a user can trigger ("payment declined") belongs in a result type at the application layer (worked example 3).
6. **The platform cost is stated, not hidden.** `closed` is C# 15, which means `net11.0`. On .NET 10, the same model works with an `abstract record` base and a `private protected` constructor (so only this assembly can derive from it) — but without compiler-proven exhaustiveness, so every switch needs a discard arm and you've lost point 3. And persistence is a separate mapping decision: EF Core's inheritance support is designed for entity types, so the pragmatic options are a JSON column or a flat persistence shape (a status column plus nullable columns) mapped to the closed hierarchy in the repository. The flat shape is fine *in the database* — it's the domain that shouldn't have it (Module 19).

---

## Worked example 2 — "Review this code."

```csharp
public record Customer(Guid Id, string Name, Tier Tier, List<string> Tags)
{
    public string Slug => field ??= Name.ToLowerInvariant().Replace(' ', '-');
}

public sealed class CustomerService(ICustomerRepository repo, ILogger<CustomerService> logger)
{
    private readonly ICustomerRepository _repo = repo;

    public async Task<string> DescribeAsync(Guid id)
    {
        var customer = (await repo.FindAsync(id))!;
        logger.LogInformation($"Describing customer {customer.Name}");

        return customer.Tier switch
        {
            Tier.Gold   => "gold",
            Tier.Silver => "silver",
            _           => "standard",
        };
    }

    public Customer Rename(Customer customer, string name) => customer with { Name = name };

    public Task SaveAsync(Customer customer) => _repo.SaveAsync(customer);
}

public static class CustomerExtensions
{
    extension(Customer customer)
    {
        public bool HasOrders => OrderDatabase.Instance.Orders.Any(o => o.CustomerId == customer.Id);
    }
}
```

**A strong review, ordered by severity** (and saying that you're ordering it):

1. **`Rename` produces a customer whose `Slug` is wrong.** `with` clones every field, including the hidden backing field behind `field ??=`. If `Slug` was read before the rename, the clone carries the old slug forever (Concept 11). The same cache also breaks equality: two customers with identical data compare unequal if only one has had `Slug` read. Fix: compute `Slug` on demand, or derive it in a method — never cache inside a record.
2. **Equality is also broken by `Tags`.** `List<string>` compares by reference, so value equality is fiction, and `with` shares the same list between the original and the clone — mutate one, both change (Concept 19). Fix: an immutable collection *and* a custom equality member, or take `Tags` out of the record.
3. **Is `Customer` an entity?** It has an `Id` and it's being saved, so it almost certainly is — in which case it shouldn't be a record at all. Value equality on an entity breaks change tracking and identity-based collections; two different customers with the same data would be "equal" (Concept 21). Ask the question before recommending a fix, because the answer changes the fix.
4. **`!` on `FindAsync` converts "not found" into a `NullReferenceException` one line later.** "Not found" is an expected outcome; it belongs in the return type (a nullable or a result union) and should become a 404 at the edge (Concepts 39, 48).
5. **Two copies of the repository.** `_repo` is initialized from `repo`, and `DescribeAsync` also uses `repo` directly — so the compiler captures the parameter into a hidden field *as well*. CS9124 warns about exactly this. Pick one (Concept 13).
6. **The extension property does database I/O through a static singleton.** Property syntax promises cheap, pure, and repeatable; this is a synchronous query hidden behind `customer.HasOrders`, with a dependency no one can see or replace in a test. It should be an async method on a service that takes its dependency through the constructor (Concept 45; Module 15 for the sync-over-I/O cost).
7. **The log line.** An interpolated string loses the structured `Name` property, formats even if `Information` is disabled, and — worth raising — writes a customer's name, which is personal data, into logs. Use a message template or `[LoggerMessage]`, and log the ID rather than the name (Concept 58; Module 28).
8. **The discard maps any unknown tier to "standard".** Enums aren't closed, so some fallback is required (Concept 28) — but mapping unknown values to a business answer means a newly added `Tier.Platinum` is silently described as standard. Prefer explicit arms for every named value and a fallback that throws, so the new value fails loudly in tests.
9. **No `CancellationToken`** on the async methods (Module 15, Concept 19).
10. **`Rename` validates nothing.** `with` never runs a constructor, so there is no point at which an empty name is rejected (Concept 18).

The closing line that signals architect thinking: *"Items 5 and 7 shouldn't need a human — CS9124 as an error and CA2254 for the log template would catch them in CI, and once the methods take tokens, CA2016 enforces that they're forwarded. I'd raise that as a separate change to the analyzer baseline."*

---
## Worked example 3 — "Your service returns several business outcomes. Show me the endpoint."

This is Concept 39's stance — domain outcomes in the type, infrastructure failures as exceptions, translation at the edge — built end to end.

**The domain side** declares the outcomes as a union. The service signature now documents every way the operation can end:

```csharp
public sealed record Placed(OrderId Id, decimal Total);
public sealed record OutOfStock(string Sku);
public sealed record PaymentDeclined(string Reason);
public sealed record ValidationErrors(IReadOnlyDictionary<string, string[]> Errors);

public union PlaceOrderResult(Placed, OutOfStock, PaymentDeclined, ValidationErrors);

public interface IOrderService
{
    Task<PlaceOrderResult> PlaceAsync(PlaceOrderRequest request, CancellationToken ct);
}
```

**The edge** maps each outcome to HTTP with one exhaustive switch. Minimal APIs' `Results<…>` is itself union-shaped — a return type listing the possible results — so the mapping is a switch from one closed set to another:

```csharp
app.MapPost("/orders", async Task<Results<Created<OrderResponse>, ValidationProblem, Conflict<ProblemDetails>, ProblemHttpResult>>
    (PlaceOrderRequest request, IOrderService orders, CancellationToken ct) =>
{
    PlaceOrderResult result = await orders.PlaceAsync(request, ct);

    return result switch
    {
        Placed p           => TypedResults.Created($"/orders/{p.Id.Value}", new OrderResponse(p.Id, p.Total)),
        ValidationErrors v => TypedResults.ValidationProblem(v.Errors.ToDictionary()),
        OutOfStock o       => TypedResults.Conflict(new ProblemDetails
                              {
                                  Title = "Out of stock",
                                  Detail = $"SKU {o.Sku} is not available.",
                              }),
        PaymentDeclined d  => TypedResults.Problem(title: "Payment declined", detail: d.Reason,
                                                   statusCode: StatusCodes.Status402PaymentRequired),
        null               => throw new UnreachableException("PlaceAsync returned an empty result."),
    };
});
```

**What to point out while you write it:**

- **Two closed sets, one mapping.** The switch is target-typed to `Results<…>`; each arm converts implicitly into it. Because `Results<…>` also supplies endpoint metadata, the OpenAPI document describes 201, 400, 409, and the problem response without a single `[ProducesResponseType]`.
- **The `null` arm handles `default(PlaceOrderResult)`**, the empty union (Concept 35). It isn't a discard: a `null` pattern on a union tests the *contents*, so the four type arms still have to be present and the switch still breaks when a case is added. Whether the compiler insists on this arm depends on its null-state tracking of the value; keeping it explicit documents the assumption either way.
- **Adding `AlreadyExists` to the union breaks exactly one line of this endpoint** (CS8509), and the decision it forces — which status code? — is a genuine API design decision that should be made by a person, not defaulted.
- **Infrastructure failures aren't in the union.** A database timeout inside `PlaceAsync` throws, propagates past this endpoint, and becomes a 500 problem response via `AddProblemDetails()` and the exception-handler middleware. That's correct: the endpoint can't do anything useful about it, and the stack trace is exactly what the on-call engineer needs.
- **The union doesn't cross the wire.** Minimal APIs *can* return a union directly — .NET 11's STJ serializes the active case — but that produces a 200 with a different body shape per outcome, which is right for a value-shaped contract like `IntOrString` and wrong for outcomes that have different HTTP semantics. Here, the wire contract is status codes plus RFC 9457 problem details; the union is an in-process type.
- **On .NET 10**, the same design works with a closed-by-convention `abstract record` base plus a discard arm, or a library union type such as OneOf. The endpoint shape is identical; only the compile-time guarantee is weaker.

---

## Worked example 4 — "We moved a service from .NET 8 to .NET 10. Three things changed. Explain them."

Moving from `net8.0` to `net10.0` silently moves the language from C# 12 to C# 14 (Concept 4) — two language versions, plus two years of BCL and analyzer changes. The three symptoms and their causes:

**Symptom 1 — a new warning in a legacy class, and then a wrong value in production.**

```csharp
public class LegacySetting
{
    private string field = "";                       // an old naming convention
    public string Value
    {
        get => field;                                 // C# 12: this.field.  C# 14: the synthesized backing field
        set => field = value.Trim();
    }
    public override string ToString() => field;      // still this.field — outside the accessor
}
```

In C# 14, `field` inside an accessor is the contextual keyword (Concept 11). The property now reads and writes a *new* compiler-generated backing field, while `ToString` and every other member still use the declared one — so the two drift apart. The compiler warns at each rebinding, but under a policy that lets warnings through, it ships. Fix: `this.field`, `@field`, or a rename. Lesson: **language-version warnings on an upgrade PR are never noise.**

**Symptom 2 — an in-house expression-tree visitor started throwing.**

```csharp
Expression<Func<Product, bool>> filter = p => allowedIds.Contains(p.Id);   // allowedIds is a Guid[]
```

With first-class spans (Concept 54), `allowedIds.Contains(…)` now binds to `MemoryExtensions.Contains` over a `ReadOnlySpan<Guid>` instead of `Enumerable.Contains`. EF Core recognizes and translates the new shape; a hand-written visitor that pattern-matched on `Enumerable.Contains` — for a search-index query builder, say — no longer finds it, and interpreting the tree fails because it involves a `ref struct`. Fix: teach the visitor the new method (or cast the receiver to `IEnumerable<Guid>` at the call site as a stopgap). Lesson: **overload resolution is semantics, and the compiler owns it.**

**Symptom 3 — the build broke on warnings no one had seen before.**

With `<AnalysisLevel>latest-recommended</AnalysisLevel>` and warnings-as-errors in CI, the SDK upgrade brought new analyzer rules and new default severities. Nothing in the code changed; the rule set did. This one isn't a bug — it's policy working — but it has to be *expected*: either pin the analysis level (`10.0-recommended`) and move it deliberately, or float it and accept that the upgrade PR is where new rules land and get triaged.

**And one pleasant surprise worth mentioning**: allocation profiles improved at many call sites with no code change, because `string.Join`, `Path.Combine`, `Task.WhenAll`, and others gained `params ReadOnlySpan<T>` overloads that recompiled code now binds to (Concept 55).

**The process answer** — which is what the question is really asking for: pin the SDK in `global.json`; upgrade on a dedicated branch; read the C# compiler breaking-changes document and the .NET breaking-changes pages for every version you're crossing (here: 9 *and* 10); build with warnings as errors and triage every new diagnostic; run the full test suite including integration tests and any expression-tree or serialization tests; canary; then roll out (Concept 70).

---
## Worked example 5 — "Introduce strongly-typed IDs. What does it take, end to end?"

Concept 22 made the case in one line — `readonly record struct OrderId(Guid Value)` turns an ID mix-up into a compile error — and named the integration cost. This example pays it: serialization, persistence, HTTP binding, and the `default` problem.

**1. The type.** Not positional this time, deliberately: a hand-written constructor validates, and a get-only property means `with { Value = … }` doesn't compile, so there's no bypass (Concept 18).

```csharp
[JsonConverter(typeof(OrderIdJsonConverter))]
public readonly record struct OrderId : IParsable<OrderId>
{
    public Guid Value { get; }

    public OrderId(Guid value)
    {
        if (value == Guid.Empty) throw new ArgumentException("An OrderId cannot be empty.", nameof(value));
        Value = value;
    }

    public static OrderId New() => new(Guid.CreateVersion7());          // time-ordered: friendlier to B-tree indexes

    public static OrderId Parse(string s, IFormatProvider? provider) =>
        TryParse(s, provider, out var id) ? id : throw new FormatException($"'{s}' is not a valid OrderId.");

    public static bool TryParse([NotNullWhen(true)] string? s, IFormatProvider? provider, out OrderId result)
    {
        if (Guid.TryParse(s, provider, out var guid) && guid != Guid.Empty)
        {
            result = new OrderId(guid);
            return true;
        }
        result = default;
        return false;
    }

    public override string ToString() => Value.ToString();             // logs show the GUID, not "OrderId { Value = … }"
}
```

**2. JSON.** Without a converter, STJ writes `{"value":"…"}`. The converter makes the wire format a plain string and handles dictionary keys:

```csharp
public sealed class OrderIdJsonConverter : JsonConverter<OrderId>
{
    public override OrderId Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options) =>
        reader.TokenType == JsonTokenType.String && reader.TryGetGuid(out var guid) && guid != Guid.Empty
            ? new OrderId(guid)
            : throw new JsonException("Expected a non-empty GUID string for OrderId.");

    public override void Write(Utf8JsonWriter writer, OrderId value, JsonSerializerOptions options) =>
        writer.WriteStringValue(value.Value);

    // Dictionary<OrderId, T> serializes as a JSON object keyed by the ID
    public override OrderId ReadAsPropertyName(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options) =>
        OrderId.TryParse(reader.GetString(), null, out var id) ? id : throw new JsonException("Invalid OrderId key.");

    public override void WriteAsPropertyName(Utf8JsonWriter writer, OrderId value, JsonSerializerOptions options) =>
        writer.WritePropertyName(value.Value.ToString());
}
```

Throw `JsonException` rather than letting the constructor's `ArgumentException` escape, so malformed input becomes a 400 at the API edge rather than a 500. A converter referenced by `[JsonConverter]` also works with STJ's source-generated contexts, which keeps the type AOT-friendly (Concept 62).

**3. EF Core.** One converter, applied by convention to every `OrderId` property in the model:

```csharp
public sealed class OrderIdConverter()
    : ValueConverter<OrderId, Guid>(id => id.Value, value => new OrderId(value));

protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Properties<OrderId>().HaveConversion<OrderIdConverter>();
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>().Property(o => o.Id).ValueGeneratedNever();   // the domain assigns OrderId.New()
}
```

Two traps to name: **compare IDs, don't reach into `.Value` in queries** — `Where(o => o.Id == id)` translates cleanly because the converter applies to the parameter, while `o.Id.Value` targets a member EF can't map to a column; and **key generation** — EF can't invent an `OrderId`, so either the domain assigns one (as here, via the constructor) or you register a value generator.

**4. HTTP binding.** Minimal APIs bind route and query values through a static `TryParse`, which `IParsable<T>` provides — so the ID is a first-class parameter, and a malformed ID is rejected with a 400 before your handler runs:

```csharp
app.MapGet("/orders/{id}", async Task<Results<Ok<OrderResponse>, NotFound>>
    (OrderId id, IOrderReader reader, CancellationToken ct) =>
        await reader.FindAsync(id, ct) is { } order ? TypedResults.Ok(order) : TypedResults.NotFound());
```

Check the OpenAPI output, though: a custom JSON converter isn't visible to schema generation, so you may need a schema transformer to describe `OrderId` as a string with `format: uuid` rather than an object.

**5. The `default` problem.** `default(OrderId)` — an uninitialized field, an array element, `new OrderId()` — never runs the constructor and holds `Guid.Empty`. A struct cannot prevent it. The mitigations are the ones you already have: validate at the boundaries (the converter and `TryParse` both reject empty), assign IDs in entity constructors, and — if you use a generator — let its analyzer ban `default` construction at compile time.

**6. The tests that prove it:** a JSON round trip, including as a dictionary key and with an empty GUID rejected; an EF `ToQueryString()` assertion that the filter translates to a parameterized `WHERE`; a route test showing `/orders/not-a-guid` returns 400; and a dictionary-key test proving equality and hashing work.

**7. Or generate it.** At three ID types, the hand-written version is fine and teaches the team what's happening. At thirty, it's boilerplate with bugs in it, and a generator earns its place (Concept 59). With Vogen, the whole of steps 1–3 collapses to roughly this (check the current option names in its documentation):

```csharp
[ValueObject<Guid>(conversions: Conversions.SystemTextJson | Conversions.EfCoreValueConverter)]
public readonly partial struct OrderId
{
    private static Validation Validate(Guid input) =>
        input == Guid.Empty ? Validation.Invalid("An OrderId cannot be empty.") : Validation.Ok;
}
```

Vogen also ships analyzers that flag `default(OrderId)` and parameterless construction — closing the gap in step 5 that no hand-written struct can close. StronglyTypedId (Andrew Lock) is the lighter-weight alternative.

**The judgement to finish on:** strongly-typed IDs are worth it for aggregate and entity identifiers, where mixing two up is a plausible, expensive bug. They're not worth it for every integer in the codebase — and saying where you'd stop is part of the answer.

---

## Worked example 6 — "Should our services' event contracts use C# 15 unions?"

This is a design-round question disguised as a language question, and the answer is Concept 40's rule — **closed inside, open at the edges** — made concrete.

**Start with what the question is really about: independent deployment.** The payments service will add `PaymentRefunded` before the fulfilment service is redeployed; that's what independent deployability means (Module 11). A consumer whose contract is a closed set, compiled before the new case existed, has no arm for it. If the deserializer can even produce the value, the consumer's switch throws `SwitchExpressionException`; more likely, deserialization itself fails on an unknown shape, and the message goes to the dead-letter queue — along with every subsequent one of that type until someone deploys.

**Then separate the two type systems.** The wire contract is *open by construction*; the consumer's domain model is *closed by construction*; an anti-corruption layer translates between them:

```csharp
// Wire model — open: any type string, any version, the payload kept raw
public sealed record EventEnvelope(string Type, int Version, DateTimeOffset OccurredAt, JsonElement Payload);

// Consumer's domain model — closed, including an explicit case for "something we don't know yet"
public closed record class PaymentEvent;
public sealed record class PaymentAuthorized(string PaymentId, decimal Amount) : PaymentEvent;
public sealed record class PaymentFailed(string PaymentId, string Reason) : PaymentEvent;
public sealed record class UnrecognizedPaymentEvent(string Type, int Version, JsonElement Raw) : PaymentEvent;

// Anti-corruption layer — the one place a discard is correct, because the wire really is open
static PaymentEvent Translate(EventEnvelope e) => (e.Type, e.Version) switch
{
    ("payment.authorized", 1) => Read<PaymentAuthorizedV1>(e).ToDomain(),
    ("payment.failed", 1)     => Read<PaymentFailedV1>(e).ToDomain(),
    _                         => new UnrecognizedPaymentEvent(e.Type, e.Version, e.Payload),
};

static T Read<T>(EventEnvelope e) =>
    e.Payload.Deserialize<T>(JsonSerializerOptions.Web)
    ?? throw new InvalidMessageException($"Empty payload for {e.Type} v{e.Version}.");

// Handler — exhaustive, no discard: the unknown case is handled deliberately, not silently
Task HandleAsync(PaymentEvent evt, CancellationToken ct) => evt switch
{
    PaymentAuthorized a        => ReserveStockAsync(a, ct),
    PaymentFailed f            => ReleaseReservationAsync(f, ct),
    UnrecognizedPaymentEvent u => ParkForReviewAsync(u, ct),      // log, metric, park — never crash
};
```

**Now answer the actual question — "unions or not?" — in three parts:**

1. **Inside the consumer: yes to a closed set** (a closed hierarchy here, because the cases are related and share a base), with an exhaustive handler. Adding `PaymentRefunded` to the consumer's model is a compile-time to-do list, exactly as in worked example 1.
2. **On the wire: no closed set, and a discriminator you own.** The `Type` and `Version` fields are the contract, documented and versioned (Module 11's schema evolution rules). Note what .NET 11's STJ does with a union — it writes the active case with *no* discriminator (Concept 35), which is right for describing an existing discriminator-free format and wrong for a new event contract you control. If the producer wants to serialize its own closed hierarchy directly, `[JsonPolymorphic(InferClosedTypePolymorphism = true)]` gives it discriminators without registering every type — but the consumer should still read through an envelope, because a closed base is implicitly abstract, so there's no base type for an unknown discriminator to fall back to.
3. **Between them: the tolerant reader.** Unknown types become an explicit case, not an exception. That case is observable (a metric and a log line), recoverable (parked, replayable once the consumer is updated), and never blocks the partition or queue behind it.

**The one-sentence summary for the interviewer:** *"Unions and closed hierarchies are the best tool we've had for making a consumer handle every case it knows about — and the wrong tool for describing a contract that another team will extend, so I'd use them on the inside of the anti-corruption layer and keep the wire open with an explicit unknown case."*

---
