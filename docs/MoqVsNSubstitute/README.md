# Moq vs NSubstitute — feature matrix, pros/cons, and code snippets


**Table of Contents**

1. [Version note](#1-version-note)
2. [Summary](#2-summary)
3. [Feature matrix (high-level)](#3-feature-matrix-high-level)
4. [Pro / Con summary (short)](#4-pro--con-summary-short)
5. [Recommendation guidance (how to choose)](#5-recommendation-guidance-how-to-choose)
6. [Migration considerations](#6-migration-considerations)
7. [Large list of example snippets](#7-large-list-of-example-snippets)
    <br/>7.1. [Moq-only (or Moq-first) features](#71-moq-only-or-moq-first-features--examples-that-moq-offers-as-dedicated-apis-no-direct-nsubstitute-counterpart)
    <br/>7.2. [NSubstitute-first features / conveniences](#72-nsubstitute-first-features--conveniences--examples-where-nsubstitute-tends-to-be-terser-or-more-ergonomic)
    <br/>7.3. [More practical small examples](#73-more-practical-small-examples-common-unit-test-patterns)
8. [Practical checklist](#8-practical-checklist--things-to-look-for-in-your-codebase-before-switching)
9. [Closing notes](#9-closing-notes)
10. [References](#10-references)



## 1. Version note

This comparison is based on the public APIs and behaviors of Moq and NSubstitute as they existed around mid‑2024 (representative of Moq 4.x series and NSubstitute 4.x series). Libraries evolve frequently — verify the current upstream documentation/releases for any feature that is critical to your decision.

## 2. Summary
- This document compares Moq and NSubstitute: what each exposes as first‑class APIs, notable differences in ergonomics and capabilities, migration considerations, a short pro/con summary, and a large set of example snippets showing idioms from both libraries.
- Important caveat: both libraries evolve. This comparison captures the commonly used differences and conveniences as of recent versions, but it is not mathematically exhaustive. If a precise feature is critical, verify against the current upstream docs.

## 3. Feature matrix (high-level)
- Table rows highlight notable features where Moq and NSubstitute differ (either capability or ergonomics). The “Direct support” column describes whether the capability exists as a dedicated, built-in API in the library (Yes = first-class API; Partial = doable but with different API/ergonomics; No = no built-in equivalent).

| Feature / area | Moq direct support? | NSubstitute direct support? | Notes |
|---|---:|---:|---|
| Create instance from expression (LINQ-to-mocks: Mock.Of<T>(expr)) | Yes | No | Moq provides concise expression→mock helper. NSubstitute requires explicit Substitute.For + Returns setups. |
| Protected member Setup/Verify (.Protected()) | Yes | No / Partial | Moq has a Protected() DSL. NSubstitute needs reflection or ForPartsOf hacks. |
| SetupAllProperties() | Yes | Partial | Moq provides a single-call convenience. NSubstitute supports property assignment/behavior but not the same one‑call initialization. |
| MockBehavior.Strict (global strict mode) | Yes | No | Moq can make mocks strict; NSubstitute prefers interaction assertions (Received) and explicit When(...) setups. |
| DefaultValue.Mock (recursive automatic mocks) | Yes | No | Moq can auto-create nested mocks; NSubstitute requires explicit substitutes for nested members. |
| Mock.Reset / ResetCalls | Yes | No | Moq has Reset helpers; with NSubstitute you normally create a fresh substitute. |
| Protected event raise helpers | Yes (via Protected API) | Partial | NSubstitute raises public events easily; protected event raising requires workarounds. |
| SetupSequence (fluent helper) | Yes | Partial | Both support sequential returns; Moq has a dedicated SetupSequence fluent API; NSubstitute uses Returns(...) overloads. |
| As<TInterface>() on existing mock (add interface after creation) | Yes | Partial | Moq supports .As<T>(). NSubstitute can create multi-interface substitute at creation time but not add interfaces later via .As. |
| Expression-rich verification helpers | Yes | Partial | Moq offers many expression-based helpers. NSubstitute uses Received/DidNotReceive and Arg matchers; verification style differs. |
| Substitute.For<T>() returns real instance (no mock.Object) | No (Moq uses Mock<T>.Object) | Yes | NSubstitute returns the substitute instance directly (no .Object). Ergonomic difference. |
| Multi-interface creation at creation time | Partial (Mock.As exists) | Yes | More concise in NSubstitute to create substitute implementing multiple interfaces in one call. |
| When(...).Do(...) concise side-effects for voids | Partial | Yes | Both can do callbacks/side-effects; NSubstitute’s When(...).Do(...) often reads more directly for voids. |
| CallInfo in Returns for dynamic access to args | Partial | Yes (CallInfo) | NSubstitute’s CallInfo-based Returns is ergonomic for dynamic argument access. Moq uses typed Returns delegates. |
| Received.InOrder inline verification | Partial (MockSequence) | Yes | NSubstitute Received.InOrder is concise and inline; Moq uses MockSequence which is more verbose. |
| Compact Arg.Do for inline argument capture | No | Yes | NSubstitute provides Arg.Do for capturing while matching. Moq uses Callback which is more separate. |
| Delegate mocking ergonomics | Partial | Yes | NSubstitute handles delegate substitutes concisely. Moq can mock delegates but with more ceremony. |

## 4. Pro / Con summary (short)
- Moq (Pros)
  - Rich expression-based Setup/Verify and many convenience helpers (Protected(), SetupAllProperties, DefaultValue.Mock, SetupSequence, MockBehavior.Strict, Reset).
  - Wide usage, many examples and community recipes that assume Mock<T>.
  - Good when you need advanced control over non-public members and recursive mock generation.
- Moq (Cons)
  - More ceremony: Mock<T>.Object indirection, sometimes verbose APIs.
  - Some patterns require more boilerplate (in-order checks, delegate mocking).
- NSubstitute (Pros)
  - Concise interaction-oriented API: Substitute.For returns the instance, When(...).Do(...), Received/DidNotReceive, Received.InOrder, CallInfo-based Returns, Arg.Do.
  - Readability: tests often read closer to natural language (arrange/substitute/act/assert).
  - Easier to create multi-interface substitutes and to capture args inline.
- NSubstitute (Cons)
  - Lacks convenient first-class helpers for protected/non-public members and recursive default mocks.
  - Migration from Moq features like SetupAllProperties, Protected(), DefaultValue.Mock, MockBehavior.Strict may require refactors or different test patterns.

## 5. Recommendation guidance (how to choose)
- Keep Moq if:
  - Your test suite relies heavily on Protected(), SetupAllProperties, recursive default mocks, strict mocks, or Reset semantics, and migration cost is high.
  - Tooling/recipes used across the org assume Moq.Mock<T>.
- Consider NSubstitute if:
  - You prefer concise interaction-style tests (Received, When/Do), Arg.Do inline capturing, Received.InOrder, and you value terser syntax.
  - Your tests do not rely heavily on Moq-specific features listed above.
- Migration cost: if many tests use Moq-specific conveniences, estimate refactor time. Many behavioral tests map 1:1, but protected and recursive-mock usages need special treatment.

## 6. Migration considerations
- Features that require attention when migrating Moq -> NSubstitute:
  - Protected() usage: convert to partial substitutes + reflection or redesign to test via public behavior.
  - SetupAllProperties/default recursive mocks: refactor tests to explicitly create substitutes for nested collaborators or change the SUT to allow easier injection.
  - MockBehavior.Strict: replace by explicit When(...).Do(...) side-effect setups and assertions on Received calls to detect unexpected calls.
  - Reset calls: test setup should create fresh substitutes per test rather than resetting a shared mock between tests (preferred pattern anyway).
- Tests that primarily assert interactions (method calls, ordering, arguments) generally port cleanly and often get simpler with NSubstitute.

## 7. Large list of example snippets
- The snippets below demonstrate the idioms previously described. Each snippet is preceded by a comment describing what it demonstrates. The code is C# style pseudo-code (not necessarily fully compilable as-is) and intended to show the differences and patterns.

### 7.1 Moq-only (or Moq-first) features — examples that Moq offers as dedicated APIs (no direct NSubstitute counterpart)

```csharp
// Moq: LINQ-to-mocks (Mock.Of<T>(expr)) - create a mock using an expression shorthand
var moq = Mock.Of<IFoo>(f => f.Prop == 5 && f.Get("x") == "y");

// Equivalent with NSubstitute requires explicit setup (no single expression API)
var sub = Substitute.For<IFoo>();
sub.Prop.Returns(5);
sub.Get("x").Returns("y");
```

```csharp
// Moq: Protected() API for setting up a protected member
var mock = new Mock<MyClass>();
mock.Protected()
    .Setup<string>("ProtectedMethod", ItExpr.IsAny<int>())
    .Returns("ok");

// NSubstitute: no direct Protected() API; you'd need Substitute.ForPartsOf<T>() or reflection tricks
// Example (partial substitute, but not the same Protected().Setup florality):
var partial = Substitute.ForPartsOf<MyClass>();
// may require protected method being virtual and additional setup via reflection
```

```csharp
// Moq: SetupAllProperties() - initialize automatic backing for all (virtual) properties
var mock = new Mock<IFoo>();
mock.SetupAllProperties();
mock.Object.Prop = 123;
Assert.Equal(123, mock.Object.Prop);

// NSubstitute has property semantics but no SetupAllProperties convenience call
var sub = Substitute.For<IFoo>();
sub.Prop = 123; // works for concrete auto-properties but SetupAllProperties is a Moq convenience
```

```csharp
// Moq: Strict mock behaviour - unexpected calls throw
var strict = new Mock<IFoo>(MockBehavior.Strict);
strict.Setup(x => x.DoSomething()).Verifiable();
// Calling an unexpected method will throw in strict mode

// NSubstitute pattern: no MockBehavior.Strict — prefer explicit When(...).Do(...) or verify Received() to assert expected calls
```

```csharp
// Moq: DefaultValue.Mock for recursive automatic mocks
var m = new Mock<IFoo> { DefaultValue = DefaultValue.Mock };
var child = m.Object.Child; // child is automatically a mock that can be set up

// NSubstitute: no recursive automatic mock creation; create explicit substitutes for nested props
```

```csharp
// Moq: Reset helpers to clear setups / calls
var m = new Mock<IFoo>();
m.Setup(x => x.Get()).Returns(1);
Mock.Reset(m); // clears setups

// NSubstitute: no Reset API; create a fresh Substitute.For<IFoo>() instead in tests
```

```csharp
// Moq: Protected().Raise to raise protected events via the Protected DSL
var mock = new Mock<MyClass>();
mock.Protected().Raise("OnSomething", EventArgs.Empty);

// NSubstitute: raising protected events is not first-class; public events can be raised with Raise.Event
```

```csharp
// Moq: SetupSequence fluent API for sequential returns
var m = new Mock<IFoo>();
m.SetupSequence(x => x.Get())
 .Returns(1)
 .Returns(2)
 .Throws(new Exception("boom"));

// NSubstitute alternative:
var s = Substitute.For<IFoo>();
s.Get().Returns(1, 2, x => { throw new Exception("boom"); });
```

```csharp
// Moq: Mock.As<TInterface>() to add interface on existing mock
var m = new Mock<BaseClass>();
m.As<IAnother>().Setup(a => a.Some()).Returns(1);

// NSubstitute: create substitute for multiple interfaces at creation time is possible, but no As<T>() on existing substitute
var both = Substitute.For(new[] { typeof(BaseClass), typeof(IAnother) }, new object[] { /* ctor args */ });
```

```csharp
// Moq: expression-heavy verification helper example
mock.Verify(m => m.Method(It.Is<string>(s => s.StartsWith("x"))), Times.Once);

// NSubstitute: use Received with Arg.Is:
repo.Received(1).Method(Arg.Is<string>(s => s.StartsWith("x")));
```

```csharp
// Moq: Callback + Returns chained on Setup (typical style)
var m = new Mock<IFoo>();
int captured = 0;
m.Setup(x => x.Do(It.IsAny<int>()))
 .Callback<int>(arg => { captured = arg; })
 .Returns(true);

// NSubstitute: When(...).Do(...) for side-effects plus Returns for return-value
var s = Substitute.For<IFoo>();
int captured2 = 0;
s.When(x => x.Do(Arg.Any<int>()))
 .Do(ci => { captured2 = ci.Arg<int>(); });
s.Do(Arg.Any<int>()).Returns(true);
```

### 7.2 NSubstitute-first features / conveniences — examples where NSubstitute tends to be terser or more ergonomic

```csharp
// NSubstitute: Substitute.For<T>() returns the instance directly (no .Object)
var sub = Substitute.For<IFoo>(); // use sub directly in DI or SUTs

// Moq requires mock.Object:
var mock = new Mock<IFoo>();
var instance = mock.Object;
```

```csharp
// NSubstitute: create substitute implementing multiple interfaces in one call (concise)
var multi = Substitute.For(new[] { typeof(IFoo), typeof(IBar) }, null);

// Moq would require creation and then As<T>() or more ceremony.
```

```csharp
// NSubstitute: When(...).Do(...) concise side-effect setup for voids
var sub = Substitute.For<IFoo>();
bool called = false;
sub.When(x => x.VoidMethod(Arg.Any<int>()))
   .Do(ci => { called = true; });

// Moq equivalent:
var mock = new Mock<IFoo>();
mock.Setup(x => x.VoidMethod(It.IsAny<int>()))
    .Callback<int>(i => { called = true; });
```

```csharp
// NSubstitute: CallInfo usage inside Returns to access args dynamically
sub.SomeMethod(Arg.Any<int>()).Returns(ci => $"got {ci.Arg<int>()}");

// Moq uses typed Returns, e.g. .Returns<int>(i => $"got {i}"), which is fine but not CallInfo-indexed.
```

```csharp
// NSubstitute: Arg.Do inline argument capture while matching
int captured = 0;
sub.DoSomething(Arg.Do<int>(x => captured = x));

// Moq equivalent uses Callback:
mock.Setup(x => x.DoSomething(It.IsAny<int>()))
    .Callback<int>(x => captured = x);
```

```csharp
// NSubstitute: inline in-order verification is concise
Received.InOrder(() =>
{
    sub.First();
    sub.Second();
});

// Moq uses MockSequence (more verbose):
var seq = new MockSequence();
mock.InSequence(seq).Setup(m => m.First());
mock.InSequence(seq).Setup(m => m.Second());
```

```csharp
// NSubstitute: multiple sequential returns are concise
sub.Get().Returns(1, 2, 3); // successive calls return 1 then 2 then 3

// Moq equivalent: SetupSequence(...).Returns(1).Returns(2).Returns(3);
```

```csharp
// NSubstitute: concise delegate substitution
var del = Substitute.For<Func<int, string>>();
del(5).Returns("five");

// Moq can create delegate mocks but it's less ergonomic.
```

```csharp
// Callback differences side-by-side: Moq vs NSubstitute
// Moq: Setup...Callback...Returns (fluent)
mock.Setup(x => x.Process(It.IsAny<string>()))
    .Callback<string>(s => { Console.WriteLine("moq callback " + s); })
    .Returns(true);

// NSubstitute: When(...).Do(...) for side-effect, and Returns for return-value
sub.When(x => x.Process(Arg.Any<string>()))
   .Do(ci => { Console.WriteLine("nsub callback " + ci.Arg<string>()); });
sub.Process(Arg.Any<string>()).Returns(true);
```

### 7.3 More practical small examples (common unit test patterns)

```csharp
// NSubstitute: Return a Task for async method (Task<bool> CreateIndexOnStartup())
var repo = Substitute.For<IFooRepository<FooData>>();
repo.CreateIndexOnStartup().Returns(Task.FromResult(true));

// Verify it was called once
await repo.Received(1).CreateIndexOnStartup();

// Moq equivalent:
var moqRepo = new Mock<IFooRepository<FooData>>();
moqRepo.Setup(r => r.CreateIndexOnStartup()).ReturnsAsync(true);

// Verify it was called once
moqRepo.Verify(r => r.CreateIndexOnStartup(), Times.Once);
```

```csharp
// NSubstitute: capture argument when SetAsync is called
FooData? captured = null;
repo.When(r => r.SetAsync(Arg.Any<string>(), Arg.Any<FooData>(), Arg.Any<TimeSpan?>()))
    .Do(ci => captured = ci.Arg<FooData>(1));

await repo.SetAsync("foo:2", new FooData { Id = "2" }, null);
await repo.Received(1).SetAsync("foo:2", Arg.Any<FooData>(), Arg.Any<TimeSpan?>());
Assert.Same(captured!.Id, "2");

// Moq equivalent:
FooData? capturedMoq = null;
var moqRepo2 = new Mock<IFooRepository<FooData>>();
moqRepo2.Setup(r => r.SetAsync(It.IsAny<string>(), It.IsAny<FooData>(), It.IsAny<TimeSpan?>()))
  .Callback<string, FooData, TimeSpan?>((k, v, t) => capturedMoq = v)
  .Returns(Task.CompletedTask);

await moqRepo2.Object.SetAsync("foo:2", new FooData { Id = "2" }, null);
moqRepo2.Verify(r => r.SetAsync("foo:2", It.IsAny<FooData>(), It.IsAny<TimeSpan?>()), Times.Once);
Assert.Same(capturedMoq!.Id, "2");
```

```csharp
// NSubstitute: make RemoveAsync throw for a specific key
repo.RemoveAsync("bad-key").Returns<Task<bool>>(x => throw new InvalidOperationException("boom"));
await Assert.ThrowsAsync<InvalidOperationException>(async () => await repo.RemoveAsync("bad-key"));

// Moq equivalent:
var moqRepo3 = new Mock<IFooRepository<FooData>>();
moqRepo3.Setup(r => r.RemoveAsync("bad-key")).ThrowsAsync(new InvalidOperationException("boom"));
await Assert.ThrowsAsync<InvalidOperationException>(async () => await moqRepo3.Object.RemoveAsync("bad-key"));
```

## 8. Practical checklist — things to look for in your codebase before switching
- Count occurrences of Moq.Protected(), SetupAllProperties(), DefaultValue.Mock, MockBehavior.Strict, Mock.Reset — these require focused changes.
- Identify tests that rely on MockSequence or expression-based verification with heavy It.Is<> predicate usage — usually portable but check readability changes.
- If your codebase has internal helper code expecting Mock<T> type (libraries/tools), update those helpers or keep Moq.

## 9. Closing notes
- Both Moq and NSubstitute are mature libraries and most real-world tests can be implemented in either. The decision is largely about ergonomics and the particular advanced features you need.
- If your team values terse, interaction-based tests and less ceremony, NSubstitute is attractive. If you rely on Moq’s protected/non-public helpers, recursive mocks, strict behavior, or expression-heavy verification, staying on Moq or estimating migration effort is prudent.
- If you’d like, I can produce a migration checklist tailored to your repository (scan for Moq-specific APIs and generate a prioritized refactor plan) — point me to the code or give me grep output for Mock. or Protected() usage and I’ll draft a plan.

## 10. References
- Moq docs: https://github.com/moq/moq4
- NSubstitute docs: https://nsubstitute.github.io/
- When migrating, check current library changelogs for new features that may narrow gaps above.
