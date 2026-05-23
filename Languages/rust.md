# Rust Cheatsheet

## Mental Model

Rust is a systems language built around **ownership**: every value has exactly one owner, and when the owner goes out of scope, the value is dropped (freed). This eliminates garbage collection and data races at compile time. The borrow checker enforces two rules: you can have either *one mutable reference* or *any number of immutable references* — never both simultaneously. If your code compiles, it's memory safe.

---

## Install & Minimal Setup

```bash
# Install via rustup (manages toolchains and targets)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Verify
rustc --version
cargo --version

# New project
cargo new my-project          # binary (src/main.rs)
cargo new my-lib --lib        # library (src/lib.rs)
cd my-project

# Build & run
cargo run                     # build + execute
cargo build                   # debug build → target/debug/
cargo build --release         # optimized → target/release/
cargo check                   # type-check without building (fast)
cargo test                    # run all tests
```

---

## Core Concepts

### 1. Ownership

```rust
// Each value has exactly one owner
let s1 = String::from("hello");
let s2 = s1;          // s1 is MOVED — ownership transferred to s2
// println!("{}", s1); // ❌ compile error: s1 is invalid

// Clone to get an independent copy
let s1 = String::from("hello");
let s2 = s1.clone();  // deep copy — both are valid
println!("{} {}", s1, s2); // ✅

// Stack types (Copy trait) — implicitly copied, no move
let x = 5;
let y = x;
println!("{} {}", x, y); // ✅ — integers, floats, bools, char, tuples of Copy types
```

### 2. Borrowing & References

```rust
fn print_length(s: &String) {     // borrow — does not take ownership
    println!("Length: {}", s.len());
}

let s = String::from("hello");
print_length(&s);                  // pass a reference
println!("{}", s);                 // s is still valid here

// Mutable reference — only ONE at a time
let mut s = String::from("hello");
let r = &mut s;
r.push_str(", world");
// Can't use s again here until r goes out of scope
```

### 3. Slices

```rust
let s = String::from("hello world");
let hello = &s[0..5];   // &str slice — a view into the String
let world = &s[6..11];

let arr = [1, 2, 3, 4, 5];
let slice = &arr[1..3]; // [2, 3] — type is &[i32]
```

### 4. Structs & Impl

```rust
#[derive(Debug, Clone)]
struct User {
    name: String,
    email: String,
    active: bool,
}

impl User {
    // Associated function (constructor pattern)
    fn new(name: &str, email: &str) -> Self {
        User {
            name: name.to_string(),
            email: email.to_string(),
            active: true,
        }
    }

    // Method — &self borrows, &mut self mutates, self consumes
    fn deactivate(&mut self) {
        self.active = false;
    }
}

let mut user = User::new("Joshua", "j@example.com");
user.deactivate();
println!("{:?}", user);
```

### 5. Enums & Pattern Matching

```rust
#[derive(Debug)]
enum Shape {
    Circle(f64),              // tuple variant
    Rectangle(f64, f64),
    Triangle { base: f64, height: f64 }, // struct variant
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle(r)               => std::f64::consts::PI * r * r,
        Shape::Rectangle(w, h)         => w * h,
        Shape::Triangle { base, height } => 0.5 * base * height,
    }
}

// match is exhaustive — compiler forces you to handle all variants
```

### 6. Option & Result (No null, no exceptions)

```rust
// Option<T> — value may or may not exist
fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("Joshua".to_string()) } else { None }
}

match find_user(1) {
    Some(name) => println!("Found: {}", name),
    None       => println!("Not found"),
}

// Shorthand
let name = find_user(1).unwrap_or("Unknown".to_string());
let name = find_user(1)?;   // propagate None upward (in fn returning Option)

// Result<T, E> — operation may fail
fn parse_number(s: &str) -> Result<i32, std::num::ParseIntError> {
    s.trim().parse::<i32>()
}

match parse_number("42") {
    Ok(n)    => println!("Parsed: {}", n),
    Err(e)   => println!("Error: {}", e),
}

// ? operator — propagate errors upward (in fn returning Result)
fn process(s: &str) -> Result<i32, Box<dyn std::error::Error>> {
    let n = parse_number(s)?;   // returns Err early if parse fails
    Ok(n * 2)
}
```

### 7. Traits (Interfaces)

```rust
trait Summarize {
    fn summary(&self) -> String;

    // Default implementation
    fn short_summary(&self) -> String {
        format!("{}...", &self.summary()[..50])
    }
}

struct Article {
    title: String,
    body: String,
}

impl Summarize for Article {
    fn summary(&self) -> String {
        format!("{}: {}", self.title, self.body)
    }
}

// Trait bounds — accept any type that implements Summarize
fn print_summary(item: &impl Summarize) {
    println!("{}", item.summary());
}

// Generic version
fn print_summary<T: Summarize>(item: &T) {
    println!("{}", item.summary());
}
```

### 8. Iterators & Closures

```rust
let numbers = vec![1, 2, 3, 4, 5];

// Closures — inline anonymous functions
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
let evens: Vec<&i32>  = numbers.iter().filter(|x| *x % 2 == 0).collect();
let sum: i32          = numbers.iter().sum();
let product: i32      = numbers.iter().fold(1, |acc, x| acc * x);

// Chaining is lazy — nothing runs until collect() or similar
let result: Vec<i32> = (1..=100)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .take(5)
    .collect();  // [4, 16, 36, 64, 100]
```

### 9. Error Handling with `thiserror` / `anyhow`

```rust
// thiserror — for library errors (defines your own error types)
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Not found: {id}")]
    NotFound { id: u32 },
}

// anyhow — for application code (simple dynamic errors)
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<String> {
    std::fs::read_to_string(path)
        .with_context(|| format!("Failed to read config from {}", path))
}
```

### 10. Async / Await

```rust
use tokio;

#[tokio::main]
async fn main() {
    let result = fetch_data().await;
    println!("{:?}", result);
}

async fn fetch_data() -> Result<String, reqwest::Error> {
    let body = reqwest::get("https://api.example.com/data")
        .await?
        .text()
        .await?;
    Ok(body)
}

// Run concurrently
let (a, b) = tokio::join!(task_a(), task_b());
```

---

## Most-Used Patterns

### Cargo.toml Dependencies

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
anyhow = "1"
thiserror = "1"
```

### Serde — Serialize / Deserialize

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    name: String,
    age: u32,
}

// Serialize to JSON string
let user = User { name: "Joshua".to_string(), age: 28 };
let json = serde_json::to_string(&user).unwrap();

// Deserialize from JSON string
let parsed: User = serde_json::from_str(&json).unwrap();
```

### Vec Operations

```rust
let mut v: Vec<i32> = Vec::new();
let mut v = vec![1, 2, 3];

v.push(4);
v.pop();              // Option<i32>
v.len();
v.is_empty();
v.contains(&3);
v.sort();
v.dedup();            // remove consecutive duplicates (sort first)
v.retain(|x| *x > 2); // keep only elements where condition is true
```

### HashMap

```rust
use std::collections::HashMap;

let mut map: HashMap<String, i32> = HashMap::new();
map.insert("key".to_string(), 1);
map.get("key");                        // Option<&i32>
map.entry("key".to_string()).or_insert(0);  // insert if not present
map.contains_key("key");
map.remove("key");

for (k, v) in &map {
    println!("{}: {}", k, v);
}
```

---

## Gotchas

- **Move vs Copy** — `String`, `Vec`, custom structs are moved. Primitive types (`i32`, `bool`, `f64`) are copied. When in doubt, derive `Clone` and call `.clone()`.
- **`&str` vs `String`** — `&str` is a borrowed string slice (usually a literal or slice of a `String`). `String` is owned, heap-allocated. Functions that don't need ownership should accept `&str`.
- **Lifetime annotations** — the compiler infers lifetimes most of the time. You need explicit annotations when returning references from functions that have multiple input references.
- **`unwrap()` in production** — panics on `None`/`Err`. Use `?`, `unwrap_or`, `map`, or `match` instead.
- **The borrow checker rejects valid-feeling code** — if you're fighting it, step back and reconsider ownership. Usually you need to clone, restructure the code, or return an owned value instead of a reference.
- **`iter()` vs `into_iter()`** — `iter()` borrows elements (`&T`), `into_iter()` consumes the collection (gives `T`), `iter_mut()` gives `&mut T`.

---

## Quick Links

- [The Rust Book](https://doc.rust-lang.org/book/) — the definitive resource; chapters 4–6 on ownership are essential
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Rustlings](https://github.com/rust-lang/rustlings) — interactive exercises
- [crates.io](https://crates.io) — package registry
- [docs.rs](https://docs.rs) — auto-generated crate documentation
- [Tokio](https://tokio.rs) — async runtime for Rust
