---
name: rust-patterns
description: Idiomatic Rust patterns, best practices, and conventions for building safe, efficient, and maintainable Rust applications.
---

# Rust Development Patterns

Idiomatic Rust patterns and best practices for building safe, efficient, and maintainable applications.

## When to Activate

- Writing new Rust code
- Reviewing Rust code
- Refactoring existing Rust code
- Designing Rust crates/modules

## Core Principles

### 1. Ownership and Borrowing

Rust's ownership system is its defining feature. Understand and leverage it.

```rust
// Good: Borrow when you don't need ownership
fn word_count(text: &str) -> usize {
    text.split_whitespace().count()
}

// Good: Take ownership when you need to store the value
fn set_name(&mut self, name: String) {
    self.name = name;
}

// Bad: Unnecessary clone
fn process(data: &str) {
    let owned = data.to_string(); // Don't clone unless you need ownership
    println!("{owned}");
}

// Good: Use Cow for conditional ownership
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_"))
    } else {
        Cow::Borrowed(input)
    }
}
```

### 2. Make Invalid States Unrepresentable

Use the type system to enforce correctness at compile time.

```rust
// Bad: Runtime validation needed
struct Order {
    status: String, // "pending", "shipped", "delivered" - what if typo?
    tracking_number: Option<String>,
}

// Good: Compiler enforces valid states
enum OrderStatus {
    Pending,
    Shipped { tracking_number: String },
    Delivered { tracking_number: String, delivered_at: DateTime<Utc> },
}

struct Order {
    id: OrderId,
    status: OrderStatus,
}
```

### 3. Parse, Don't Validate

Convert unstructured data into structured types at the boundary.

```rust
// Bad: Validate and hope it stays valid
fn process_email(email: &str) -> Result<(), Error> {
    if !email.contains('@') {
        return Err(Error::InvalidEmail);
    }
    // email could be passed around and misused elsewhere
    send_email(email)
}

// Good: Parse into a validated type
#[derive(Debug, Clone)]
pub struct Email(String);

impl Email {
    pub fn parse(value: &str) -> Result<Self, ValidationError> {
        if !value.contains('@') || value.len() < 3 {
            return Err(ValidationError::InvalidEmail(value.to_string()));
        }
        Ok(Self(value.to_lowercase()))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

fn send_email(to: &Email) {
    // Guaranteed valid — no runtime check needed
}
```

### 4. Leverage Zero-Cost Abstractions

Rust's iterators, generics, and traits compile to code as efficient as hand-written loops.

```rust
// Good: Iterator chains are zero-cost
let total: f64 = orders.iter()
    .filter(|o| o.is_active())
    .map(|o| o.total())
    .sum();

// Equivalent to manual loop but more readable and composable
```

## Error Handling

### Library Errors with thiserror

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DatabaseError {
    #[error("connection failed: {0}")]
    ConnectionFailed(String),

    #[error("query failed: {query}")]
    QueryFailed {
        query: String,
        #[source]
        source: sqlx::Error,
    },

    #[error("record not found: {0}")]
    NotFound(String),

    #[error(transparent)]
    Internal(#[from] sqlx::Error),
}
```

### Application Errors with anyhow

```rust
use anyhow::{bail, Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {path}"))?;

    let config: Config = toml::from_str(&content)
        .context("failed to parse config TOML")?;

    if config.port == 0 {
        bail!("port must be non-zero");
    }

    Ok(config)
}
```

### Error Conversion Pattern

```rust
// Convert between error types cleanly
impl From<DatabaseError> for ApiError {
    fn from(err: DatabaseError) -> Self {
        match err {
            DatabaseError::NotFound(msg) => ApiError::NotFound(msg),
            other => ApiError::Internal(other.to_string()),
        }
    }
}
```

## Struct and Enum Patterns

### Builder Pattern

```rust
#[derive(Debug)]
pub struct Request {
    url: String,
    method: Method,
    headers: HashMap<String, String>,
    timeout: Duration,
}

#[derive(Debug, Default)]
pub struct RequestBuilder {
    url: Option<String>,
    method: Method,
    headers: HashMap<String, String>,
    timeout: Duration,
}

impl RequestBuilder {
    pub fn new(url: impl Into<String>) -> Self {
        Self {
            url: Some(url.into()),
            timeout: Duration::from_secs(30),
            ..Default::default()
        }
    }

    pub fn method(mut self, method: Method) -> Self {
        self.method = method;
        self
    }

    pub fn header(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.headers.insert(key.into(), value.into());
        self
    }

    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = timeout;
        self
    }

    pub fn build(self) -> Result<Request, BuildError> {
        let url = self.url.ok_or(BuildError::MissingUrl)?;
        Ok(Request {
            url,
            method: self.method,
            headers: self.headers,
            timeout: self.timeout,
        })
    }
}
```

### NewType Pattern

```rust
/// A validated user ID.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct UserId(String);

impl UserId {
    pub fn new(id: impl Into<String>) -> Result<Self, ValidationError> {
        let id = id.into();
        if id.is_empty() {
            return Err(ValidationError::EmptyId);
        }
        Ok(Self(id))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl std::fmt::Display for UserId {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}
```

### Enum State Machine

```rust
enum Connection {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session: Session },
    Error { reason: String, retries: u32 },
}

impl Connection {
    fn connect(self) -> Self {
        match self {
            Connection::Disconnected => Connection::Connecting { attempt: 1 },
            Connection::Error { retries, .. } => Connection::Connecting { attempt: retries + 1 },
            other => other, // Already connecting or connected
        }
    }

    fn on_success(self, session: Session) -> Self {
        match self {
            Connection::Connecting { .. } => Connection::Connected { session },
            other => other,
        }
    }

    fn on_failure(self, reason: String) -> Self {
        match self {
            Connection::Connecting { attempt } => Connection::Error {
                reason,
                retries: attempt,
            },
            other => other,
        }
    }
}
```

## Trait Patterns

### Trait as Interface

```rust
pub trait Repository: Send + Sync {
    type Error: std::error::Error + Send + Sync + 'static;

    fn find_by_id(&self, id: &str) -> Result<Option<Entity>, Self::Error>;
    fn save(&self, entity: &Entity) -> Result<(), Self::Error>;
    fn delete(&self, id: &str) -> Result<bool, Self::Error>;
}

// Implement for production
pub struct PostgresRepository {
    pool: PgPool,
}

impl Repository for PostgresRepository {
    type Error = sqlx::Error;

    fn find_by_id(&self, id: &str) -> Result<Option<Entity>, Self::Error> {
        // actual database query
        todo!()
    }

    fn save(&self, entity: &Entity) -> Result<(), Self::Error> {
        todo!()
    }

    fn delete(&self, id: &str) -> Result<bool, Self::Error> {
        todo!()
    }
}

// Implement for tests
#[cfg(test)]
pub struct MockRepository {
    entities: std::sync::Mutex<HashMap<String, Entity>>,
}
```

### Extension Traits

```rust
pub trait StringExt {
    fn truncate_to(&self, max_len: usize) -> &str;
    fn is_blank(&self) -> bool;
}

impl StringExt for str {
    fn truncate_to(&self, max_len: usize) -> &str {
        if self.len() <= max_len {
            self
        } else {
            &self[..self.floor_char_boundary(max_len)]
        }
    }

    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
}
```

### Default Implementations

```rust
pub trait Handler {
    fn handle(&self, request: Request) -> Response;

    // Default implementation — can be overridden
    fn health_check(&self) -> Response {
        Response::ok("healthy")
    }

    fn name(&self) -> &str {
        std::any::type_name::<Self>()
    }
}
```

## Concurrency Patterns

### Shared State with Arc<Mutex<T>>

```rust
use std::sync::{Arc, Mutex};

#[derive(Clone)]
pub struct AppState {
    cache: Arc<Mutex<HashMap<String, CacheEntry>>>,
}

impl AppState {
    pub fn new() -> Self {
        Self {
            cache: Arc::new(Mutex::new(HashMap::new())),
        }
    }

    pub fn get(&self, key: &str) -> Option<CacheEntry> {
        let cache = self.cache.lock().unwrap();
        cache.get(key).cloned()
    }

    pub fn set(&self, key: String, value: CacheEntry) {
        let mut cache = self.cache.lock().unwrap();
        cache.insert(key, value);
    }
}
```

### Prefer RwLock for Read-Heavy Workloads

```rust
use std::sync::{Arc, RwLock};

pub struct ConfigStore {
    config: Arc<RwLock<Config>>,
}

impl ConfigStore {
    pub fn get(&self) -> Config {
        self.config.read().unwrap().clone()
    }

    pub fn update(&self, new_config: Config) {
        let mut config = self.config.write().unwrap();
        *config = new_config;
    }
}
```

### Channel-Based Communication

```rust
use tokio::sync::mpsc;

async fn producer(tx: mpsc::Sender<Event>) {
    loop {
        let event = poll_events().await;
        if tx.send(event).await.is_err() {
            break; // Receiver dropped
        }
    }
}

async fn consumer(mut rx: mpsc::Receiver<Event>) {
    while let Some(event) = rx.recv().await {
        process_event(event).await;
    }
}

// Wire them together
let (tx, rx) = mpsc::channel(100); // Bounded channel with backpressure
tokio::spawn(producer(tx));
tokio::spawn(consumer(rx));
```

### Async Patterns with Tokio

```rust
use tokio::task::JoinSet;

async fn fetch_all(urls: Vec<String>) -> Vec<Result<Response, Error>> {
    let mut set = JoinSet::new();

    for url in urls {
        set.spawn(async move {
            reqwest::get(&url).await.map_err(Error::from)
        });
    }

    let mut results = Vec::new();
    while let Some(result) = set.join_next().await {
        results.push(result.unwrap());
    }
    results
}

// Graceful shutdown
async fn run_server(shutdown: tokio::sync::oneshot::Receiver<()>) {
    let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();

    loop {
        tokio::select! {
            Ok((stream, _)) = listener.accept() => {
                tokio::spawn(handle_connection(stream));
            }
            _ = shutdown => {
                tracing::info!("shutting down gracefully");
                break;
            }
        }
    }
}
```

## Module Organization

### Recommended Crate Structure

```
my_crate/
├── Cargo.toml
├── src/
│   ├── lib.rs          # Public API, re-exports
│   ├── main.rs         # Binary entry point (if applicable)
│   ├── config.rs       # Configuration types
│   ├── error.rs        # Error types
│   ├── domain/         # Domain models
│   │   ├── mod.rs
│   │   ├── user.rs
│   │   └── order.rs
│   ├── repository/     # Data access
│   │   ├── mod.rs
│   │   ├── traits.rs
│   │   └── postgres.rs
│   ├── service/        # Business logic
│   │   ├── mod.rs
│   │   └── user_service.rs
│   └── handler/        # HTTP handlers / CLI
│       ├── mod.rs
│       └── user_handler.rs
├── tests/              # Integration tests
│   └── api_tests.rs
└── benches/            # Benchmarks
    └── throughput.rs
```

### Module Re-exports

```rust
// src/lib.rs - Clean public API
pub mod config;
pub mod error;
pub mod domain;
pub mod service;

// Re-export commonly used types
pub use config::Config;
pub use error::AppError;
pub use domain::{User, Order};
```

## Iterator Patterns

### Chaining and Collecting

```rust
// Filter, transform, and collect
let active_names: Vec<String> = users.iter()
    .filter(|u| u.is_active)
    .map(|u| u.name.clone())
    .collect();

// Partition into two groups
let (adults, minors): (Vec<_>, Vec<_>) = users.iter()
    .partition(|u| u.age >= 18);

// Flatten nested structures
let all_orders: Vec<&Order> = customers.iter()
    .flat_map(|c| &c.orders)
    .collect();

// Fold/reduce
let total: f64 = items.iter()
    .fold(0.0, |acc, item| acc + item.price * item.quantity as f64);

// Find with condition
let admin = users.iter()
    .find(|u| u.role == Role::Admin);

// Check conditions
let all_valid = items.iter().all(|i| i.quantity > 0);
let any_expired = items.iter().any(|i| i.is_expired());
```

### Custom Iterator

```rust
struct Fibonacci {
    a: u64,
    b: u64,
}

impl Fibonacci {
    fn new() -> Self {
        Self { a: 0, b: 1 }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        let result = self.a;
        (self.a, self.b) = (self.b, self.a + self.b);
        Some(result)
    }
}

// Usage
let first_10: Vec<u64> = Fibonacci::new().take(10).collect();
```

## Smart Pointer Patterns

### When to Use Each

| Type | Use Case |
|------|----------|
| `Box<T>` | Heap allocation, trait objects, recursive types |
| `Rc<T>` | Shared ownership (single-threaded) |
| `Arc<T>` | Shared ownership (multi-threaded) |
| `Cell<T>` | Interior mutability for Copy types |
| `RefCell<T>` | Interior mutability with runtime borrow checking |
| `Mutex<T>` | Thread-safe interior mutability |
| `RwLock<T>` | Thread-safe read-heavy interior mutability |
| `Cow<T>` | Clone-on-write, avoid unnecessary allocation |

### Trait Objects vs Generics

```rust
// Use generics when type is known at compile time (zero-cost)
fn process<H: Handler>(handler: &H, request: Request) -> Response {
    handler.handle(request)
}

// Use trait objects when type varies at runtime (dynamic dispatch)
fn get_handler(config: &Config) -> Box<dyn Handler> {
    match config.mode {
        Mode::Fast => Box::new(FastHandler::new()),
        Mode::Safe => Box::new(SafeHandler::new()),
    }
}
```

## Quick Reference: Rust Idioms

| Idiom | Description |
|-------|-------------|
| `if let Some(x) = opt` | Pattern match Option |
| `value?` | Propagate errors |
| `.unwrap_or(default)` | Default on None/Err |
| `.unwrap_or_else(\|\| expr)` | Lazy default |
| `impl Into<String>` | Accept both &str and String |
| `impl AsRef<Path>` | Accept both &str and PathBuf |
| `#[derive(...)]` | Auto-implement common traits |
| `todo!()` | Placeholder for unimplemented code |
| `#[must_use]` | Warn if return value is ignored |
| `#[non_exhaustive]` | Allow adding enum variants later |

## Anti-Patterns to Avoid

```rust
// Bad: Stringly typed
fn set_status(order: &mut Order, status: &str) { /* ... */ }

// Good: Enum typed
fn set_status(order: &mut Order, status: OrderStatus) { /* ... */ }

// Bad: Clone everything
fn process(items: Vec<Item>) -> Vec<String> {
    items.iter().map(|i| i.name.clone()).collect()
}

// Good: Take ownership if you need it, borrow if you don't
fn process(items: &[Item]) -> Vec<&str> {
    items.iter().map(|i| i.name.as_str()).collect()
}

// Bad: Index-based iteration
for i in 0..items.len() {
    process(&items[i]);
}

// Good: Iterator-based
for item in &items {
    process(item);
}

// Bad: Arc<Mutex<T>> when not shared across threads
let data = Arc::new(Mutex::new(vec![]));

// Good: Use simpler types when single-threaded
let data = RefCell::new(vec![]);

// Bad: Returning impl Trait from a trait method (not allowed)
// Good: Return Box<dyn Trait> or use associated types
```
