# xUnit (v2) vs. NUnit (v4) — Feature Comparison Table

| ID  | Category                | Feature/Behavior                           | NUnit                                                                                                      | xUnit                                                                                                  | Notes                                                                                                                                                                                   |
|-----|-------------------------|--------------------------------------------|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 01 | Attributes & Metadata   | [Category] attribute / [Trait]             | ✅ `[Category]` attribute                                                                                   | ⚠️ `[Trait]` attribute or method                                                                       | NUnit has first-class category support, xUnit uses trait key/value pairs.                                                                        |
| 02 | Attributes & Metadata   | [Explicit] Attribute                       | ✅ `[Explicit]` attribute available to mark tests                                                           | ❌ No `[Explicit]` (use `[Fact(Skip = "...")]` for static skip)                                        | NUnit lets you mark tests to run only when explicitly invoked.                                                                                   |
| 03 | Attributes & Metadata   | [TestCaseSource]/[Test] usage              | ✅ `[TestCaseSource]` does **not** require `[Test]` (using both is redundant but works).                    | ❌ `[InlineData]` and `[MemberData]` **must** be used with `[Theory]` (not `[Fact]`).                   | **NUnit is preferable:** you don't have to care if a test uses test data or not; just add `[TestCaseSource]`. In xUnit, you must switch between `[Fact]` and `[Theory]` depending on test data. |
| 04 | Attributes & Metadata   | [TestFixture] class attribute              | ✅ Optional since NUnit 2.5. Public classes with test methods are discovered as test fixtures even without it. | ✅ Not used or required. Any public class with `[Fact]` or `[Theory]` is a test class.                  | Both frameworks do not require a class attribute for test discovery, making setup easier.                                                        |
| 05 | Attributes & Metadata   | Test Timeout                               | ✅ `[Timeout(n)]` attribute supported on tests/fixtures                                                     | ❌ No built-in per-test timeout attribute                                                              | xUnit requires external means (e.g., cancellation tokens or test runner features).                                                               |
| 06 | Code & Test Structure   | Async void Test Support                    | ⚠️ Supported, but not recommended                                                                          | ❌ Not supported (must use `Task` return type for async tests)                                          | xUnit enforces best practice for async tests.                                                                                                    |
| 07 | Code & Test Structure   | Dispose vs OneTimeTearDown                 | ❗ Similar to `[OneTimeTearDown]`<br>🔹 `Dispose()` is called once after all tests in a fixture/class.<br>⚠️ Not per-test, use `[TearDown]` for that. | ✅ Per-test precision<br>🔹 `Dispose()` is called after **every single test**.                          | NUnit’s `Dispose()` is equivalent to `[OneTimeTearDown]` (class-level), while xUnit’s `Dispose()` is always per-test (instance-level).            |
| 08 | Code & Test Structure   | Fixture/Context Sharing                    | ✅ `[TestFixture]`, `[TestFixtureSource]`                                                                   | ✅ `[Collection]`, class fixtures via interfaces                                                        | xUnit encourages constructor injection for context sharing.                                                                                       |
| 09 | Code & Test Structure   | Internal test classes                      | ✅ Yes<br>🔹 Can derive from internal test base classes and access protected/internal members.<br>🔸 More encapsulated test hierarchies. | ❌ No<br>🔹 Test classes must be public.<br>🔸 Forces exposure of internal APIs or rework for testability. | NUnit can test internal classes and derive from internal/protected base classes that use internal production types. xUnit requires all test classes to be public. |
| 10 | Code & Test Structure   | Parameterized Tests                        | ✅ `[TestCase]`, `[TestCaseSource]` (type-safe)                                                             | ⚠️ `[Theory]`, `[MemberData]`, `[ClassData]`, `[InlineData]`                                           | xUnit's approach is more flexible but less type-safe and more verbose for complex cases.                                                         |
| 11 | Code & Test Structure   | Setup/Teardown                             | ✅ `[SetUp]`, `[TearDown]`, `[OneTimeSetUp]`, `[OneTimeTearDown]`                                           | ⚠️ Use constructor for setup, `IDisposable.Dispose` for teardown                                       | No attributes for setup/teardown in xUnit; use class/instance lifecycle.                                                                         |
| 12 | Code & Test Structure   | Test class instance per test               | ❌ No<br>🔹 Default: same instance for all test methods in a fixture.<br>🔸 Can opt-in to per-test instance. | ✅ Yes<br>🔹 Always new instance per test.<br>🔸 Ensures no state sharing between tests.                  | xUnit’s approach guarantees each test starts with a fresh instance, making tests safer by default.                                               |
| 13 | Data & Sources          | TestCaseSource / Theories                  | ✅ `[TestCaseSource]` supports any object, type-safe                                                        | ⚠️ `[MemberData]` requires `IEnumerable<object[]>`, less type-safe, static & serializable for cross-domain | NUnit is simpler and type-safe for complex test data, xUnit is more verbose/complex.                                                            |
| 14 | Data & Sources          | TestCaseSource/private static field         | ✅ Yes<br>🔹 Test data sources can be private or internal static fields, properties, or methods.            | ❌ No<br>🔹 `[MemberData]`/`[ClassData]` sources must be public static members.                          | NUnit allows private/internal static fields for `[TestCaseSource]`, supporting encapsulation of test data. xUnit requires public static members. |
| 15 | Execution & Discovery   | Parallel Test Execution                    | ⚠️ Opt-in (default: off, enable via config or attribute)                                                    | ⚠️ Opt-out (default: on, can disable via config or attribute)                                          | Both support fine-grained parallelism, but default behaviors differ.                                                                             |
| 16 | Execution & Discovery   | Skip at Runtime                            | ✅ `Assert.Ignore()` and `Assert.Inconclusive()`                                                            | ❌ No runtime skip; only static skip via `[Fact(Skip = "...")]`                                         | NUnit supports dynamic skip; xUnit requires custom extension or fails the test.                                                                  |
| 17 | Execution & Discovery   | Test Discovery                             | ⚠️ Test names can be duplicated (by params/data)                                                            | ❌ Test method names must be unique per class                                                          | xUnit may hang or misbehave with duplicate test method names.                                                                                    |
| 18 | Logging & Output        | Logging to Console                         | ✅ Console logging captured and displayed in all runners                                                    | ❌ `Console.WriteLine` output not reliably captured, esp. in Rider                                      | xUnit recommends using `ITestOutputHelper`, which is more complex, and Rider may not show console output.                                        |

---

### 🔎 Summary of Favorable Items

| Favorable                    | Rows (IDs)                                 | Count |
|-----------------------------|--------------------------------------------|-------|
| **NUnit favorable**         | 01, 02, 03, 05, 09, 10, 13, 14, 16, 18     | 10    |
| **xUnit favorable**         | 07, 12                                     | 2     |
| **Both favorable**          | 04, 08                                     | 2     |
| **Neutral or Mixed**        | 06, 11, 15, 17                             | 4     |
| **Total Rows**              |                                            | 18    |

- **NUnit favorable:** `[Category]` attribute, `[Explicit]` attribute, `[TestCaseSource]/[Test]` usage, test timeout, internal test classes, parameterized tests, test case source/theories, test case source/private static field, skip at runtime, logging to console.
- **xUnit favorable:** Per-test `Dispose` (resource cleanup), test class instance per test (isolation).
- **Both favorable:** No required class attribute for test discovery, fixture/context sharing.
- **Neutral or Mixed:** Async void test support, setup/teardown, parallel test execution, test discovery (duplicate names).

---

## Notes and Additional Differences

- **IDE Integration:** Rider and Visual Studio have top-tier NUnit support for console/log output; xUnit may require more setup for log capturing, especially in Rider.
- **Serialization in Data Sources:** xUnit requires test data for `[MemberData]` and `[ClassData]` to be serializable for cross-process test runners, making complex data harder.
- **Test Output:** xUnit recommends `ITestOutputHelper` for capturing test output, but this is not as intuitive as `Console.WriteLine` and may not be shown in all runners.
- **Test Initialization:** NUnit’s `[SetUp]` and `[TearDown]` run before and after each test; xUnit runs constructor and `Dispose`.
- **Parallelism Defaults:** NUnit is explicit (parallelism opt-in), xUnit is implicit (parallelism on by default).
- **TestCaseSource/MemberData:** NUnit’s approach is type-safe and can use instance/non-static sources; xUnit requires static members returning `IEnumerable<object[]>`, which is less type-safe and more verbose.

---

## References

- [NUnit Documentation](https://docs.nunit.org/)
- [xUnit Documentation](https://xunit.net/docs/getting-started/netfx/visual-studio)

