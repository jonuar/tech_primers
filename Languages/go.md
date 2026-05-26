# Go (Golang) Cheatsheet

## Mental Model

Go is a statically typed, compiled language designed for **simplicity and concurrency**. It has no classes (only structs + methods), no inheritance (only interfaces), no exceptions (only explicit error returns), and no generics complexity (simple type parameters since 1.18). The concurrency model is built around **goroutines** (lightweight threads) and **channels** (typed message pipes). If it compiles, it's likely correct — Go's compiler is strict and opinionated.

---

## Install & Minimal Setup

```bash
# Install via official installer at https://go.dev/dl/
# or
brew install go          # macOS

# Verify
go version

# New module
mkdir my-project && cd my-project
go mod init github.com/username/my-project   # creates go.mod

# Run / Build
go run main.go
go build -o my-app ./...
go test ./...

# Add a dependency
go get github.com/gin-gonic/gin@latest
go mod tidy    # clean up unused deps
```

```go
// main.go — minimal program
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

---

## Core Concepts

### 1. Variables & Types

```go
// Declaration
var name string = "Joshua"
var age int = 28

// Short declaration (inside functions only)
name := "Joshua"
age := 28

// Multiple
x, y := 10, 20

// Zero values — Go initializes everything
var i int       // 0
var f float64   // 0.0
var b bool      // false
var s string    // ""

// Constants
const Pi = 3.14159
const (
    StatusOK   = 200
    StatusNotFound = 404
)

// Basic types
int, int8, int16, int32, int64
uint, uint8, uint16, uint32, uint64
float32, float64
complex64, complex128
bool
string
byte    // alias for uint8
rune    // alias for int32 (unicode code point)
```

### 2. Control Flow

```go
// if — no parentheses, braces required
if x > 10 {
    fmt.Println("big")
} else if x > 5 {
    fmt.Println("medium")
} else {
    fmt.Println("small")
}

// if with init statement (scoped to the if block)
if val, err := doSomething(); err != nil {
    log.Fatal(err)
} else {
    fmt.Println(val)
}

// for — the only loop in Go
for i := 0; i < 10; i++ { }         // C-style
for i < 10 { }                       // while-style
for { }                              // infinite loop (break to exit)

// range — iterate over slices, maps, channels, strings
for i, v := range []int{1, 2, 3} {
    fmt.Println(i, v)
}
for k, v := range myMap { }
for _, v := range mySlice { }        // discard index
for i := range mySlice { }           // index only

// switch — no fallthrough by default, no break needed
switch status {
case 200:
    fmt.Println("OK")
case 404:
    fmt.Println("Not found")
default:
    fmt.Println("Unknown")
}

// switch with no condition (cleaner than if-else chains)
switch {
case score >= 90: grade = "A"
case score >= 80: grade = "B"
default:          grade = "C"
}
```

### 3. Functions

```go
// Basic function
func add(a, b int) int {
    return a + b
}

// Multiple return values — idiomatic Go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 2)
if err != nil {
    log.Fatal(err)
}

// Named return values
func minMax(nums []int) (min, max int) {
    min, max = nums[0], nums[0]
    for _, n := range nums[1:] {
        if n < min { min = n }
        if n > max { max = n }
    }
    return  // naked return — returns named values
}

// Variadic
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
sum(1, 2, 3)
sum(nums...)  // unpack a slice

// First-class functions
apply := func(f func(int) int, x int) int {
    return f(x)
}
double := func(x int) int { return x * 2 }
apply(double, 5)  // 10

// defer — runs at function exit, LIFO order
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil { return err }
    defer f.Close()   // guaranteed to run even if panic occurs
    // ... use f
    return nil
}
```

### 4. Pointers

```go
x := 42
p := &x          // p is a *int (pointer to x)
fmt.Println(*p)  // dereference → 42
*p = 100         // modify through pointer → x is now 100

// Structs are often passed by pointer to avoid copying
func (u *User) Activate() {
    u.Active = true   // modifies the original
}
```

### 5. Structs & Methods

```go
type User struct {
    ID     int
    Name   string
    Email  string
    Active bool
}

// Constructor pattern (Go has no constructors)
func NewUser(name, email string) *User {
    return &User{
        Name:   name,
        Email:  email,
        Active: true,
    }
}

// Value receiver — doesn't modify the original
func (u User) String() string {
    return fmt.Sprintf("%s <%s>", u.Name, u.Email)
}

// Pointer receiver — modifies the original
func (u *User) Deactivate() {
    u.Active = false
}

u := NewUser("Joshua", "j@example.com")
u.Deactivate()
fmt.Println(u)  // calls String()
```

### 6. Interfaces

```go
// Interface — implicit satisfaction, no "implements" keyword
type Writer interface {
    Write(data []byte) (int, error)
}

type Logger interface {
    Log(msg string)
    LogError(err error)
}

// Struct satisfies Logger implicitly by having both methods
type ConsoleLogger struct{}

func (c ConsoleLogger) Log(msg string)      { fmt.Println(msg) }
func (c ConsoleLogger) LogError(err error)  { fmt.Fprintln(os.Stderr, err) }

func Process(logger Logger, data string) {
    logger.Log("Processing: " + data)
}

// Empty interface — holds any value (avoid when possible)
var anything interface{} = 42
anything = "now a string"

// Type assertion
val, ok := anything.(string)
if ok {
    fmt.Println("string:", val)
}

// Type switch
switch v := anything.(type) {
case int:    fmt.Println("int:", v)
case string: fmt.Println("string:", v)
default:     fmt.Println("unknown type")
}
```

### 7. Slices & Maps

```go
// Slices — dynamic arrays
s := []int{1, 2, 3}
s = append(s, 4, 5)
s2 := s[1:3]              // slice [2, 3] — shares memory with s
s3 := make([]int, 0, 10)  // length 0, capacity 10

// Maps
m := map[string]int{
    "a": 1,
    "b": 2,
}
m["c"] = 3

val, ok := m["a"]   // ok = false if key doesn't exist
if !ok {
    fmt.Println("key not found")
}

delete(m, "a")
len(m)

// Iterate
for k, v := range m {
    fmt.Println(k, v)
}
```

### 8. Error Handling

```go
// Custom error type
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}

// errors package — wrapping and unwrapping
import "errors"

var ErrNotFound = errors.New("not found")

func getUser(id int) (*User, error) {
    if id <= 0 {
        return nil, fmt.Errorf("getUser: invalid id %d: %w", id, ErrNotFound)
    }
    return &User{ID: id}, nil
}

// Check error type
_, err := getUser(-1)
if errors.Is(err, ErrNotFound) {
    fmt.Println("user not found")
}

var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Println("field:", ve.Field)
}
```

### 9. Goroutines & Channels

```go
// Goroutine — launch with 'go' keyword
go func() {
    fmt.Println("running concurrently")
}()

// Channel — typed communication pipe
ch := make(chan int)         // unbuffered — blocks until both sides ready
ch := make(chan int, 10)     // buffered — blocks only when full

// Send and receive
go func() { ch <- 42 }()   // send (goroutine)
val := <-ch                 // receive (blocks until value arrives)

// Close and range over channel
close(ch)
for v := range ch {         // reads until channel closed
    fmt.Println(v)
}

// select — wait on multiple channels
select {
case v := <-ch1:
    fmt.Println("from ch1:", v)
case v := <-ch2:
    fmt.Println("from ch2:", v)
case <-time.After(1 * time.Second):
    fmt.Println("timeout")
}
```

### 10. sync & Context

```go
import "sync"

// WaitGroup — wait for goroutines to finish
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println("worker", id)
    }(i)
}
wg.Wait()

// Mutex — protect shared state
var mu sync.Mutex
mu.Lock()
counter++
mu.Unlock()

// Context — cancellation and deadlines
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

select {
case result := <-doWork(ctx):
    fmt.Println(result)
case <-ctx.Done():
    fmt.Println("timed out:", ctx.Err())
}
```

---

## Most-Used Patterns

### HTTP Server with net/http

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Response struct {
    Message string `json:"message"`
}

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(Response{Message: "ok"})
    })

    http.ListenAndServe(":8080", mux)
}
```

### HTTP with Gin

```go
import "github.com/gin-gonic/gin"

r := gin.Default()

r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(200, gin.H{"id": id})
})

r.POST("/users", func(c *gin.Context) {
    var body UserCreate
    if err := c.ShouldBindJSON(&body); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    c.JSON(201, gin.H{"name": body.Name})
})

r.Run(":8080")
```

### Generics (Go 1.18+)

```go
func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

doubled := Map([]int{1, 2, 3}, func(x int) int { return x * 2 })
```

---

## Gotchas

- **`:=` only inside functions** — use `var` at package level.
- **Unused variables are compile errors** — Go will not compile if you declare a variable and never use it. Use `_` to discard.
- **Unused imports are compile errors** — same rule for imports.
- **Goroutine leaks** — always provide a way for goroutines to stop (context cancellation, closing channels). A goroutine with no exit path leaks forever.
- **Map iteration order is random** — never rely on map iteration order. Sort keys explicitly if order matters.
- **nil slice vs empty slice** — `var s []int` is nil; `s := []int{}` is empty but not nil. Both have length 0, but `json.Marshal` produces `null` vs `[]`.
- **Value vs pointer receivers** — be consistent within a type. If any method needs a pointer receiver (to mutate), make all methods pointer receivers.
- **Closing a closed channel panics** — use a `sync.Once` or careful design to ensure only one goroutine closes a channel.

---

## Quick Links

- [Go Tour](https://go.dev/tour/) — interactive intro, start here
- [Effective Go](https://go.dev/doc/effective_go) — idiomatic patterns
- [Go by Example](https://gobyexample.com) — concise pattern reference
- [pkg.go.dev](https://pkg.go.dev) — package documentation
- [Gin Framework](https://gin-gonic.com/docs/)
- [awesome-go](https://github.com/avelino/awesome-go) — curated library list
