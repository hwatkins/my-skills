---
name: rust-core
description: Expert Rust development with ownership, borrowing, lifetimes, traits, error handling, and idiomatic patterns. Use for any Rust code.
---

# Rust Development

You are an expert in Rust development with deep knowledge of systems programming, memory safety, and zero-cost abstractions.

## Core Principles

- Write idiomatic, safe Rust code with clear ownership semantics
- Prefer zero-cost abstractions — don't pay for what you don't use
- Leverage the type system to make invalid states unrepresentable
- Use `clippy` and `rustfmt` conventions consistently
- Favor explicitness over implicitness

## Naming Conventions

- Use `snake_case` for functions, variables, modules, and file names
- Use `PascalCase` for types, traits, and enum variants
- Use `SCREAMING_SNAKE_CASE` for constants and statics
- Prefix unused variables with `_`
- Use descriptive names (e.g., `is_valid`, `parse_config`, `ConnectionPool`)

## Ownership & Borrowing

- Prefer borrowing (`&T`, `&mut T`) over ownership transfer when the caller needs to retain the value
- Use `Clone` sparingly — understand the cost
- Prefer `&str` over `String` in function parameters
- Prefer `&[T]` over `Vec<T>` in function parameters
- Return owned types (`String`, `Vec<T>`) from functions that create new data
- Use lifetimes explicitly only when the compiler can't infer them

```rust
// ✅ Good: Borrow when you don't need ownership
fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}

// ✅ Good: Take ownership when you need to store it
fn add_user(users: &mut Vec<String>, name: String) {
    users.push(name);
}

// ❌ Bad: Unnecessary clone
fn greet(name: String) -> String {
    format!("Hello, {name}!")
}
```

## Error Handling

- Use `Result<T, E>` for recoverable errors, `panic!` only for unrecoverable bugs
- Define custom error types with `thiserror` for libraries, `anyhow` for applications
- Use the `?` operator for error propagation
- Never use `.unwrap()` in production code — use `.expect("reason")` at minimum
- Prefer `if let` / `match` over `.unwrap()` for `Option` and `Result`

```rust
// ✅ Good: Custom error with thiserror
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("not found: {0}")]
    NotFound(String),
    #[error("database error")]
    Database(#[from] sqlx::Error),
    #[error("validation failed: {0}")]
    Validation(String),
}

// ✅ Good: Propagate with ?
fn load_config(path: &Path) -> Result<Config, AppError> {
    let contents = fs::read_to_string(path)?;
    let config: Config = toml::from_str(&contents)?;
    Ok(config)
}

// ❌ Bad: Unwrap in production
let config = fs::read_to_string("config.toml").unwrap();
```

## Structs & Enums

- Use structs for data, enums for variants
- Derive common traits: `Debug`, `Clone`, `PartialEq` as needed
- Use builder pattern or `Default` for complex construction
- Prefer newtype pattern for type safety (e.g., `struct UserId(u64)`)

```rust
// ✅ Good: Newtype for type safety
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(pub u64);

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Email(String);

impl Email {
    pub fn new(value: &str) -> Result<Self, ValidationError> {
        if value.contains('@') {
            Ok(Self(value.to_lowercase()))
        } else {
            Err(ValidationError::InvalidEmail)
        }
    }
}
```

## Traits

- Use traits to define shared behavior
- Prefer trait bounds over trait objects when possible (static dispatch)
- Use `impl Trait` in return position for zero-cost abstraction
- Use `dyn Trait` only when you need dynamic dispatch

```rust
// ✅ Good: Static dispatch with generics
fn process<T: Serialize + Send>(item: T) -> Result<(), Error> {
    // ...
}

// ✅ Good: impl Trait for return types
fn create_handler() -> impl Fn(Request) -> Response {
    |req| Response::ok(req.body())
}

// Use dyn Trait when variants are determined at runtime
fn get_storage(config: &Config) -> Box<dyn Storage> {
    match config.storage_type {
        StorageType::S3 => Box::new(S3Storage::new()),
        StorageType::Local => Box::new(LocalStorage::new()),
    }
}
```

## Pattern Matching

- Use exhaustive matching — avoid wildcard `_` catch-alls when possible
- Destructure in function parameters and `let` bindings
- Use `matches!` macro for simple boolean checks
- Use `if let` for single-variant matching

```rust
// ✅ Good: Exhaustive matching
match status {
    Status::Active => activate(user),
    Status::Inactive => deactivate(user),
    Status::Suspended { reason, until } => suspend(user, reason, until),
}

// ✅ Good: if let for single variant
if let Some(user) = find_user(id) {
    greet(&user);
}

// ✅ Good: matches! for boolean checks
if matches!(status, Status::Active | Status::Pending) {
    // ...
}
```

## Iterators & Functional Patterns

- Prefer iterator chains over manual loops
- Use `collect()` with type annotation to drive collection type
- Use `map`, `filter`, `filter_map`, `flat_map` for transformations
- Use `fold` / `reduce` for aggregation

```rust
// ✅ Good: Iterator chain
let active_emails: Vec<String> = users
    .iter()
    .filter(|u| u.is_active())
    .map(|u| u.email.clone())
    .collect();

// ✅ Good: filter_map for combined filter + transform
let valid_ids: Vec<u64> = input
    .iter()
    .filter_map(|s| s.parse::<u64>().ok())
    .collect();
```

## Modules & Project Structure

- One module per file, match file structure to module hierarchy
- Use `pub` sparingly — expose minimal public API
- Re-export important types from parent modules
- Use `mod.rs` or `module_name.rs` (prefer the latter for flat structure)

```
src/
├── main.rs          # or lib.rs
├── config.rs
├── error.rs
├── db/
│   ├── mod.rs
│   ├── connection.rs
│   └── queries.rs
└── handlers/
    ├── mod.rs
    ├── users.rs
    └── tasks.rs
```

## Performance

- Profile before optimizing — use `cargo flamegraph`, `criterion` for benchmarks
- Prefer stack allocation over heap when size is known
- Use `Cow<'_, str>` when a function may or may not need to allocate
- Avoid unnecessary allocations in hot paths
- Use `Vec::with_capacity` when the size is known ahead of time

## Unsafe & FFI

- Minimize `unsafe` — use only when safe abstractions can't express the invariant
- Every `unsafe` block **must** have a `// SAFETY:` comment explaining why it's sound
- Encapsulate `unsafe` behind safe public APIs — callers should never need `unsafe`
- Prefer existing safe abstractions (`Vec`, `Box`, `Arc`) over raw pointers

```rust
// ✅ Good: Safe wrapper around unsafe
pub fn split_at_unchecked(s: &str, mid: usize) -> (&str, &str) {
    assert!(s.is_char_boundary(mid), "not a char boundary");
    // SAFETY: We verified mid is a valid char boundary above
    unsafe { (s.get_unchecked(..mid), s.get_unchecked(mid..)) }
}

// ❌ Bad: Unsafe without justification or bounds checking
pub fn bad_split(s: &str, mid: usize) -> (&str, &str) {
    unsafe { (s.get_unchecked(..mid), s.get_unchecked(mid..)) }
}
```

**FFI basics:**

```rust
// Calling C from Rust
extern "C" {
    fn strlen(s: *const std::ffi::c_char) -> usize;
}

fn safe_strlen(s: &std::ffi::CStr) -> usize {
    // SAFETY: CStr guarantees null-terminated, valid pointer
    unsafe { strlen(s.as_ptr()) }
}

// Exposing Rust to C
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

- Use `CStr`/`CString` for C string interop (never raw `*const u8`)
- Use `bindgen` to auto-generate FFI bindings from C headers
- Run `cargo miri test` to detect undefined behavior in unsafe code
- Audit all `unsafe` during code review — treat it as a red flag to verify

## Common Crates

| Purpose | Crate |
|---------|-------|
| Serialization | `serde`, `serde_json`, `toml` |
| Error handling | `thiserror` (lib), `anyhow` (app) |
| HTTP client | `reqwest` |
| Web framework | `axum`, `actix-web` |
| Database | `sqlx`, `diesel` |
| Async runtime | `tokio` |
| CLI | `clap` |
| Logging | `tracing`, `tracing-subscriber` |
| Testing | `proptest`, `rstest`, `mockall` |

## Code Organization

- Keep functions short and focused
- Separate pure logic from I/O (functional core, imperative shell)
- Use `impl` blocks to group related methods
- Place tests in the same file with `#[cfg(test)]` module
