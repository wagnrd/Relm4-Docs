# Threads and Async

Most UI updates happen instantly, but some operations take time: network requests, file I/O, complex calculations. Relm4 provides several mechanisms to handle these without freezing your UI.

## The Problem

When an operation blocks the main thread, the entire UI freezes:

```
Without proper threading:
───────────────────────────────
User clicks button
        │
        ▼
UI thread starts heavy work
        │
        ▼
UI freezes ════════════════════
        │
        ▼
Work completes after 5 seconds
        │
        ▼
UI unfreezes
```

## Relm4's Solutions

### 1. Workers

Components without widgets that run on a separate thread. Ideal for:
- CPU-bound operations
- Blocking I/O
- Tasks that must run sequentially

### 2. Commands

Background tasks that run on a thread pool and send results back. Ideal for:
- One-shot async operations
- Network requests
- Parallel execution

### 3. Async Components

Components that can handle async operations directly. Ideal for:
- Multiple concurrent async operations
- Async streams
- Complex async workflows

## Quick Comparison

| Feature | Workers | Commands | Async Components |
|---------|---------|----------|------------------|
| **Execution** | Separate thread | Thread pool | Async runtime |
| **Concurrency** | One at a time | Multiple parallel | Multiple parallel |
| **Best for** | CPU-bound | I/O-bound | Async-native |
| **Blocking** | Yes (intentional) | No | No |
| **Memory** | Own thread stack | Shared pool | Minimal |

## When to Use Each

### Use Workers When:

- Operations must run one at a time
- You need guaranteed sequential processing
- Blocking operations that can't be made async
- Long-running computations

### Use Commands When:

- Multiple operations can run in parallel
- Operations are naturally async (HTTP requests, file I/O)
- You need fire-and-forget with result callback
- Simple one-shot tasks

### Use Async Components When:

- Complex async workflows with multiple branches
- You need to coordinate multiple async operations
- Operations depend on each other's results
- You're already using async/await extensively

## Global Thread Configuration

Relm4 uses a global thread pool for background tasks:

```rust
use relm4::{RELM_THREADS, RELM_BLOCKING_THREADS};

// Set number of worker threads
RELM_THREADS.set(4).unwrap();

// Set number of blocking threads
RELM_BLOCKING_THREADS.set(16).unwrap();
```

## Summary

- Long-running operations can freeze your UI
- Workers handle blocking tasks on dedicated threads
- Commands run async tasks on a thread pool
- Async components handle async/await natively
- Choose the right tool for your use case
