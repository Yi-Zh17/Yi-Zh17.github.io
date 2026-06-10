+++
date = '2026-03-12T08:36:22Z'
title = 'High-Performance KV-Store Server'
ShowToc = true
+++

# Why an In-Memory KV-Store?

In areas like High-Frequency Trading (HFT) and edge computing, system requirements emphasize not just overall throughput, but deterministic performance. Traditional relational databases often introduce high latency variance due to disk I/O and network overhead. To address this, I built an in-memory Key-Value (KV) Store in C++ to explore how to implement data querying that meets those requirements.

An in-memory KV store is a database that lives in RAM instead of on disk. It provides ultra-fast read/write operations on unique keys paired with values, and is especially useful for caching, real-time analysis, session storage, and similar workloads. The goals I had in mind were low latency, high throughput, predictable performance, and concurrency support.

I also wanted to practice deploying software in heterogeneous environments. After developing the project locally on an x86 machine, I containerized it using Docker and set up a GitHub Actions CI/CD pipeline to deploy it to an ARM-based AWS Graviton instance — partly as a sanity check that the system was genuinely portable across architectures.

## Tech Stack

- **Language:** C++17 (with a focus on RAII and memory safety)
- **Build System:** Makefile
- **DevOps:** Docker & GitHub Actions (CI/CD)
- **Cloud:** AWS EC2 (ARM/Graviton)

---

# System Architecture Overview

## Design Principles

Two concerns shaped the architecture from the start:

**Mitigating $O(n)$ search times.** Standard hash maps like `std::unordered_map` handle collisions with linked lists. As the load factor grows, traversing those lists pushes lookup time toward $O(n)$. A custom hash table is needed to avoid this and keep operations at $O(1)$ even under heavy load.

**Minimising OS allocator overhead.** Every `new` or `malloc` call goes through the OS allocator, which introduces non-deterministic latency due to system calls and internal lock contention. The solution is to reserve a large block of memory upfront and manage allocations manually — so that during steady-state operation, no heap allocation happens at all.

## Core Components

The system is split into five components to keep things modular:

- **Memory Pool:** Manages raw memory, bypassing the standard heap.
- **Hash Table:** The core data structure, optimised for fast lookups and cache efficiency.
- **Thread Pool:** Manages worker threads that process concurrent client requests.
- **Server:** A TCP interface that handles network communication via Linux `epoll`.
- **Logger:** A thread-safe logging utility for monitoring system state.

## Data Flow

Here is how the command `SET foo bar` flows through all five components.

### 1. Server — Receives, parses & dispatches bytes

The server loops on `epoll_wait`. When a command arrives, it is read into a stack buffer and parsed into a vector of `string_view`s. Those views are then copied into `std::string`s and assembled into a `Task` struct (which also carries the client's file descriptor). The struct is pushed onto the thread pool's task queue.

### 2. Thread Pool — Queues & delegates tasks to workers

When a task is enqueued, a sleeping worker thread is woken up via a condition variable. The worker pops the task, unlocks the queue immediately so other workers can proceed, and then dispatches the command — calling `insert`, `get`, or `remove` on the hash table as appropriate.

### 3. Hash Table — Robin Hood open-addressing insertion

To insert, the hash table hashes the key to get a starting index, then calls `allocate()` on the thread-local memory pool. The key and value are copied into the allocated slot using `memcpy`. The table then probes for a position using Robin Hood hashing. If a slot with the same key is found, the old entry is deallocated and overwritten; otherwise the new entry is placed according to the Robin Hood invariant.

### 4. Memory Pool — Free list allocation

Each worker thread owns its own memory pool. At startup, the pool reserves a large block of memory, divides it into fixed-size chunks, and chains them into a free list. When `allocate()` is called, it pops the head of the list and returns it — no system call, no lock, constant time. Deallocation is equally simple: the returned chunk is prepended back onto the list.

### 5. Thread Pool — Sends the response

After a successful insertion, the worker writes `+OK\r\n` back to the client file descriptor.

Here is a diagram of the full flow:

```
Client
  │  "SET foo bar\n"
  ▼
Server (epoll_wait)
  │  read() → stack buffer
  │  parseMessage() → string_views → owned strings
  │  Task{fd, ["SET","foo","bar"]}
  ▼
ThreadPool::enqueue()
  │  mutex lock → push → notify_one
  ▼
Worker thread (thread_tasks)
  │  pop Task, unlock immediately
  │  dispatch: SET → table->insert("foo","bar")
  ▼
HashTable::insert()
  │  write-lock rw_lock
  │  hash("foo") → index
  │  local_pool.allocate()  ← no malloc, no lock
  │  placement-new KV, memcpy strings
  │  Robin Hood probe → place Entry
  │  return true
  ▼
Worker thread
  │  write(fd, "+OK\r\n", 5)
  ▼
Client receives "+OK"
```

---

# Memory Pool

This component is central to achieving predictable latency. By managing memory at the application level, the system avoids the overhead and variance that comes with the standard heap.

## Why `std::byte`?

The memory pool is built on a pre-allocated `std::vector<std::byte>`. Introduced in C++17, `std::byte` is designed specifically for raw memory manipulation. Unlike `char` or `unsigned char`, it does not support arithmetic operations, which prevents accidental arithmetic bugs while still allowing the bitwise operations needed for low-level memory work.

## The Problem: Random Deallocation

A naive pool might just increment a pointer for each allocation. That works fine until entries start being deleted. A KV store has unpredictable deletion patterns, so a bump-pointer pool quickly fills up with holes — freed chunks scattered throughout the buffer that can't be reused without an expensive linear scan.

## The Solution: A Free List

To handle fragmentation without giving up speed, the pool uses a free list. This keeps both allocation and deallocation at strictly $O(1)$, regardless of how fragmented the pool becomes.

**1. Initialisation:**

During construction, the pool formats the pre-allocated vector so that each chunk stores a pointer to the next chunk in the chain. A private pointer `slot` is set to point to the first chunk.

{{< figure src="memory-pool.png" align="center" caption="Memory pool initialised state" >}}

**2. Allocation ($O(1)$):**

When memory is requested, the pool:

- Reads the address currently stored in `slot`.
- Updates `slot` to point to the address stored *within* that chunk (i.e. the next free slot).
- Returns the original address to the caller.

**3. Deallocation ($O(1)$):**

When memory is freed:

- The pool writes the current `slot` address into the newly freed chunk.
- `slot` is updated to point to the freed chunk, making it the new head.

{{< figure src="memory-pool-modified.png" align="center" caption="After some allocations and deallocations" >}}

## Thread-Local Ownership

Each worker thread owns its pool as a `thread_local` variable:

```cpp
thread_local MemoryPool local_pool(DEFAULT_CHUNK_NUM, DEFAULT_CHUNK_SIZE);
```

This is what makes the pool lock-free. Because no two threads share a pool, `allocate()` and `deallocate()` never need synchronisation — they are just pointer operations. The hash table's write lock (`rw_lock`) still protects the shared table structure, but the memory allocation itself is completely off the critical path.

## Performance Impact

The free list design guarantees constant-time memory operations regardless of the pool's fragmentation level. This is what gives the system its low-latency variance — the allocator will never stall a worker thread waiting on a system call or a lock that another thread holds.

---

# Hash Table

The hash table is a flat open-addressing table using Robin Hood hashing. The key motivation for open addressing over chaining is cache efficiency: all entries live in a single contiguous array, so lookups stay cache-friendly even when probing past the initial bucket.

## Structure

Each slot in the table is an `Entry`:

```cpp
struct Entry {
    SlotState state = EMPTY;  // EMPTY, OCCUPIED, or DELETED
    int dib = 0;              // Distance from Initial Bucket
    KV* data;                 // Pointer into the memory pool
};
```

The `dib` field (distance from initial bucket) is the key to Robin Hood hashing. It tracks how far an entry has been displaced from the slot it originally hashed to.

The `KV` struct holds fixed-size arrays for the key and value:

```cpp
struct KV {
    std::array<char, KEY_SIZE>   key;    // max 64 bytes
    std::array<char, VALUE_SIZE> value;  // max 192 bytes
};
```

Using fixed-size arrays keeps the layout predictable and avoids any dynamic allocation inside the hash table itself.

## Robin Hood Hashing

Standard open addressing tends to form long probe chains for "unlucky" keys — ones that hash to a slot already occupied by many others. Robin Hood hashing fixes this by enforcing a fairness rule: if the entry being inserted has travelled further from its home than the entry currently sitting in a slot, they swap. The displaced entry continues probing.

The effect is that probe distances are kept roughly equal across all entries, which bounds worst-case lookup time and keeps average probe length short.

## Insertion

```cpp
bool HashTable::insert(string_view key_view, string_view value_view) {
    // ...
    size_t index = hash(key_view) % capacity;
    void* mem = local_pool.allocate();
    KV* newKV = new (mem) KV();  // placement new — no heap allocation
    // memcpy key and value into newKV ...

    Entry current_entry{ .state = OCCUPIED, .dib = 0, .data = newKV };

    while (true) {
        if (table[index].state == EMPTY || table[index].state == DELETED) {
            table[index] = current_entry;
            return true;
        }
        if (string_view(table[index].data->key.data()) == key_view) {
            local_pool.deallocate(table[index].data);  // free old KV
            table[index] = current_entry;              // overwrite
            return true;
        }
        if (current_entry.dib > table[index].dib) {
            swap(current_entry, table[index]);  // Robin Hood swap
        }
        current_entry.dib++;
        index = (index + 1) % capacity;
    }
}
```

Placement new (`new (mem) KV()`) constructs the `KV` object directly in the memory pool chunk, bypassing the heap entirely.

## Lookup and the Early-Exit Optimisation

The `get` implementation takes advantage of the Robin Hood invariant to terminate early:

```cpp
if (table[index].state == OCCUPIED && current_distance > table[index].dib) {
    return nullopt;  // early exit
}
```

If the probe has already travelled further than the entry sitting in the current slot, the key being searched for cannot exist further along — it would have stolen that slot during insertion. This means `GET` on a missing key does not have to scan the entire table.

## Deletion and Tombstones

Deletion marks the slot as `DELETED` rather than `EMPTY`:

```cpp
table[index].state = DELETED;
```

This distinction matters because of how probing works. If a slot were cleared to `EMPTY` on deletion, a subsequent `GET` would stop probing at that gap and incorrectly report a miss, even if the key was inserted further along in the probe chain. The `DELETED` tombstone lets probing continue through the gap.

The downside is that tombstones accumulate over time, slightly lengthening probe chains. This is a known trade-off with open addressing and would need compaction logic in a long-running system.

## Concurrency

A `std::shared_mutex` provides reader-writer locking on the table. `GET` takes a shared lock, so multiple reads can proceed in parallel. `SET` and `DEL` take an exclusive lock. This is a coarse-grained approach — the whole table is locked per operation — but it is straightforward to reason about and sufficient for the workload this system targets.

---

# Thread Pool

The thread pool creates one worker thread per logical CPU core using `std::thread::hardware_concurrency()`. All workers share a single task queue protected by a `std::mutex` and a `std::condition_variable`.

## Lifecycle

Workers run `thread_tasks()` in a loop:

```cpp
void ThreadPool::thread_tasks() {
    while (true) {
        unique_lock<mutex> lock(queue_mutex);
        condition.wait(lock, [this] {
            return this->stop || !this->tasks.empty();
        });

        if (this->stop && this->tasks.empty()) return;

        Task t = move(this->tasks.front());
        this->tasks.pop();
        lock.unlock();  // release the queue before doing work

        // ... dispatch SET / GET / DEL ...
    }
}
```

A few things are worth noting here. The `condition.wait` call uses a predicate to guard against spurious wakeups — the thread will only proceed if there is actually a task to process or a shutdown has been requested. The lock is released immediately after the task is popped, so other workers do not have to wait while one thread is busy doing a hash table operation. And `std::move` is used when taking tasks off the queue to avoid copying the strings inside the command vector.

## Shutdown

The destructor sets `stop = true`, notifies all workers, and joins them:

```cpp
ThreadPool::~ThreadPool() {
    queue_mutex.lock();
    this->stop = true;
    queue_mutex.unlock();
    condition.notify_all();
    for (auto& worker : workers) worker.join();
}
```

Workers check the `stop` flag alongside the empty-queue condition, so they drain any remaining tasks before exiting cleanly.

## Thread-Local Pool Initialisation

Each worker thread's memory pool is declared as a `thread_local` variable inside `HashTable.cpp`:

```cpp
thread_local MemoryPool local_pool(DEFAULT_CHUNK_NUM, DEFAULT_CHUNK_SIZE);
```

`thread_local` in C++ means the variable is initialised the first time each thread accesses it, and destroyed when that thread exits. Since `local_pool` is used only inside hash table operations — which are always called from a worker thread — each worker gets its own pool on its first operation and holds it for its lifetime.

---

# Server

The server handles networking via Linux `epoll`, which lets a single thread monitor many connections simultaneously without blocking.

## Setup

On construction, the server creates a TCP socket, sets `SO_REUSEADDR` so the port can be reused immediately after a crash, binds to the given port, and starts listening. The socket is then set to non-blocking mode and registered with `epoll`.

## Event Loop

The main loop calls `epoll_wait` to block until something happens:

```cpp
while (true) {
    int num_events = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
    for (int i = 0; i < num_events; i++) {
        if (events[i].data.fd == this->fd) {
            // New connection: accept, set non-blocking, register with epoll
        } else {
            // Existing client: read, parse, enqueue task
        }
    }
}
```

New connections are accepted, set to non-blocking, and registered with `epoll` so their subsequent messages are handled in the same loop. If a read returns zero or negative, the connection is closed.

## Dual Parser

Clients can send commands in two formats: plain text (e.g. `SET foo bar\n`) or the Redis RESP protocol (e.g. `*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n`). The parser checks the first byte to decide which path to take:

```cpp
vector<string_view> Server::callParse(char* buffer) {
    if (*buffer == '*') return parseRESP(buffer);
    else                return parseMessage(buffer);
}
```

Supporting RESP means the server can be tested with `redis-cli` directly, which is convenient for debugging. The plain-text parser is a simple sliding-pointer tokeniser that splits on spaces.

Both parsers return `string_view`s into the stack buffer. These are immediately copied into owned `std::string`s before being moved into a `Task`, because the buffer will be overwritten on the next read.

---

# DevOps: Docker & CI/CD

## Multi-Stage Docker Build

The Dockerfile uses a two-stage build to keep the final image small:

```dockerfile
# Stage 1: build
FROM gcc:latest AS builder
WORKDIR /app
COPY . ./
RUN make server

# Stage 2: runtime
FROM debian:trixie-slim
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

The builder stage compiles the binary inside a full GCC image. The runtime stage starts fresh from a minimal Debian image and copies only the compiled binary. The result is an image that contains no build tools or source files — just the server binary and its runtime dependencies.

## CI/CD with GitHub Actions

A GitHub Actions workflow builds and tests the project on every push. The pipeline targets an AWS Graviton (ARM) instance, which required setting up cross-compilation or using QEMU emulation during the build step. This was the main engineering challenge on the DevOps side: the local development machine is x86, so ensuring the binary built correctly for ARM needed explicit attention to the toolchain.

The CI pipeline runs the test binaries from the `test/` directory after each build, catching any regressions before they reach the deployed instance.

---

# Benchmarks

## Reproducing Environment

Because the server is sensitive to CPU topology, memory bandwidth, kernel scheduling, and Docker overhead, these benchmark numbers should be read in the context of the machine they were measured on.

| Component | Details |
|---|---|
| Machine | Lenovo Legion Y7000P 2020H |
| CPU | Intel Core i7-10875H, 8 cores / 16 threads, up to 5.10 GHz |
| Architecture | x86_64 |
| Memory | 15 GiB RAM |
| OS | Linux Mint 22.2 (Ubuntu 24.04 base), Linux kernel 6.8.0-124-generic |
| Compiler | GCC 13.3.0 |
| Runtime | Docker 29.5.3 |
| Benchmark tool | `memtier_benchmark` 64-bit, libevent 2.1.12-stable |

## Result

The server was benchmarked using `memtier_benchmark` with 4 threads and 50 concurrent clients against a running Docker container on the same machine:

| Metric | Result |
|---|---|
| Throughput | ~244,000 ops/sec |
| p99 latency | 1.08 ms |

The p99 latency staying under 2 ms across the full workload reflects the main design goal: not just high throughput, but low variance. Standard allocators can cause latency spikes at the tail because allocation time is non-deterministic — the free list pool largely eliminates that source of variance.

---

# Limitations

This project was built as a learning exercise in systems programming, and there are a few significant limitations worth being upfront about.

**No persistence.** All data lives in RAM. A crash or restart wipes everything. A production system would need a WAL (Write-Ahead Log) or periodic snapshots to disk.

**No eviction.** The memory pool is fixed-size. When all `DEFAULT_CHUNK_NUM` chunks (10,000 per thread) are used up, `allocate()` returns `nullptr` and insertions fail silently with `-ERR insertion failed`. There is no LRU or any other eviction strategy.

**Tombstone accumulation.** The `DELETED` tombstone approach means deleted slots are never truly reclaimed. A long-running server with heavy deletion traffic would see probe chains lengthen over time. Compaction or Robin Hood backward-shifting on deletion would fix this.

**Coarse locking.** The single `shared_mutex` on the hash table means all writers serialise. Under write-heavy workloads with many cores, this becomes a bottleneck. Sharding the table into independent buckets, each with its own lock, is the standard approach for scaling this further.

**Key and value size limits.** Keys are capped at 64 bytes and values at 192 bytes. These are compile-time constants (`KEY_SIZE` and `VALUE_SIZE`), so there is no dynamic sizing. Anything larger is silently rejected.
