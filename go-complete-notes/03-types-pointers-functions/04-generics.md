# Generics

Go added **generics** (type parameters) in **Go 1.18** (March 2022). Generics let you write functions and types that work with any type while keeping type safety at compile time.

> 🔑 **Key idea:** Generics let one function serve many concrete types — with full compile-time checking and none of the boxing that `any` requires.

---

## Why Go Didnt Have Generics for So Long

Go designers intentionally left out generics for over a decade:

1. **Simplicity.** Go prioritizes code that is easy to read. Generics add complexity to the type system, the compiler, and the code itself.
2. **Compile-time safety without generics.** You cannot accidentally add a string to an integer. Go already guarantees this.
3. **Code clarity.** Before generics, you used interfaces or wrote type-specific code. Both make types explicit at the call site.
4. **Runtime cost.** Many languages implement generics through type erasure (Java) or boxing, which adds overhead. Go wanted generics without sacrificing performance.

The Go team eventually concluded that the pain of not having generics — particularly for container and collection types — outweighed the cost. The design in 1.18 was carefully constrained to avoid the complexity of generics in languages like Java, C++, or Rust.

---

## The Problem Generics Solve

Without generics, you need either duplication or type-unsafe `any`:

### Duplication

```go
func containsInt(slice []int, target int) bool {
    for _, v := range slice {
        if v == target { return true }
    }
    return false
}

func containsString(slice []string, target string) bool {
    for _, v := range slice {
        if v == target { return true }
    }
    return false
}
```

### Using `any` (type-unsafe)

```go
func contains(slice []any, target any) bool {
    for _, v := range slice {
        if v == target { return true }
    }
    return false
}
```

Problems:

- **No type safety.** You can pass `[]int` and a `string` as target — the compiler will not catch this.
- **Requires boxing.** Every value must be converted to `any`, which allocates memory.
- **Requires unboxing.** When you get a result back, you need a type assertion.
- **Loses compile-time guarantees.** The type relationship is invisible to the compiler.

> ⚠️ **Watch out:** `[]any` isn't a generalization of `[]int` — it's a different, slower type. Every element gets boxed and every read needs an assertion.

### Generics solve this

```go
func contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target { return true }
    }
    return false
}

func main() {
    nums := []int{1, 2, 3, 4, 5}
    fmt.Println(contains(nums, 3))       // true

    names := []string{"alice", "bob", "charlie"}
    fmt.Println(contains(names, "bob"))  // true
}
```

One function, full type safety, no allocation overhead.

---

## Type Parameters

Type parameters appear in square brackets after the function name:

```go
// [T any] declares T as a type parameter that can be anything
func Identity[T any](value T) T {
    return value
}

fmt.Println(Identity("hello"))        // hello
fmt.Println(Identity(42))             // 42
fmt.Println(Identity[int](42))        // 42 — explicit type
```

### Multiple Type Parameters

```go
func Pair[A any, B any](a A, b B) (A, B) {
    return a, b
}

p1, p2 := Pair("x", 42)
fmt.Println(p1, p2)   // x 42
```

### Type Inference

Go infers type parameters from function arguments. You rarely need to specify them:

```go
func mapSlice[T any, R any](s []T, f func(T) R) []R {
    result := make([]R, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

func main() {
    nums := []int{1, 2, 3}
    strs := mapSlice(nums, strconv.Itoa) // Go infers T=int, R=string
    fmt.Println(strs)                     // [1 2 3]
}
```

When Go cannot infer all type parameters (no args reference the type param), you must specify:

```go
func zero[T any]() T {
    var zero T
    return zero
}

n := zero[int]()   // must specify — no args to infer from
```

---

## Constraints

A **constraint** restricts which types a type parameter can be.

> 🔑 **Key idea:** A constraint is an interface that lists allowed types — `any`, `comparable`, unions like `int | float64`, or `~T` matching underlying types.

### Built-in Constraints

- **`any`** — equivalent to `interface{}`. Accepts every type.
- **`comparable`** — accepts types that support `==` and `!=` (numbers, strings, bools, pointers, channels, structs of comparable fields). Does NOT include slices, maps, or functions.

### Union Constraints

```go
// T can be int or float64
func Add[T int | float64](a, b T) T {
    return a + b
}
```

### Tilde (~) — Underlying Type

`~int` means "any type whose **underlying type** is int" (including named types like `type Age int`):

> 🧠 **Think of it as:** The tilde matches the type *family* (`~int` accepts every type built on `int`), while `int` alone matches only the exact type.

```go
func Double[T ~int | ~float64](x T) T {
    return x * 2
}

type MyInt int
fmt.Println(Double(MyInt(5)))   // 10 — works because MyInt's underlying type is int
```

Without `~`, only exact types in the union are allowed. With `~`, any type sharing the underlying type works.

### Custom Constraint Interfaces

```go
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64
}

func Sum[T Number](values []T) T {
    var total T
    for _, v := range values {
        total += v
    }
    return total
}

fmt.Println(Sum([]int{1, 2, 3}))       // 6
fmt.Println(Sum([]float64{1.5, 2.5}))  // 4
```

### Constraints with Methods

A constraint can include methods that the type parameter must implement:

```go
type Stringer interface {
    ~string
    String() string   // method requirement
}

func greet[T Stringer](people []T) {
    for _, p := range people {
        fmt.Println("Hello,", p.String())
    }
}
```

> In Go, constraints **are** interfaces. This dual nature unifies generics with the existing interface system.

---

## Type Sets

A type set is the set of types defined by a constraint. The compiler checks that every operation inside a generic function is valid for **every** type in the type set:

```go
type Number interface {
    ~int | ~float64
}

func average[T Number](nums []T) float64 {
    var total float64
    for _, n := range nums {
        total += float64(n)   // valid for all types in Number
    }
    return total / float64(len(nums))
}
```

If you added `~string`, the function would fail to compile because `float64("hello")` is not valid.

---

## Generic Types (Structs)

You can parameterize types too:

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    last := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return last, true
}

func main() {
    intStack := &Stack[int]{}
    intStack.Push(10)
    intStack.Push(20)
    v, ok := intStack.Pop()
    fmt.Println(v, ok)   // 20 true

    strStack := &Stack[string]{}
    strStack.Push("hello")
}
```

Each `Stack[int]` and `Stack[string]` is a completely separate type.

### Important Limitation: No Generic Methods

Methods cannot have type parameters separate from the receiver. Only the receiver (the type itself) can be generic:

> ⚠️ **Gotcha:** Methods can't declare their own type parameters. If a method needs its own `[U any]`, refactor it into a standalone generic function.

```go
type Box[T any] struct{ v T }

// Allowed — T comes from the type
func (b Box[T]) Get() T { return b.v }

// NOT allowed — method-level type parameter
// func (b Box[T]) Convert[U any]() U { ... }
```

If you need a method with an additional type parameter, make it a standalone generic function.

---

## Generic Slices and Maps

Standard library helper packages (Go 1.21+):

```go
import (
    "maps"
    "slices"
)

nums := []int{3, 1, 2}
slices.Sort(nums)           // [1 2 3]
slices.Contains(nums, 2)    // true
slices.Max(nums)            // 3

m := map[string]int{"a": 1, "b": 2}
maps.Clone(m)
// maps.Keys, maps.Values, maps.Equal, etc.
```

Use these instead of rolling your own.

---

## Common Generic Patterns

### Generic Map/Filter/Reduce

```go
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

func Filter[T any](s []T, pred func(T) bool) []T {
    var result []T
    for _, v := range s {
        if pred(v) {
            result = append(result, v)
        }
    }
    return result
}

func Reduce[T, U any](s []T, init U, f func(U, T) U) U {
    acc := init
    for _, v := range s {
        acc = f(acc, v)
    }
    return acc
}
```

### Generic Set

```go
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable](items ...T) *Set[T] {
    s := &Set[T]{items: make(map[T]struct{})}
    for _, item := range items {
        s.items[item] = struct{}{}
    }
    return s
}

func (s *Set[T]) Add(item T)           { s.items[item] = struct{}{} }
func (s *Set[T]) Contains(item T) bool { _, ok := s.items[item]; return ok }
func (s *Set[T]) Remove(item T)        { delete(s.items, item) }
```

### Generic Result Type

```go
type Result[T any] struct {
    value T
    err   error
}

func Ok[T any](value T) Result[T]     { return Result[T]{value: value} }
func Err[T any](err error) Result[T]  { return Result[T]{err: err} }

func (r Result[T]) Unwrap() (T, error) { return r.value, r.err }

func FlatMapResult[T any, R any](r Result[T], fn func(T) Result[R]) Result[R] {
    if r.err != nil {
        return Err[R](r.err)
    }
    return fn(r.value)
}
```

### Generic "Must" Helper

```go
func Must[T any](value T, err error) T {
    if err != nil {
        panic(err)
    }
    return value
}

v := Must(strconv.Atoi("42"))   // 42 or panics
```

---

## Generics vs Interfaces

This is the central design decision. When should you use generics, and when should you use interfaces?

> 🧠 **Memory aid:** Interfaces = same shape, different behavior. Generics = different types, same behavior. Pick the one that matches your problem.

### Use Interfaces When

You need **polymorphic behavior** — different types doing different things:

```go
type Writer interface {
    Write(data []byte) (int, error)
}

func WriteData(w Writer, data []byte) error {
    _, err := w.Write(data)
    return err
}
```

With interfaces, the type determines the behavior. `File.Write` does something different from `Buffer.Write`. The caller does not care what the concrete type is.

### Use Generics When

You need to operate on values of the **same logical kind**, where the type is incidental:

```go
func First[T any](slice []T) (T, bool) {
    if len(slice) == 0 {
        var zero T
        return zero, false
    }
    return slice[0], true
}
```

With generics, the behavior is the same for all types. `First([]int{1, 2, 3})` and `First([]string{"a", "b"})` do the same thing.

### Decision Table

| Question | Use Interfaces | Use Generics |
|----------|---------------|--------------|
| Do different types need different behavior? | Yes | No |
| Are you defining a contract types must implement? | Yes | No |
| Are you writing code that works identically for multiple types? | No | Yes |
| Do you need compile-time type safety? | No | Yes |
| Are you wrapping or transforming values of the same kind? | No | Yes |

### Constraints Are Interfaces

In Go, constraints are interfaces. A type parameter can be restricted to types that satisfy an interface:

```go
type JSONMarshaler interface {
    MarshalJSON() ([]byte, error)
}

func MarshalAll[T JSONMarshaler](items []T) ([][]byte, error) {
    var results [][]byte
    for _, item := range items {
        data, err := item.MarshalJSON()
        if err != nil { return nil, err }
        results = append(results, data)
    }
    return results, nil
}
```

This dual nature is how Go unifies generics with the existing interface system.

### Generic Repository Interface

```go
type Repository[T any] interface {
    Get(id int) (T, error)
    Save(item T) error
}
```

The interface is the contract. The generic type parameter makes the contract reusable across different domain types.

---

## Constraints in Standard Library

- `golang.org/x/exp/constraints`: `Ordered`, `Signed`, `Unsigned`, `Integer`, `Float`
- Go 1.21+: `cmp` package for `cmp.Ordered`

```go
func Min[T cmp.Ordered](a, b T) T {
    if a < b { return a }
    return b
}
```

> `slices` and `maps` are in the standard library (Go 1.21+). `constraints` is in `golang.org/x/exp`.

---

## When NOT to Use Generics

- **Don't use generics when an interface works.** If different types should behave differently, an interface is the right abstraction.
- **Don't use generics for simple type safety.** If you only have two or three types and the function is trivial, concrete functions are easier to read.

> 💡 **Pro tip:** Ask first: "Does an interface or a plain function already handle this?" Only reach for a type parameter when it removes genuine duplication.

- **Don't use generics when the constraint would be `any`.** You are not gaining type safety, just adding syntax.
- **Don't confuse `any` with "any type is acceptable."** When `T` is `any`, the compiler only allows operations valid for all types (which is almost nothing).
- **Don't forget generics are statically typed.** When `T` is an interface type and `comparable` is used, comparisons between different dynamic types panic at runtime.

---

## Performance Notes

| Approach | Allocation | Type Safety | Runtime Cost |
|----------|-----------|-------------|--------------|
| `any` (interface{}) | Box every value (~30 ns each) | None | Type assertion per use |
| Generics | No boxing, direct calls | Full compile-time | Stspecialization per type (GC shape stashing) |
| Concrete per-type | No boxing | Full | Zero overhead but code duplication |
| Reflection | Box + reflect overhead | None | ~100-1000x slower |

Go generics use **GC shape stsharing** — types with the same GC shape share code, avoiding code explosion while still getting type safety. This is faster than interface boxing but slightly slower than fully monomorphized generics (like C++ templates).

---

## Modern Practices

- Use generics for type-safe containers and utilities only when they reduce real duplication
- Prefer `any` and `comparable` as constraints before building custom ones
- Use `slices` and `maps` standard library functions (Go 1.21+) instead of rolling your own
- Use `~T` in constraints to match underlying types when named types need inclusion
- Keep constraints minimal — accept only what the function actually requires
- Generics require Go 1.18+; ensure toolchain and go.mod reflect the minimum version

## Common Mistakes

- Replacing a simple interface with generics where `any` or a small interface would suffice
- Over-constraining type parameters with more restrictions than needed
- Using `comparable` when the function never compares values with `==` or `!=`
- Writing generic methods on named types (not allowed — only the receiver type can be generic)
- Assuming type inference always works (sometimes you must specify types explicitly)
- Mixing generics with `reflect` — the two operate at different levels
- Overusing generics — prefer concrete types and interfaces when generics add no real value

## Key Takeaways

1. Generics (Go 1.18+) add **type parameters** `[T any]`.
2. Use **constraints** (`any`, `comparable`, unions, `~`, custom interfaces).
3. Generic **structs** let you build reusable containers (Stack, Set, etc.).
4. Methods cannot add their own type params — only the receiver type can be generic.
5. Favor type inference; specify types only when needed.
6. Use `slices`/`maps` stdlib helpers (1.21+).
7. Generics and interfaces serve different purposes: generics for same-behavior-different-types, interfaces for same-interface-different-behavior.
8. Generics trade some clarity for reusability — use them when they truly reduce duplication.

## Next

Continue to [05-error-handling.md](05-error-handling.md).
