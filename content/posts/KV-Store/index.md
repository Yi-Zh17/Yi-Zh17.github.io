+++
date = '2026-03-12T08:36:22Z'
title = 'High-Performance KV-Store Server'
ShowToc = true
+++

# Why an In-Memory KV-Store?

In areas like High-Frequency Trading (HFT) and edge computing, system requirements emphasize not just overall throughput, but deterministic performance. Traditional relational databases often introduce high latency variance due to disk I/O and network overhead. To address this, I decided to build an in-memory Key-Value (KV) Store in C++ to explore how to achieve predictable, low-latency data querying.

Additionally, I wanted to practice deploying software in heterogeneous environments. After developing the project locally on an x86 machine, I containerized the application using Docker and set up a GitHub Actions CI/CD pipeline to deploy it to an ARM-based AWS Graviton instance. This process helped me verify the system's cross-architecture portability.

## Tech Stack

- **Language:** C++17 (Focusing on RAII and memory safety)

- **Build System:** Makefile

- **DevOps:** Docker & GitHub Actions (CI/CD)

- **Cloud:** AWS EC2 (ARM/Graviton)

---

# System Architecture Overview

To maintain predictable execution times, I focused on reducing two primary performance bottlenecks: network latency and memory allocation overhead. The architecture separates the data storage logic from the concurrent access management. Here is how a request flows through the system:

*(Insert Diagram Here)*

## Design Principles

- **Mitigating $O(n)$ Search Times:** Standard hash maps, such as `std::unordered_map`, typically handle hash collisions using linked lists. As the load factor increases, traversing these lists degrades performance toward $O(n)$. I designed the Hash Table to minimize this degradation and maintain $O(1)$ lookup times even under heavy load.

- **Minimizing OS Allocator Overhead:** Using standard `new` or `malloc` calls involves the OS memory allocator, introducing non-deterministic execution time due to system calls and lock contention. To resolve this, I implemented a Thread-Local Memory Pool. Each thread is assigned a pre-allocated chunk of memory, enabling independent allocations and deallocations without OS intervention.

## Core Components

To keep the system modular and maintainable, I divided it into five components:

- **Memory Pool:** Handles raw memory management, bypassing the standard heap.

- **Hash Table:** The core data structure, optimized for fast lookups and cache efficiency.

- **Thread Pool:** Manages worker threads to process multiple concurrent client requests.

- **Server:** A TCP interface that handles network communication.

- **Logger:** A thread-safe logging utility to monitor system states.

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