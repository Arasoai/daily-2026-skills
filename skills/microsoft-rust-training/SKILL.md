---
name: microsoft-rust-training
description: Comprehensive Rust training curriculum from Microsoft covering beginner through expert levels across seven structured books with exercises and deep dives.
triggers:
  - "rust training material"
  - "learn rust from scratch"
  - "rust for C++ programmers"
  - "rust async deep dive"
  - "rust patterns and best practices"
  - "type driven correctness rust"
  - "rust engineering practices"
  - "serve rust training books locally"
---

# Microsoft Rust Training

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

Microsoft's RustTraining is a seven-book curriculum covering Rust from beginner to expert level. Books are organized by background (C/C++, C#, Python) and topic (async, patterns, type-level correctness, engineering practices). Each book contains 15–16 chapters with Mermaid diagrams, editable Rust playgrounds, and exercises.

---

## Installation & Setup

### Prerequisites

Install Rust via [rustup](https://rustup.rs/):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
```

### Clone and install tooling

```bash
git clone https://github.com/microsoft/RustTraining.git
cd RustTraining

cargo install mdbook mdbook-mermaid
```

### Build and serve all books

```bash
cargo xtask build        # Build all books → site/
cargo xtask serve        # Build + serve at http://localhost:3000
cargo xtask deploy       # Build all books → docs/ (GitHub Pages output)
cargo xtask clean        # Remove site/ and docs/
```

### Serve a single book

```bash
cd async-book && mdbook serve --open        # http://localhost:3000
cd c-cpp-book && mdbook serve --open
cd python-book && mdbook serve --open
```

---

## Book Map

| Book Directory | Level | Audience |
|---|---|---|
| `c-cpp-book/` | 🟢 Bridge | C / C++ programmers |
| `csharp-book/` | 🟢 Bridge | C# / Java / Swift programmers |
| `python-book/` | 🟢 Bridge | Python programmers |
| `async-book/` | 🔵 Deep Dive | Tokio, streams, cancellation |
| `rust-patterns-book/` | 🟡 Advanced | Pin, allocators, lock-free, unsafe |
| `type-driven-correctness-book/` | 🟣 Expert | Type-state, phantom types, capabilities |
| `engineering-book/` | 🟤 Practices | Build scripts, CI/CD, Miri, cross-compilation |

---

## Key Concepts by Book

### Bridge Books — Ownership & Borrowing

```rust
// Ownership: value moves into the function, caller can't use it after
fn take_ownership(s: String) {
    println!("{s}");
} // s is dropped here

// Borrowing: caller retains ownership
fn borrow(s: &String) {
    println!("{s}");
}

// Mutable borrow: only one at a time
fn mutate(s: &mut String) {
    s.push_str(" world");
}

fn main() {
    let mut greeting = String::from("hello");
    mutate(&mut greeting);
    borrow(&greeting);
    take_ownership(greeting);
    // println!("{greeting}"); // compile error: moved
}
```

### Bridge Books — Error Handling

```rust
use std::num::ParseIntError;
use std::fmt;

#[derive(Debug)]
enum AppError {
    Parse(ParseIntError),
    OutOfRange(i32),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Parse(e) => write!(f, "parse error: {e}"),
            AppError::OutOfRange(n) => write!(f, "{n} is out of range"),
        }
    }
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self {
        AppError::Parse(e)
    }
}

fn parse_positive(s: &str) -> Result<i32, AppError> {
    let n: i32 = s.parse()?;   // ? uses From<ParseIntError>
    if n < 0 { return Err(AppError::OutOfRange(n)); }
    Ok(n)
}
```

### Async Book — Tokio Basics

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (a, b) = tokio::join!(
        fetch("https://example.com/a"),
        fetch("https://example.com/b"),
    );
    println!("{a:?} {b:?}");
}

async fn fetch(url: &str) -> Result<String, reqwest::Error> {
    reqwest::get(url).await?.text().await
}
```

```rust
// Cancellation-safe pattern with select!
use tokio::sync::mpsc;

async fn process(mut rx: mpsc::Receiver<String>) {
    loop {
        tokio::select! {
            Some(msg) = rx.recv() => {
                handle(msg).await;
            }
            _ = tokio::signal::ctrl_c() => {
                println!("Shutting down");
                break;
            }
        }
    }
}

async fn handle(msg: String) {
    println!("Got: {msg}");
}
```

### Rust Patterns Book — Pin and Self-Referential Types

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

struct SelfRef {
    data: String,
    ptr: *const String,   // points into data
    _pin: PhantomPinned,
}

impl SelfRef {
    fn new(s: &str) -> Pin<Box<Self>> {
        let mut boxed = Box::pin(SelfRef {
            data: s.to_string(),
            ptr: std::ptr::null(),
            _pin: PhantomPinned,
        });
        // Safety: we never move boxed after this
        let ptr = &boxed.data as *const String;
        unsafe { boxed.as_mut().get_unchecked_mut().ptr = ptr; }
        boxed
    }

    fn get(&self) -> &str {
        // Safety: ptr is valid as long as self is pinned
        unsafe { (*self.ptr).as_str() }
    }
}
```

### Rust Patterns Book — Custom Allocator

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering};

struct TrackingAllocator {
    allocated: AtomicUsize,
}

unsafe impl GlobalAlloc for TrackingAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let ptr = System.alloc(layout);
        if !ptr.is_null() {
            self.allocated.fetch_add(layout.size(), Ordering::Relaxed);
        }
        ptr
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        System.dealloc(ptr, layout);
        self.allocated.fetch_sub(layout.size(), Ordering::Relaxed);
    }
}

#[global_allocator]
static ALLOCATOR: TrackingAllocator = TrackingAllocator {
    allocated: AtomicUsize::new(0),
};
```

### Type-Driven Correctness — Type-State Pattern

```rust
use std::marker::PhantomData;

struct Locked;
struct Unlocked;

struct Safe<State> {
    contents: String,
    _state: PhantomData<State>,
}

impl Safe<Locked> {
    pub fn new(contents: &str) -> Self {
        Safe { contents: contents.to_string(), _state: PhantomData }
    }

    pub fn unlock(self, _pin: u32) -> Safe<Unlocked> {
        Safe { contents: self.contents, _state: PhantomData }
    }
}

impl Safe<Unlocked> {
    pub fn read(&self) -> &str {
        &self.contents
    }

    pub fn lock(self) -> Safe<Locked> {
        Safe { contents: self.contents, _state: PhantomData }
    }
}

fn main() {
    let safe = Safe::new("secret");
    // safe.read();  // compile error: method not on Locked
    let open = safe.unlock(1234);
    println!("{}", open.read());
    let _locked = open.lock();
}
```

### Type-Driven Correctness — Phantom Types for Unit Safety

```rust
use std::marker::PhantomData;
use std::ops::Add;

struct Meters;
struct Seconds;

#[derive(Clone, Copy, Debug)]
struct Quantity<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<U> Quantity<U> {
    fn new(value: f64) -> Self {
        Quantity { value, _unit: PhantomData }
    }
}

impl<U> Add for Quantity<U> {
    type Output = Self;
    fn add(self, rhs: Self) -> Self {
        Quantity::new(self.value + rhs.value)
    }
}

fn main() {
    let d1 = Quantity::<Meters>::new(10.0);
    let d2 = Quantity::<Meters>::new(5.0);
    let total = d1 + d2;
    println!("Total: {} m", total.value);

    // let t = Quantity::<Seconds>::new(3.0);
    // let bad = d1 + t;  // compile error: mismatched types
}
```

### Engineering Book — Build Scripts

```rust
// build.rs — generate code or link native libraries
fn main() {
    // Re-run only when these files change
    println!("cargo:rerun-if-changed=src/proto/api.proto");
    println!("cargo:rerun-if-changed=build.rs");

    // Link a native library
    println!("cargo:rustc-link-lib=ssl");
    println!("cargo:rustc-link-search=native=/usr/lib/x86_64-linux-gnu");

    // Expose a compile-time env var
    let profile = std::env::var("PROFILE").unwrap();
    println!("cargo:rustc-env=BUILD_PROFILE={profile}");
}
```

---

## Common Patterns

### Idiomatic iterator chains

```rust
fn word_lengths(text: &str) -> Vec<(String, usize)> {
    text.split_whitespace()
        .map(|w| (w.to_lowercase(), w.len()))
        .filter(|(_, len)| *len > 3)
        .collect()
}
```

### Using `thiserror` for library errors

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DbError {
    #[error("connection refused: {0}")]
    Connection(String),
    #[error("query failed: {source}")]
    Query { #[from] source: sqlx::Error },
    #[error("not found: id={id}")]
    NotFound { id: u64 },
}
```

### Newtype pattern for API safety

```rust
struct UserId(u64);
struct OrderId(u64);

fn get_order(user: UserId, order: OrderId) -> Option<String> {
    // Can't accidentally swap user/order IDs at call site
    Some(format!("user={} order={}", user.0, order.0))
}
```

---

## Repository Structure

```
RustTraining/
├── async-book/
│   └── src/          # .md chapters + SUMMARY.md
├── c-cpp-book/src/
├── csharp-book/src/
├── python-book/src/
├── rust-patterns-book/src/
├── type-driven-correctness-book/src/
├── engineering-book/src/
├── xtask/            # cargo xtask build/serve/deploy/clean
├── site/             # local preview output (gitignored)
├── docs/             # GitHub Pages output
└── .github/workflows/pages.yml
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `cargo xtask serve` not found | Run from repo root; check `xtask/` directory exists |
| `mdbook` command not found | `cargo install mdbook mdbook-mermaid` |
| Mermaid diagrams not rendering | Ensure `mdbook-mermaid` is installed and `book.toml` has the preprocessor |
| Port 3000 in use | `mdbook serve --port 3001` |
| `rustup` not in PATH after install | `source "$HOME/.cargo/env"` or restart shell |
| Build fails on `pages.yml` locally | Deploy workflow is CI-only; use `cargo xtask build` locally |

### Verify your toolchain

```bash
rustc --version        # rustc 1.x.x
cargo --version        # cargo 1.x.x
mdbook --version       # mdbook vX.Y.Z
mdbook-mermaid --version
```

### Check a book builds cleanly

```bash
cd async-book
mdbook build 2>&1 | grep -i error
```

---

## Contributing

- Source for each book lives in `<book-dir>/src/` as Markdown files.
- `SUMMARY.md` in each `src/` controls chapter order and sidebar.
- Diagrams use [Mermaid](https://mermaid.js.org/) fenced code blocks (` ```mermaid `).
- Exercises are inline code blocks or links to the [Rust Playground](https://play.rust-lang.org/).
- GitHub Actions deploys `docs/` to GitHub Pages on every push to `master`.
