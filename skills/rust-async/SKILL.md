---
name: rust-async
description: Async Rust with Tokio, futures, concurrency patterns, channels, and performance. Use when building async services, networking, or concurrent Rust applications.
---

# Async Rust Patterns

Expert guidance for building concurrent, async applications in Rust with Tokio.

## Core Concepts

- Rust futures are lazy — they do nothing until `.await`ed or spawned
- Use `tokio` as the async runtime (default for most Rust async work)
- Prefer structured concurrency — spawn tasks with clear ownership
- Avoid blocking the async runtime — use `spawn_blocking` for CPU-heavy or blocking I/O

## Runtime Setup

```rust
// ✅ Good: Multi-threaded runtime (default)
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Your async code here
    Ok(())
}

// For libraries, don't pick a runtime — let the consumer choose
// Just return futures, don't call block_on
```

## Spawning Tasks

- Use `tokio::spawn` for independent, concurrent work
- Use `JoinHandle` to await results from spawned tasks
- Use `JoinSet` to manage groups of tasks
- Always handle errors from spawned tasks (they can panic)

```rust
use tokio::task::JoinSet;

// ✅ Good: JoinSet for managing multiple tasks
async fn fetch_all(urls: Vec<String>) -> Vec<Result<String, reqwest::Error>> {
    let mut set = JoinSet::new();

    for url in urls {
        set.spawn(async move {
            reqwest::get(&url).await?.text().await
        });
    }

    let mut results = Vec::new();
    while let Some(res) = set.join_next().await {
        match res {
            Ok(result) => results.push(result),
            Err(e) => eprintln!("Task panicked: {e}"),
        }
    }
    results
}

// ✅ Good: spawn_blocking for CPU-intensive work
async fn hash_password(password: String) -> Result<String, Error> {
    tokio::task::spawn_blocking(move || {
        bcrypt::hash(&password, bcrypt::DEFAULT_COST)
    })
    .await?
}

// ❌ Bad: Blocking the async runtime
async fn bad_hash(password: &str) -> String {
    bcrypt::hash(password, 12).unwrap() // Blocks the executor!
}
```

## Channels

- Use `tokio::sync::mpsc` for multi-producer, single-consumer
- Use `tokio::sync::broadcast` for multi-producer, multi-consumer
- Use `tokio::sync::oneshot` for single-value responses
- Use `tokio::sync::watch` for latest-value broadcasting

```rust
use tokio::sync::mpsc;

// ✅ Good: mpsc for work queues
async fn worker_pool() {
    let (tx, mut rx) = mpsc::channel::<Task>(100);

    // Spawn workers
    for _ in 0..4 {
        let mut rx = rx.clone(); // Won't compile — mpsc rx isn't Clone
    }

    // Instead, use a shared receiver pattern:
    let (tx, rx) = mpsc::channel::<Task>(100);
    let rx = Arc::new(Mutex::new(rx));

    for _ in 0..4 {
        let rx = Arc::clone(&rx);
        tokio::spawn(async move {
            loop {
                let task = rx.lock().await.recv().await;
                match task {
                    Some(task) => process(task).await,
                    None => break,
                }
            }
        });
    }
}

// ✅ Good: oneshot for request-response
use tokio::sync::oneshot;

struct Request {
    data: String,
    respond_to: oneshot::Sender<Response>,
}

async fn handle_request(req: Request) {
    let result = process(&req.data).await;
    let _ = req.respond_to.send(result);
}
```

## Select & Timeouts

- Use `tokio::select!` to race multiple futures
- Always handle all branches — don't leave futures dangling
- Use `tokio::time::timeout` for deadline enforcement

```rust
use tokio::time::{timeout, Duration};

// ✅ Good: Timeout on operations
async fn fetch_with_timeout(url: &str) -> Result<String, Error> {
    timeout(Duration::from_secs(10), reqwest::get(url))
        .await
        .map_err(|_| Error::Timeout)?
        .map_err(Error::Network)?
        .text()
        .await
        .map_err(Error::Network)
}

// ✅ Good: Select for racing futures
use tokio::select;

async fn run(mut shutdown: tokio::sync::broadcast::Receiver<()>) {
    let mut interval = tokio::time::interval(Duration::from_secs(1));

    loop {
        select! {
            _ = interval.tick() => {
                do_periodic_work().await;
            }
            _ = shutdown.recv() => {
                tracing::info!("Shutting down gracefully");
                break;
            }
        }
    }
}
```

## Shared State

- Use `Arc<Mutex<T>>` for shared mutable state (prefer `tokio::sync::Mutex` for async)
- Use `Arc<RwLock<T>>` when reads vastly outnumber writes
- Use `DashMap` for concurrent hash maps without explicit locking
- Minimize lock scope — hold locks for the shortest time possible

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

// ✅ Good: Shared state with RwLock
#[derive(Clone)]
struct AppState {
    db: Pool<Postgres>,
    cache: Arc<RwLock<HashMap<String, CachedItem>>>,
}

async fn get_cached(state: &AppState, key: &str) -> Option<CachedItem> {
    // Read lock — multiple readers allowed
    state.cache.read().await.get(key).cloned()
}

async fn set_cached(state: &AppState, key: String, value: CachedItem) {
    // Write lock — exclusive access
    state.cache.write().await.insert(key, value);
}

// ❌ Bad: Holding lock across await points
async fn bad_update(state: &AppState) {
    let mut cache = state.cache.write().await;
    let data = fetch_from_db().await; // Lock held during I/O!
    cache.insert("key".into(), data);
}

// ✅ Good: Minimize lock scope
async fn good_update(state: &AppState) {
    let data = fetch_from_db().await; // No lock held
    state.cache.write().await.insert("key".into(), data);
}
```

## Graceful Shutdown

- Use `tokio::signal` to listen for SIGTERM/SIGINT
- Use broadcast channels or `CancellationToken` to propagate shutdown
- Drain in-flight work before exiting

```rust
use tokio_util::sync::CancellationToken;

async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let token = CancellationToken::new();

    let worker_token = token.clone();
    let worker = tokio::spawn(async move {
        loop {
            select! {
                _ = worker_token.cancelled() => break,
                _ = do_work() => {}
            }
        }
        cleanup().await;
    });

    // Wait for shutdown signal
    tokio::signal::ctrl_c().await?;
    tracing::info!("Shutdown signal received");
    token.cancel();

    worker.await?;
    Ok(())
}
```

## Streams

- Use `tokio_stream` or `futures::Stream` for async iterators
- Use `StreamExt` for combinators (`map`, `filter`, `buffer_unordered`)
- Use `buffer_unordered` for concurrent processing with backpressure

```rust
use futures::stream::{self, StreamExt};

// ✅ Good: Process stream with concurrency limit
async fn process_urls(urls: Vec<String>) -> Vec<String> {
    stream::iter(urls)
        .map(|url| async move {
            reqwest::get(&url).await?.text().await
        })
        .buffer_unordered(10) // Max 10 concurrent requests
        .filter_map(|r| async { r.ok() })
        .collect()
        .await
}
```

## Common Mistakes

```rust
// ❌ Don't hold std::sync::Mutex across .await
let guard = std_mutex.lock().unwrap();
some_async_fn().await; // DEADLOCK RISK

// ✅ Use tokio::sync::Mutex for async code
let guard = tokio_mutex.lock().await;

// ❌ Don't forget to handle JoinHandle errors
tokio::spawn(async { risky_work().await });  // Panic is silently swallowed

// ✅ Always handle spawn results
let handle = tokio::spawn(async { risky_work().await });
match handle.await {
    Ok(result) => result?,
    Err(e) => tracing::error!("Task panicked: {e}"),
}

// ❌ Don't create runtime inside async context
async fn bad() {
    let rt = tokio::runtime::Runtime::new().unwrap(); // Panics!
}

// ❌ Don't use async when sync is fine
async fn add(a: i32, b: i32) -> i32 { a + b } // Unnecessary async
fn add(a: i32, b: i32) -> i32 { a + b }       // Just use sync
```

## Performance Tips

- Use `buffer_unordered` instead of spawning unbounded tasks
- Batch database operations instead of one-at-a-time queries
- Use connection pooling (`bb8`, `deadpool`, `sqlx::Pool`)
- Profile with `tokio-console` for runtime introspection
- Set appropriate channel buffer sizes — too small causes backpressure, too large wastes memory
