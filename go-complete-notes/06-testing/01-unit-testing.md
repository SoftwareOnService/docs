# Unit Testing in Go

Testing is a first-class citizen in Go. The language ships with a built-in
`testing` package and full toolchain support — no third-party framework required.
Test files end in `_test.go`, test functions start with `Test`, and `go test`
compiles, links, and executes them automatically. Test files are excluded from
regular `go build`, so test code never pollutes your production binary.

---

## Test File Conventions

- Test files must end with `_test.go` (e.g., `math_test.go` tests `math.go`).
- Test functions must be exported (start with uppercase) and follow the pattern
  `TestXxx(*testing.T)` where `Xxx` does not start with a lowercase letter.
- Test files can import different packages than production code (e.g.,
  `net/http/httptest`) without requiring those dependencies in production.

```
mathutil/
    add.go
    add_test.go
```

> 🔑 **Key idea:** The `_test.go` suffix is the signal. Go's toolchain automatically
> isolates test files from production builds, so your test code can never leak into
> a deployed binary.

---

## Running Tests

```sh
go test                              # run all tests in the current package
go test ./...                        # run all tests recursively
go test -v                           # verbose output (shows each test name)
go test -run TestAdd                 # run only tests whose name matches regex
go test -run "TestAdd/positive"      # run a specific subtest
go test -timeout 30s                 # set a timeout (default: 10 minutes)
go test -short                       # skip long-running tests
go test -count 3                     # run each test N times
go test -shuffle on                  # randomize execution order
go test -race                        # enable the race detector
go test -count=1                     # disable test caching
```

> 💡 **Pro tip:** Go caches test results when inputs are unchanged, which can confuse
> you when debugging. Reach for `go test -count=1 ./...` to force a fresh real run.

The `-short` flag lets tests skip themselves:

```go
func TestIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }
    // ... long-running test
}
```

---

## A Basic Unit Test

```go
package mathops

import "testing"

func TestAdd(t *testing.T) {
    got := Add(2, 3)
    want := 5
    if got != want {
        t.Errorf("Add(2,3) = %d; want %d", got, want)
    }
}
```

Output on failure:

```
--- FAIL: TestAdd (0.00s)
    math_test.go:9: Add(2,3) = 6; want 5
FAIL
```

---

## Table-Driven Tests

The idiomatic Go pattern for testing multiple cases is a table of
input/expected pairs. Instead of writing separate test functions for each case,
define a slice of test cases and iterate over them.

```go
package mathops

import "testing"

func TestAddTable(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive numbers", 2, 3, 5},
        {"negatives", -2, -3, -5},
        {"zero", 0, 0, 0},
        {"mixed signs", -1, 7, 6},
        {"large numbers", 1_000_000, 2_000_000, 3_000_000},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d,%d) = %d; want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

Why table-driven tests are preferred:

1. **Conciseness**: Adding a new case is one line in the slice.
2. **Consistency**: All cases follow the same structure.
3. **Readability**: The test logic is written once; the variations are data.
4. **Subtest support**: Each case runs as a named subtest.

Anti-pattern — separate functions for each case:

```go
// Bad: repetitive, does not scale
func TestAddPositive(t *testing.T) {
    if Add(2, 3) != 5 { t.Error("failed") }
}
func TestAddZero(t *testing.T) {
    if Add(0, 0) != 0 { t.Error("failed") }
}
```

When multiple functions share similar test structures, define the struct once:

```go
type testCase struct {
    name     string
    input    string
    expected string
}
```

---

## Subtests with `t.Run`

`t.Run(name, fn)` creates **subtests** with readable names. This enables
selective execution via `-run`, independent failure reporting, and shared
setup per group.

### Selective Execution

```go
func TestParseConfig(t *testing.T) {
    t.Run("valid JSON", func(t *testing.T) { /* ... */ })
    t.Run("invalid JSON", func(t *testing.T) { /* ... */ })
    t.Run("missing required field", func(t *testing.T) { /* ... */ })
}
```

```sh
go test -run TestParseConfig/missing_required_field
```

### Nested Subtests

Subtests can be nested for finer organization. Run all GET tests with
`go test -run TestHTTPServer/GET`:

```go
func TestHTTPServer(t *testing.T) {
    t.Run("GET", func(t *testing.T) {
        t.Run("/users", func(t *testing.T) { /* test GET /users */ })
        t.Run("/users/:id", func(t *testing.T) { /* test GET /users/:id */ })
    })
    t.Run("POST", func(t *testing.T) {
        t.Run("/users", func(t *testing.T) { /* test POST /users */ })
    })
}
```

### Parallel Subtests

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"case 1", 1, 2, 3},
        {"case 2", 100, 200, 300},
        {"case 3", -1, -2, -3},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // runs concurrently with other parallel subtests
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

Caveats: In Go 1.22+, loop variables are per-iteration; before that, shadow
with `tt := tt`. Parallel subtests must not share mutable state without
synchronization. `t.Parallel()` pauses execution until the parent function
returns, then runs concurrently.

> ⚠️ **Watch out:** Before Go 1.22, the loop variable is reused across iterations —
> that's the classic `tt := tt` bug. And never let parallel subtests scribble on a
> shared map or slice without a mutex.

---

## Test Helpers with `t.Helper()`

`t.Helper()` marks the calling function as a test helper. Without it, error
messages point to the line inside the helper. With it, the error points to the
line in the test that called the helper.

```go
func assertEqual(t *testing.T, got, want int) {
    t.Helper()   // report the failure at the CALLER's line, not here
    if got != want {
        t.Errorf("got %d; want %d", got, want)
    }
}

func TestAdd(t *testing.T) {
    assertEqual(t, Add(2, 3), 5)
}
```

Generic helpers with Go 1.21+ generics:

```go
func equal[T comparable](t *testing.T, got, want T) {
    t.Helper()
    if got != want {
        t.Errorf("got %v; want %v", got, want)
    }
}
```

Use `t.Helper()` in any function that is called from test functions, reports
errors, or encapsulates assertion logic.

> 🧠 **Memory aid:** Think of `t.Helper()` as "blame the caller." It pushes the
> failure report up the call stack to the line that actually called the helper,
> not the helper's own internals.

---

## The `testing.T` Methods

| Method | Purpose |
|--------|---------|
| `Error(args...)` | Log + mark test as failed (continue) |
| `Errorf(format, args)` | Log formatted + fail (continue) |
| `Fatal(args...)` | Log + fail + **stop** test immediately |
| `Fatalf(format, args)` | Log formatted + fail + stop |
| `Log(args...)` | Log (shown with `-v`) |
| `Logf(format, args)` | Log formatted |
| `Helper()` | Mark helper (hide from failure stack) |
| `Parallel()` | Run test in parallel |
| `Cleanup(func())` | Register teardown (runs on failure too) |
| `Skip(args...)` | Skip test |
| `Name()` | Current test name |

### `t.Fatal` vs `t.Error`

- Use `t.Fatal` when subsequent assertions depend on a condition being true.
  `t.Fatal` calls `runtime.Goexit()`, which stops the goroutine — code after it
  does NOT execute, but deferred functions DO.
- Use `t.Error` when the failure is informational and other checks are still
  meaningful.

```go
func TestUserCreation(t *testing.T) {
    user, err := CreateUser("alice")
    if err != nil {
        t.Fatalf("CreateUser failed: %v", err) // stop: can't check user.Name
    }
    if user.Name != "alice" {
        t.Errorf("Name = %q; want %q", user.Name, "alice") // continue
    }
    if user.ID == 0 {
        t.Error("expected non-zero ID") // continue
    }
}
```

> 🔑 **Key idea:** `t.Fatal` vs `t.Error` is "can I keep going?" If the rest of the
> test would panic or misreport without this check, `Fatal`. If it's one check among
> many, `Error`.

---

## Setup and Teardown

### `TestMain` (package-level)

`TestMain` runs once before any test in the package. It controls the entire
test lifecycle.

```go
package dbutil

import (
    "os"
    "testing"
)

func TestMain(m *testing.M) {
    setupDB()
    code := m.Run()
    teardownDB()
    os.Exit(code)
}
```

Rules: only one `TestMain` per package; it must call `m.Run()` and `os.Exit`
with the result.

### Per-Test Setup with Helpers

```go
func setupServer(t *testing.T) *httptest.Server {
    t.Helper()
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"status":"ok"}`))
    }))
    t.Cleanup(srv.Close)
    return srv
}
```

`t.Cleanup` (Go 1.14+) registers a function to run when the test finishes
(LIFO order, like `defer`). It is the idiomatic replacement for `defer` in
test helpers not called directly from the test function — it runs even if the
test fails.

> 🔑 **Remember:** `t.Cleanup` runs even when the test fails or panics, so resources
> never leak. Prefer it over `defer` inside helpers — the cleanup is tied to the
> test, not the helper's stack frame.

---

## Testing Error-Returning Code

### Checking Error Values with `errors.Is`

```go
// Bad: string comparison (fragile)
if err != nil && err.Error() == "file not found" { ... }

// Good: use errors.Is (unwraps error chains)
if errors.Is(err, os.ErrNotExist) { ... }
```

### Checking Error Types with `errors.As`

```go
func TestValidation(t *testing.T) {
    err := ValidateUser("")
    if err == nil {
        t.Fatal("expected validation error; got nil")
    }

    var valErr *ValidationError
    if !errors.As(err, &valErr) {
        t.Fatalf("expected ValidationError; got %T", err)
    }
    if valErr.Field != "name" {
        t.Errorf("Field = %q; want %q", valErr.Field, "name")
    }
}
```

### Table-Driven Error Tests

```go
func TestDivide(t *testing.T) {
    tests := []struct {
        name    string
        a, b    float64
        want    float64
        wantErr error
    }{
        {"valid division", 10, 2, 5, nil},
        {"divide by zero", 10, 0, 0, ErrDivisionByZero},
        {"divide negative", -10, 2, -5, nil},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)
            if tt.wantErr != nil {
                if !errors.Is(err, tt.wantErr) {
                    t.Errorf("error = %v; want %v", err, tt.wantErr)
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if got != tt.want {
                t.Errorf("Divide(%v, %v) = %v; want %v", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

---

## Comparing Structs and Slices

```go
import "reflect"

func assertDeepEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if !reflect.DeepEqual(got, want) {
        t.Errorf("got %#v, want %#v", got, want)
    }
}
```

For comparing strings or complex diffs, use `cmp.Diff` from
`github.com/google/go-cmp`:

```go
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("mismatch (-want +got):\n%s", diff)
}
```

> 🧠 **Think of it as:** `reflect.DeepEqual` says "are these equal?" — fine for
> structs, terrible for readable output. `cmp.Diff` says "what's different?" — it
> prints a tidy `-want +got` diff you can paste into a bug report.

---

## Coverage

```sh
go test -cover                         # display coverage percentage
go test -coverprofile=coverage.out     # generate a coverage profile
go tool cover -html=coverage.out       # view coverage in browser (HTML)
go tool cover -func=coverage.out       # view coverage by function
```

The HTML view color-codes each line:

- **Green**: covered by at least one test.
- **Red**: not covered by any test.
- **Gray**: not executable (declarations, comments).

Coverage measures code **execution**, not test quality — high coverage with poor
assertions provides false confidence. Many CI pipelines enforce minimum coverage
thresholds by parsing `go tool cover -func` output.

> ⚠️ **Watch out:** 100% coverage does not mean 100% correctness. A test that calls a
> function but never asserts its result still "covers" the line — chase meaningful
> assertions, not the percentage.

---

## Benchmarking

Benchmark functions start with `Benchmark`, take `*testing.B`, and run a loop
`b.N` times (managed by the framework — never hardcode `b.N`).

> 🔑 **Key idea:** `b.N` is chosen and re-tuned by the framework until the run is
> statistically meaningful. You write the loop, the harness decides how many times
> to run it — that's why you must never hardcode the count.

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2)
    }
}
```

### Running Benchmarks

```sh
go test -bench=.                              # run all benchmarks
go test -bench=BenchmarkAdd                   # run matching benchmarks
go test -bench=. -benchtime 5s                # run for 5 seconds each
go test -bench=. -benchtime 10000x            # run for fixed iterations
go test -bench=. -benchmem                    # include memory allocation stats
```

### Benchmark Output

```
BenchmarkAdd-8     1000000000    0.25 ns/op    0 B/op    0 allocs/op
```

Columns: benchmark name with CPU count, number of iterations, time per
operation, bytes allocated per operation, number of allocations per operation.

### Comparing Approaches

```go
func BenchmarkConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "x"     // slow (immutable strings)
        }
        _ = s
    }
}

func BenchmarkBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("x")
        }
        _ = sb.String()
    }
}
```

Use `benchstat` to compare results across runs. Save results with `-count 5`
for statistical significance, then `benchstat old.txt new.txt`.

> 💡 **Pro tip:** Never trust a single benchmark run. Collect several (`-count 5` or
> more) and run them through `benchstat` — it reports the median and flags
> statistically significant changes, so you can't fool yourself with one lucky run.

### Setup and Teardown in Benchmarks

Use `b.ResetTimer()` to exclude setup time:

```go
func BenchmarkProcessLargeDataset(b *testing.B) {
    data := generateDataset(10_000_000)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        Process(data)
    }
}
```

### Parallel Benchmarks

```go
func BenchmarkConcurrentMap(b *testing.B) {
    m := NewConcurrentMap()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            m.Set("key", "value")
        }
    })
}
```

`b.RunParallel` distributes work across `GOMAXPROCS` goroutines. Use `b.Run`
with size suffixes for benchmarks that vary input size.

---

## Fuzz Testing (Go 1.18+)

Fuzzing automatically generates random inputs to find edge-case bugs that manual
testing might miss. Functions named `Fuzz` have the signature
`func FuzzXxx(f *testing.F)`.

```go
package parser

import "testing"

func FuzzParse(f *testing.F) {
    // seed corpus — known inputs to start with
    f.Add("hello world")
    f.Add("")
    f.Add("12345")
    f.Add("special: characters!@#$%")

    f.Fuzz(func(t *testing.T, input string) {
        result, err := Parse(input)
        if err != nil {
            return
        }

        serialized := result.Serialize()
        reparsed, err := Parse(serialized)
        if err != nil {
            t.Errorf("round-trip failed: Parse(%q) succeeded, "+
                "Serialize() = %q, Parse(serialized) failed: %v",
                input, serialized, err)
        }
        if reparsed != result {
            t.Errorf("round-trip mismatch: got %v; want %v", reparsed, result)
        }
    })
}
```

### Running Fuzz Tests

```sh
go test -fuzz FuzzParse                 # run the fuzzer
go test -fuzz FuzzParse -fuzztime 30s   # run for limited time
go test -fuzz FuzzParse -fuzztime 10000x # run for fixed iterations
go test -run FuzzParse                  # run seeds only (no fuzzing)
```

The fuzzer starts with seeds from `f.Add`, mutates them, and runs `f.Fuzz` on
each. Failures are saved to `testdata/fuzz/<FuzzTestName>/` and replayed on
subsequent runs. Only one `f.Fuzz` call per test; each argument after
`*testing.T` needs a matching `f.Add` seed.

---

## Example Tests (Doc Examples)

Example functions serve double duty: they are executable tests AND
documentation that appears in `go doc` and on pkg.go.dev.

```go
package mathutil

import "fmt"

func ExampleAdd() {
    result := Add(2, 3)
    fmt.Println(result)
    // Output: 5
}

func ExampleAdd_negative() {
    fmt.Println(Add(-1, -3))
    // Output: -4
}

func ExampleMap() {
    m := map[string]int{"a": 1, "b": 2}
    for k, v := range m {
        fmt.Printf("%s: %d\n", k, v)
    }
    // Unordered output:
    // a: 1
    // b: 2
}
```

Rules:

- Function name must start with `Example`, no `*testing.T` parameter.
- The `// Output:` comment is required for verification.
- Use `_` as a word separator for suffixes.
- Use `// Unordered output:` for non-deterministic order.
- Without an output comment the example compiles but does not run during testing.

> 🧠 **Think of it as:** An example test is a contract displayed on pkg.go.dev AND
> enforced by `go test`. The `// Output:` comment is both the documentation and the
> expected result — change the code and the test catches it.

---

## Testing with Temporary Files and Dirs

```go
func TestWriteFile(t *testing.T) {
    dir := t.TempDir()   // auto-created, auto-cleaned up
    path := filepath.Join(dir, "test.txt")

    err := os.WriteFile(path, []byte("hi"), 0644)
    if err != nil {
        t.Fatal(err)
    }

    data, _ := os.ReadFile(path)
    if string(data) != "hi" {
        t.Fatalf("got %q", string(data))
    }
}
```

For structured test data, load fixtures from a `testdata/` directory (Go
tooling ignores it for compilation):

```go
func loadFixture(t *testing.T, name string) []byte {
    t.Helper()
    path := filepath.Join("testdata", name)
    data, err := os.ReadFile(path)
    if err != nil {
        t.Fatalf("read fixture %s: %v", path, err)
    }
    return data
}
```

---

## The Race Detector

Always run tests with the race detector to catch concurrency bugs:

```sh
go test -race ./...
```

This compiles with the ThreadSanitizer and reports data races at runtime. It
catches bugs that are almost impossible to find by code review alone.

> ⚠️ **Gotcha:** `-race` only detects races in code that actually runs concurrently
> during the test. Run it under the race detector, but remember a clean report
> doesn't prove the absence of races — just that none happened this time.

---

## Exercises

### Exercise 1: Basic Table-Driven Test

Write a `Factorial` function and a table-driven test for it:

```go
func Factorial(n int) (int, error)
```

Tests should cover: 0, 1, 5, 10, and negative inputs. Include error checking
with `errors.Is`.

### Exercise 2: Subtests with `t.Run`

Write a `Partition` function that splits a slice into chunks:

```go
func Partition(slice []int, size int) ([][]int, error)
```

Write subtests for: normal partitioning, size larger than slice, size equal to
slice, size of 1, and size of 0 or negative (error).

### Exercise 3: Test Helper with `t.Helper()`

Write a `Contains` function for strings and a reusable `assertContains` helper
that calls `t.Helper()` across at least 5 test cases.

### Exercise 4: Benchmark

Write a `ReverseString(s string) string` function and benchmark three
approaches: byte-based reversal, rune-based reversal (using
`utf8.DecodeRuneInString`), and `[]rune` conversion. Compare with
`go test -bench . -benchmem`.

### Exercise 5: Fuzz Test

Write an `IsPalindrome(s string) bool` function. Write a fuzz test with at
least 3 seed inputs and an invariant: if `s` is a palindrome, then
`IsPalindrome` should return true when called again on the processed version.

### Exercise 6: TestMain and Setup

Write a `Cache` type with `Get` and `Set` methods backed by a map. Use
`TestMain` for package-level setup, `t.Cleanup` to reset the cache, and
`t.Parallel()` for concurrent access.

### Exercise 7: Error Testing

Write a `ValidateAge(age int) error` that returns `ErrTooYoung` if age < 0,
`ErrTooOld` if age > 150, and nil for valid ages. Write table-driven tests that
verify each error type using `errors.Is`.

---

## Summary Table

| Concept | Function/Flag | Purpose |
|---|---|---|
| Test function | `TestXxx(*testing.T)` | Define a unit test |
| Subtest | `t.Run()` | Organize and filter tests |
| Parallel | `t.Parallel()` | Run subtests concurrently |
| Helper | `t.Helper()` | Improve error reporting |
| Cleanup | `t.Cleanup()` | Register teardown logic |
| Fatal | `t.Fatal()` | Stop test immediately |
| Coverage | `-coverprofile` | Measure code coverage |
| Benchmark | `BenchmarkXxx(*testing.B)` | Measure performance |
| Fuzz | `FuzzXxx(*testing.F)` | Find bugs with random input |
| Example | `ExampleXxx()` | Document and test simultaneously |
| Skip | `t.Skip()` | Conditionally skip a test |
| Short mode | `-short` / `testing.Short()` | Skip long tests |

---

## Modern Practices

- Use table-driven tests with `t.Run` subtests for readable, maintainable cases.
- Call `t.Helper()` in assertion helpers so failure locations point to the caller.
- Prefer `t.Cleanup` (Go 1.14+) over `defer` for teardown that should run
  regardless of test path.
- Always run `go test -race ./...` in CI to catch concurrency bugs early.
- Add fuzz tests (`Fuzz*` functions) for parsing and validation code (Go 1.18+).
- Write `Example*` tests with `// Output:` comments to serve as runnable
  documentation.
- Use `go-cmp` only when the stdlib assertion pattern becomes too verbose.
- Load structured test data from `testdata/` fixtures.

---

## Common Mistakes

- **Testing implementation details** instead of observable behavior — tests break
  on refactors for no reason.
- **Sharing mutable state** across parallel tests without proper synchronization.
- **Skipping the `-race` flag** and shipping code with hidden data races.
- **Not using `t.Helper()`**, causing failure messages to point at the helper
  instead of the caller.
- **Skipping cleanup** with `defer` or `t.Cleanup`, leaving resources open.
- **Writing tests that rely on execution order** (non-independent test cases).
- **Asserting `err != nil` without including the error message** in the output.
- **Using `time.Sleep` or timing assumptions** that cause flaky tests under load.
- **Using `fmt.Println` for debugging** — use `t.Log` instead (suppressed by
  default, shown with `-v`).
- **Swallowing errors** (`resp, _ := http.Get(url)`) — hides bugs.
- **Assertions without context** — always include input, actual, and expected.
- **Loop variable capture** — before Go 1.22, shadow with `tt := tt`; in 1.22+
  this is no longer necessary.

---

## Key Takeaways

1. Test files end in `_test.go`; functions start with `Test`, take `*testing.T`.
2. **Table-driven tests** are idiomatic (struct of cases + `t.Run` subtests).
3. Use `t.Errorf` to fail-and-continue; `t.Fatalf` to fail-and-stop.
4. `go test -cover`, `go test -bench=`, `go test -fuzz=`, `go test -race`.
5. Benchmark functions start with `Benchmark`, use `b.N` — never hardcode it.
6. Fuzzing (`Fuzz*` functions) finds edge-case bugs automatically.
7. Example tests (`ExampleXxx` + `// Output:`) double as documentation.
8. Always use `t.Helper()` and `t.Cleanup` for maintainable test suites.

---

## Next

Continue to [02-tdd.md](02-tdd.md) for test-driven development in Go.
