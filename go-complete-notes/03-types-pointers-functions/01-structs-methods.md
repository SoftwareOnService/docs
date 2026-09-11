# Structs, Methods, and Composition

Go has no classes. Instead, it uses **structs** for data grouping, **methods** for behavior, and **composition (embedding)** for reuse. This trifecta replaces inheritance-based OOP with a simpler, more composable model.

> 🔑 **Key idea:** Go trades classes and inheritance for structs, methods, and embedding. Behavior is composed, not inherited.

---

## Structs

A **struct** is a composite type that groups zero or more named fields of possibly different types.

```go
type User struct {
    Name  string
    Age   int
    Email string
}
```

Fields can be of any type — other structs, slices, maps, pointers, functions:

```go
type Company struct {
    Name      string
    Employees []User
    Founded   int
    Website   *string  // pointer to string (can be nil)
}
```

### Creating Struct Values

**Zero value** — no initialization gets the zero value for each field:

```go
var u User
fmt.Println(u)          // {  0}
fmt.Println(u.Name)     // ""
```

**Composite literal (positional):**

```go
u := User{"Alice", "alice@example.com", 30}
```

> Fragile — if you add/reorder fields, every positional literal breaks. Avoid this.

**Composite literal (named fields, preferred):**

> 💡 **Note:** Prefer named-field literals `User{Name: "Alice"}` — positional literals `User{"Alice", ...}` silently break when a field is added or reordered.

```go
u := User{
    Name:  "Alice",
    Email: "alice@example.com",
    Age:   30,
}
```

Missing fields get zero values:

```go
u := User{Name: "Alice"}   // Age is 0, Email is ""
```

**Pointer to a struct:**

```go
u := &User{Name: "Alice", Age: 30}
```

This allocates a `User` on the heap and returns a pointer. Returning a pointer to a local variable is perfectly safe — the GC manages the memory.

### Accessing Fields

Dot notation — same whether the variable is a value or a pointer:

```go
u := User{Name: "Alice", Age: 30}
fmt.Println(u.Name)   // Alice
u.Age = 31

p := &User{Name: "Bob"}
fmt.Println(p.Name)   // Bob — Go auto-dereferences (*p).Name
p.Age = 25
```

You **cannot** use `.` on a nil pointer:

```go
var u *User
fmt.Println(u.Name)   // panic: nil pointer dereference
```

### Structs Are Value Types

Assigning copies the entire struct:

> 🧠 **Memory aid:** Assigning a struct is like making a photocopy — independent pages. Only slices, maps, and pointers inside it are shared.

```go
original := User{Name: "Alice", Age: 30}
copy := original
copy.Name = "Bob"

fmt.Println(original.Name) // Alice — unchanged
fmt.Println(copy.Name)     // Bob
```

Passing to a function also copies:

```go
func birthday(u User) {
    u.Age++
}

func main() {
    u := User{Name: "Alice", Age: 30}
    birthday(u)
    fmt.Println(u.Age)   // 30 — unchanged!
}
```

To modify the original, pass a pointer:

```go
func birthday(u *User) {
    u.Age++
}

func main() {
    u := User{Name: "Alice", Age: 30}
    birthday(&u)
    fmt.Println(u.Age)   // 31
}
```

> **Why value semantics?** Go defaults to value semantics for safety — each goroutine gets its own copy, reducing data races. Opt into pointer semantics explicitly when you need sharing.

### Copying Structs with Shared References

Beware of copying structs that contain slices, maps, or pointers — the copy shares underlying data:

> ⚠️ **Watch out:** A struct copy is shallow — slices and maps inside it still share backing data with the original. Modify one, and the "copy" changes too.

```go
type Person struct {
    Tags []string
}

a := Person{Tags: []string{"go", "dev"}}
b := a           // copies the slice header — SAME backing array
b.Tags[0] = "rust"
fmt.Println(a.Tags[0])   // rust — shared!
```

Deep copy:

```go
b.Tags = append([]string(nil), a.Tags...)
```

### Anonymous (Unnamed) Structs

```go
point := struct {
    X, Y int
}{X: 5, Y: 10}
fmt.Println(point.X)   // 5
```

Common in HTTP handlers for one-off payloads:

```go
var payload struct {
    Username string `json:"username"`
    Password string `json:"password"`
}
json.NewDecoder(r.Body).Decode(&payload)
```

### Comparing Structs

Structs are comparable with `==` if all fields are comparable:

```go
type Point struct{ X, Y int }
a := Point{1, 2}
b := Point{1, 2}
fmt.Println(a == b)   // true
```

Structs with slices, maps, or function fields are **not** comparable:

```go
type Team struct {
    Name    string
    Members []string
}
a := Team{Name: "Go", Members: []string{"Alice"}}
b := Team{Name: "Go", Members: []string{"Alice"}}
// fmt.Println(a == b)   // compile error!
```

---

## Struct Tags

Tags are metadata strings attached to fields, read by encoders via reflection:

```go
type User struct {
    ID       int    `json:"id" db:"id"`
    Name     string `json:"name" validate:"required"`
    Email    string `json:"email,omitempty" validate:"required,email"`
    Password string `json:"-" validate:"required"`
    Age      int    `json:"age,string"`
}
```

Format: `key:"value"`. Multiple key-value pairs separated by spaces. Backtick-delimited.

### Common JSON Tags

| Tag | Meaning |
|-----|---------|
| `json:"name"` | JSON field name is "name" |
| `json:"-"` | Skip this field entirely |
| `json:",omitempty"` | Omit if zero value |
| `json:"name,omitempty"` | Both name and omitempty |
| `json:"age,string"` | Encode int as JSON string |

```go
u := User{ID: 1, Name: "Alice", Email: "", Age: 30}
data, _ := json.Marshal(u)
// {"id":1,"name":"Alice","age":"30"}  — email omitted, age as string
```

### Reading Tags via Reflection

```go
t := reflect.TypeOf(User{})
field, _ := t.FieldByName("Email")
fmt.Println(field.Tag.Get("json"))      // "email"
fmt.Println(field.Tag.Get("validate"))  // "required,email"
```

### Struct Tag Rules

- Value must be in double quotes
- Use backticks for the tag string: `` `key:"value"` ``
- Do not use double quotes inside the value
- Only meaningful to packages that read the tag (json, xml, db, etc.)

---

## Methods

A **method** is a function with a **receiver** — a special parameter that binds the method to a type.

### Method Declaration

```go
type User struct {
    Name string
    Age  int
}

func (u User) FullName() string {
    return u.Name
}

func (u User) IsAdult() bool {
    return u.Age >= 18
}
```

- `(u User)` is the **receiver**
- Called with dot notation: `u.FullName()`
- At its core, `u.FullName()` is equivalent to `FullName(u)` — the method form is more readable

### Value Receiver

```go
func (u User) Birthday() {
    u.Age++   // modifies a copy — original unchanged
}
```

### Pointer Receiver

```go
func (u *User) CelebrateBirthday() {
    u.Age++   // modifies the original
}
```

```go
func main() {
    u := User{Name: "Alice", Age: 30}
    u.Birthday()
    fmt.Println(u.Age)          // 30 (unchanged)

    u.CelebrateBirthday()
    fmt.Println(u.Age)          // 31 (changed)
}
```

Go automatically takes the address when you call a pointer method on an addressable value: `u.Rename("Bob")` is shorthand for `(&u).Rename("Bob")`.

> ⚠️ **Gotcha:** A value receiver copies the receiver — mutations inside it vanish when the method returns. Use a pointer receiver when the method must change fields.

### When to Use Which Receiver

| Pointer receiver | Value receiver |
|------------------|----------------|
| Method modifies the receiver | Method only reads receiver |
| Large struct (avoid copy) | Small struct/value type |
| Consistency with other methods | Receiver is slice/map/chan/func |
| Struct contains sync.Mutex | Struct is immutable by design |

> **Rule of thumb:** If any method needs a pointer receiver, use pointer receivers for the type consistently. Mixing receiver types affects which interfaces the type satisfies.

### Receiver Naming Convention

Short, consistent within a type, lowercase abbreviation:

```go
func (m Money) Add(other Money) Money    { ... }
func (s Stack) Peek() (int, bool)        { ... }
func (u User) FullName() string          { ... }
```

Never use `this` or `self` — use a single letter matching the type.

### Methods on Named Types Only

You cannot add methods to built-in types:

> 🔑 **Remember:** Methods can only be declared on named types you define. Wrap `int`, `string`, or `float64` in a named type first.

```go
// INVALID
func (i int) Double() int { return i * 2 }
```

Define a named type first:

```go
type Score int

func (s Score) Double() Score { return s * 2 }
```

This is powerful — a `Celsius` is not a `Fahrenheit` even though both are `float64` underneath.

```go
type Celsius float64
func (c Celsius) ToFahrenheit() float64 { return float64(c)*9/5 + 32 }
```

### Methods on Named Slices

```go
type Scores []int

func (s Scores) Average() float64 {
    total := 0
    for _, v := range s { total += v }
    return float64(total) / float64(len(s))
}

// Pointer receiver — mutates the slice header
func (s *Scores) Append(values ...int) {
    *s = append(*s, values...)
}
```

### Method Values and Method Expressions

Methods are not plain function values. You must bind a receiver:

**Method value** — binds a specific receiver:

```go
c := Counter{n: 5}
value := c.Value      // captures c
fmt.Println(value())  // 5
```

**Method expression** — receiver becomes the first explicit parameter:

```go
valueExpr := Counter.Value
fmt.Println(valueExpr(Counter{n: 10}))   // 10
```

### The Stringer Interface

Any type implementing `String() string` controls its `fmt` output:

```go
func (u User) String() string {
    return fmt.Sprintf("%s (%d years old)", u.Name, u.Age)
}

u := User{Name: "Alice", Age: 30}
fmt.Println(u)   // Alice (30 years old)
```

---

## Method Sets

The **method set** of a type determines which interfaces it satisfies. This is one of the most important concepts in Go:

| Type | Value-receiver methods | Pointer-receiver methods |
|------|------------------------|--------------------------|
| `T` | yes | no |
| `*T` | yes | yes |

**A value receiver method belongs to both `T` and `*T`. A pointer receiver method belongs only to `*T`.**

> 🔑 **Key idea:** The method set decides interface satisfaction: `*T` carries both receiver kinds; `T` carries only value-receiver methods.

```go
type Shape interface {
    Area() float64
}

type Rect struct{ W, H float64 }

func (r Rect) Area() float64 { return r.W * r.H }       // value receiver

// Both Rect and *Rect satisfy Shape
var s1 Shape = Rect{2, 3}    // OK
var s2 Shape = &Rect{2, 3}   // OK
```

But with a pointer receiver:

```go
type Cercle struct{ R float64 }
func (c *Cercle) Area() float64 { return 3.14 * c.R * c.R }

var s Shape = &Cercle{R: 2}   // OK
// var s Shape = Cercle{R: 2} // ERROR: Cercle does not implement Shape
```

> When you store a value in an interface variable, you copy it. The copy is not addressable, so pointer-receiver methods become inaccessible.

### Compile-Time Interface Checks

```go
var _ Shape = Rect{}       // compile-time guarantee
var _ Shape = (*Rect)(nil)
```

These fail to compile if the type doesn't satisfy the interface — an excellent defensive idiom.

---

## Embedding (Composition)

Go has no inheritance. Instead, it uses **struct embedding** — composition with convenient field and method promotion.

> 🧠 **Think of it as:** Embedding is "has-a," not "is-a." `Server` contains a `Logger` and borrows its methods via promotion.

### Basic Embedding

```go
type Address struct {
    Street  string
    City    string
    Country string
}

type Person struct {
    Name    string
    Address   // embedded — fields and methods are promoted
}

func main() {
    p := Person{
        Name:    "Alice",
        Address: Address{Street: "123 Main St", City: "Portland"},
    }

    fmt.Println(p.Name)     // Alice
    fmt.Println(p.City)     // Portland — promoted
    fmt.Println(p.Street)   // 123 Main St — promoted

    // Explicit access also works
    fmt.Println(p.Address.City)   // Portland
}
```

### What Embedding Does

1. **Promotes fields** — access inner fields directly on the outer type
2. **Promotes methods** — inner type's methods become methods of the outer type

```go
type Logger struct{}

func (l Logger) Log(msg string) { fmt.Println(msg) }

type Server struct {
    Logger   // embedded
    Name     string
}

s := Server{Name: "api-server"}
s.Log("starting")   // calls Logger.Log via promotion
```

This is **not** inheritance — `Server` does not "extend" `Logger`. `Server` **contains** a `Logger` and gains its methods through promotion.

### Method Promotion and Interface Satisfaction

If `Logger` implements `io.Writer`, then `Server` also implements `io.Writer` through promotion. This is how Go achieves composition over inheritance — an outer type carries the inner type's behavior.

### Overriding Promoted Methods

```go
func (d Dog) Speak() string {
    return "Woof!"
}

d.Speak()           // "Woof!" — Dog's version
d.Animal.Speak()    // "..." — Animal's version (explicit access)
```

### Multiple Embedding

```go
type Server struct {
    Logger   // from Logger
    Metrics  // from Metrics
    Name     string
}

s := Server{Name: "api-server"}
s.Log("starting")        // from Logger
s.Record("requests", 1)  // from Metrics
```

### Name Conflicts

If two embedded types have the same method, the compiler requires explicit disambiguation:

```go
type A struct{}
func (A) Method() { fmt.Println("A") }

type B struct{}
func (B) Method() { fmt.Println("B") }

type C struct { A; B }

c := C{}
c.Method()     // compile error: ambiguous selector
c.A.Method()   // explicit: OK
c.B.Method()   // explicit: OK
```

### Embedding Interfaces

Interfaces can also be embedded to compose larger ones:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadCloser interface {
    Reader
    Writer
}
```

This is exactly how the standard library composes `io.ReadCloser`, `io.ReadWriter`, etc.

### When to Embed vs Use Named Fields

**Embed when:**

- The embedded type's methods/fields should feel like part of the outer type
- You want to reuse cross-cutting behavior (logging, timestamps, soft-delete)
- The inner type is an implementation detail

**Use a named field when:**

- The inner type should be explicitly referenced (`Items []Item`, `Customer *User`)
- You need to avoid name collisions
- The relationship should be visible in the struct definition

```go
type Order struct {
    Items    []Item     // named — should be explicit
    Customer *User      // named — important to see
    Logger              // embedded — implementation detail
}
```

### Embedding Pitfalls

**Nil embedded pointer:**

```go
type Outer struct {
    *Inner   // nil pointer!
}

o := Outer{}
o.Method()  // panic: nil pointer dereference
```

**Dog is not an Animal:**

```go
func describe(a Animal) { fmt.Println(a.Name) }
d := Dog{Animal: Animal{Name: "Rex"}}
// describe(d)   // compile error: Dog is not Animal
describe(d.Animal)   // works: extract the Animal
```

---

## Constructor Pattern

Go has no constructors. The convention is a function named `NewXxx` that returns a pointer:

```go
type Config struct {
    Host string
    Port int
    TLS  bool
}

func NewConfig(host string, port int) *Config {
    return &Config{
        Host: host,
        Port: port,
        TLS:  true,   // sensible default
    }
}

cfg := NewConfig("localhost", 8080)
```

The `NewXxx` function sets sensible defaults and returns `*Config` so callers share the same instance.

> 💡 **Pro tip:** Follow the `NewXxx()` convention for constructors: build the value with sensible defaults, then return `*T`.

---

## Practical Example: Configuration with Tags

```go
type Config struct {
    Host         string        `json:"host"`
    Port         int           `json:"port"`
    ReadTimeout  time.Duration `json:"read_timeout"`
    WriteTimeout time.Duration `json:"write_timeout"`
    Debug        bool          `json:"debug"`
    SecretKey    string        `json:"-"`
}

func LoadConfig(path string) (*Config, error) {
    cfg := &Config{
        Host:         "localhost",
        Port:         8080,
        ReadTimeout:  30 * time.Second,
        WriteTimeout: 30 * time.Second,
    }
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading config: %w", err)
    }
    if err := json.Unmarshal(data, cfg); err != nil {
        return nil, fmt.Errorf("parsing config: %w", err)
    }
    return cfg, nil
}
```

Pattern: initialize with defaults, override from file, `json:"-"` ensures secrets are never serialized.

---

## Performance Notes

| Operation | Cost |
|-----------|------|
| Struct value copy | O(fields) — copies every field |
| Passing small struct to func | Cheaper than pointer indirection |
| Passing large struct by value | Expensive — prefer pointer |
| Method call (value receiver) | Copies receiver on each call |
| Method call (pointer receiver) | No copy — passes address |
| Embedding promotion | Zero cost — compile-time sugar |
| Interface dispatch | ~2 ns overhead via itab indirection |
| Struct tag reflection | ~100 ns per field — avoid in hot paths |

---

## Modern Practices

- Use embedding for composition — it promotes fields and methods without inheritance
- Use pointer receivers consistently on a type if any method requires mutation
- Use struct tags for JSON, XML, DB, and validation formats
- Prefer `NewX()` constructor functions that return a pointer with sensible defaults
- Keep methods cohesive — group closely related behavior on the same type
- Choose value vs pointer receivers deliberately based on size, mutability, and interface needs
- Define interfaces at the consumer side, not where concrete types live
- Use compile-time interface satisfaction checks (`var _ Interface = &MyType{}`)

## Common Mistakes

- Unexported (lowercase) struct fields are silently omitted during JSON marshaling
- Copying a struct containing a mutex or slice shares the underlying data unexpectedly
- Defining a mutating method on a value receiver — the original struct is never changed
- Confusing embedding with inheritance — Dog is not an Animal, it contains one
- Using `==` to compare structs that contain slices or maps (compile error)
- Ignoring zero values — always check or initialize optional fields before use
- Mixing value and pointer receivers without considering method sets and interface satisfaction
- Receiver name inconsistency across methods of the same type

## Key Takeaways

1. **Structs** group fields; they're value types that copy on assignment.
2. Use **field-name literals** for clarity and safety.
3. **Methods** = functions with a receiver. Value receivers copy; pointer receivers mutate.
4. **Method sets**: value type `T` gets value-receiver methods only; pointer `*T` gets both.
5. **Embedding** = composition with promotion, not inheritance.
6. Struct tags carry metadata (JSON, XML, etc.).
7. No constructors — use the `NewXxx` convention.
8. Always consider method-set implications when choosing receiver types.

## Next

Continue to [02-pointers.md](02-pointers.md).
