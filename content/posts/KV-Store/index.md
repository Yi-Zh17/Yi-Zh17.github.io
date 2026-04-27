+++
date = '2026-03-12T08:36:22Z'
title = 'High-Performance KV-Store Server'
ShowToc = true
+++

# Why an In-Memory KV-Store?

In areas like High-Frequency Trading (HFT) and edge computing, system requirements emphasize not just overall throughput, but deterministic performance. Traditional relational databases often introduce high latency variance due to disk I/O and network overhead. To address this, I decided to build an in-memory Key-Value (KV) Store in C++ to explore how to implement data querying that satisfies the requirements.

An in-memory KV store is a database that runs in the RAM instead of on disks. It provides ultra-fast read/write operations of unique keys paired with values. It is especially useful for caching, real-time analysis, session storage, etc. The goal of my implementation is to achieve low latency, high throughput, predictable performance, and concurrency support.

Additionally, I wanted to practice deploying software in heterogeneous environments. After developing the project locally on an x86 machine, I containerized the application using Docker and set up a GitHub Actions CI/CD pipeline to deploy it to an ARM-based AWS Graviton instance. This process helped me verify the system's cross-architecture portability.

## Tech Stack

- **Language:** C++17 (Focusing on RAII and memory safety)

- **Build System:** Makefile

- **DevOps:** Docker & GitHub Actions (CI/CD)

- **Cloud:** AWS EC2 (ARM/Graviton)

---

# System Architecture Overview

## Design Principles
To achieve the goals defined above, here are two major principles I had in mind while designing the architecture of the system:

- **Mitigating $O(n)$ Search Times:** Standard hash maps, such as `std::unordered_map`, typically handle hash collisions using linked lists. As the load factor increases, traversing these lists degrades performance toward $O(n)$. Therefore, a cutomized Hash Table is required to minimize this degradation and maintain $O(1)$ lookup times even under heavy load.

- **Minimizing OS Allocator Overhead:** Using standard `new` or `malloc` calls involves the OS memory allocator, introducing non-deterministic execution time due to system calls and lock contention. To resolve this, a "zero-allocation" solution is desired — instead of performing heap allocation for every request, the system would reserve a large chunk of memory on the heap and manage the memory allocations on its own.

## Core Components

To keep the system modular and maintainable, I divided it into five components:

- **Memory Pool:** Handles raw memory management, bypassing the standard heap allocations.

- **Hash Table:** The core data structure, optimized for fast lookups and cache efficiency.

- **Thread Pool:** Manages worker threads to process multiple concurrent client requests.

- **Server:** A TCP interface that handles network communication.

- **Logger:** A thread-safe logging utility to monitor system states.

## Data Flow

I now give an example of how the command `SET foo bar` will flow through the components mentioned above.

### 1. Server — Receives, parses & despatches bytes

First of all, the started server looping on `epoll_wait` will read the command into a buffer. The buffer will be parsed into a vector of `string_view`s. The `string_view`s are then copied into `std::string`s and assembled into a `Task` struct (which also contains the client's file descriptor). The struct is then pushed into a `queue` managed by the thread pool.

### 2. Thread Pool — Queues & delegates tasks to workers

After a task is pushed to the `queue` in the thread pool, a worker will be waken up to pop the task out of the `queue` and execute the task. During the execution, the worker will read the command and perform corresponding operations (`SET`, `GET` & `DEL`) on the hash table.

### 3. Hash Table — Robin Hood open-addressing insertion

To perform an insertion, the hash table first calculates the hash of the key and its corresponding index. It then calls a memory allocation function in the thread-local memory pool (customized and distinct from heap allocation, described in the section below). The key-value is copied into the allocated memory slot. The hash table then updates its value by probing slots with Robin Hood hashing. If a slot with the same key is found, the old key-value pair is deallocated and overwritten; otherwise the slot will point to the new key-value pair.

### 4. Memory Pool — Slab allocation

Each worker thread owns its own memory pool. During the startup of the system, the memory pool will reserve a large memory chunk, divide it into a fixed number of smaller chunks, and chain them into a free list. When the hash table calls the `allocate()` function, it simply pops the head pointer, update it with the next available slot, and decrement the number of available slots. This operation takes constant time with zero system calls.

### 5. Thread Pool — Sends the response

After the insertion returns `true` from the hash table, the worker writes "+OK\r\n" to the cliend file descriptor.

Here is a simple diagram showing the workflow:

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

## Memory Pool

This component is central to achieving predictable latency. By managing memory at the application level, the system avoids the overhead and variance associated with the standard heap.

### Why `std::byte`?

The memory pool is built on a pre-allocated `std::vector` of `std::byte`. Introduced in C++17, `std::byte` is designed specifically for raw memory manipulation. Unlike `char` or `unsigned char`, it does not support standard arithmetic operations. This prevents accidental arithmetic bugs while still permitting the bit-wise operations necessary for low-level memory handling.

### The Problem: Random Deallocation

A naive memory pool might simply increment a pointer for each allocation. However, because a KV-store experiences random deallocations, this approach quickly exhausts the vector's capacity, leaving fragmented, unusable memory spaces behind. Searching linearly for these freed spaces to reuse them would introduce unacceptable latency into the system.

### The Solution: A Linked Free List

To address fragmentation without sacrificing speed, I implemented a Free List. This structure tracks the next available memory slot, ensuring both allocation and deallocation remain strictly $O(1)$ operations.

**1. Initialization:**

During construction, the memory pool formats the pre-allocated vector so that each empty slot stores a pointer to the next available slot. A private pointer, `next_slot`, is initialized to point to the first index.

{{< figure src="memory-pool.png" align="center" caption="Memory pool initialized state" >}}

**2. Allocation ($O(1)$):**

When memory is requested, the pool performs the following steps without needing to search:

- Reads the address currently held by `next_slot`.

- Updates `next_slot` to point to the address stored *within* that chunk (the next available slot in the chain).

- Returns the original address to the caller.

**3. Deallocation ($O(1)$):**

When memory is freed, it is threaded back into the chain:

- The pool writes the current `next_slot` address into the newly freed chunk.

- `next_slot` is updated to point to this newly freed chunk.

{{< figure src="memory-pool-modified.png" align="center" caption="After some allocations and deallocations" >}}

### Performance Impact

This Free List design guarantees that finding or returning memory takes constant time, regardless of the pool's current capacity or fragmentation level. This constant-time operation is essential for maintaining the low-variance response times expected in performance-critical applications.



# 