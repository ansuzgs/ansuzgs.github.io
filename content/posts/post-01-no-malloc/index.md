+++
title = "What malloc() does not want you to know"
author = ["pablo"]
date = 2026-05-24
draft = false
translationKey = "post-01-no-malloc"
series = ["Memory Allocation and GC from Scratch"]
subtitle = "A Bump Allocator in 80 Lines of C"
seriesPost = 1
tags = ["c", "allocator", "malloc", "memory-management", "sbrk"]
description = "Build a working memory allocator in 80 lines of C. Learn how sbrk() works, what malloc() hides, and write your first bump pointer from scratch."
slug = "bump-allocator-c-sbrk"
+++

_Post 1 of 13 — Series: Memory Allocation and Garbage Collection from Scratch_

Every time you write `malloc(128)`, something apparently magical happens: the runtime hands you a pointer to 128 bytes of memory nobody else is using. You didn't ask the operating system for permission. You didn't specify _where_ those bytes should live. They simply appeared. And when you call `free()`, they vanish back into the void.

The magic is a lie.

Underneath `malloc()` there's nothing sophisticated. There's a syscall that moves a number upward. There's a pointer that advances. There's a data structure keeping track. Your program, any C program, is running code very much like what we're going to write today.

In this post, we'll build a **working memory allocator** in approximately 80 lines of C. It's small. It's readable. It doesn't do everything `malloc()` does, it's missing important pieces we'll add in later posts. But what it does do is _real_: it asks the operating system for memory, hands it out to callers, and manages it with a struct and three functions.

The promise of this series is simple: by the end of the 13 posts, you'll have built, with your own hands,  a complete allocator with free lists, coalescing, `mmap()`-based arenas, and a working garbage collector. All compilable, all runnable, all explained.

We start with the basics. We start with the _bump pointer_.


## Section 1: How malloc() Really Works {#section-1-how-malloc-really-works}


### The Process Address Space {#the-process-address-space}

When the Linux kernel loads your program, it assigns it a **virtual address space**. On x86-64, that space is enormous (48 bits of usable addresses), but the logical structure is the same as on any architecture: a handful of well-defined regions with fixed roles.

The two that matter to us:

<pre style="width: fit-content; margin: 0 auto;">
 High addresses (0x7fff...)
┌─────────────────────────┐
│         STACK           │  ← grows downward (each function call)
│           │             │
│           ▼             │
│                         │
│     (free space)        │
│                         │
│           ▲             │
│           │             │
│         HEAP            │  ← grows upward (each malloc)
├─────────────────────────┤
│   program break (brk)   │  ← boundary: below is yours, above is not
├─────────────────────────┤
│      BSS / Data         │
│      Text (code)        │
└─────────────────────────┘
 Low addresses (0x0000...)
</pre>

The **stack** grows downward with each function call. The **heap** grows upward with each dynamic memory request. Between them lies an ocean of unmapped virtual space.

The critical point is the **program break** (`brk`). It's a per-process variable the kernel maintains. Everything _below_ the break is memory your process can read and write. Everything _above_ is kernel territory, touch it and you get a `SIGSEGV`.


### The sbrk(2) Syscall {#the-sbrk--2--syscall}

`sbrk()` is the tool that moves that break. Its conceptual signature is trivial:

```c
void *sbrk(intptr_t increment);
```

What it does: moves the program break `increment` bytes upward (or downward if `increment` is negative). Returns the _previous_ break position, that is, the base address of the new region.

Calling `sbrk(0)` without moving anything tells you where the break is right now. It's the way to "ask" without "requesting".

<pre style="width: fit-content; margin: 0 auto;">
Before sbrk(256):
        ┌───────────────────┐
        │    free space     │
 brk ──►├───────────────────┤
        │  existing heap    │
        └───────────────────┘
After sbrk(256):
        ┌───────────────────┐
        │    free space     │
 brk ──►├───────────────────┤
        │  256 new bytes    │  ← sbrk returns this address
        ├───────────────────┤
        │  existing heap    │
        └───────────────────┘
</pre>

Once you call `sbrk(n)`, those `n` bytes are _yours_. The kernel has mapped the corresponding virtual pages to physical memory (or at least promised to do so when you touch them, _demand paging_). You can read and write them until the process ends.

There's an important invariant here: `sbrk()` doesn't give you isolated blocks. It gives you a **contiguous extension** of the heap. Everything you've requested so far forms a continuous byte array. This property is simultaneously its greatest strength (simplicity) and its greatest weakness (rigidity). In Post 7, when we migrate to `mmap()`, we'll see how to overcome this limitation.


### The Mental Model: The Linear Heap {#the-mental-model-the-linear-heap}

With what we know, the life of `malloc()` boils down to this:

1.  At startup, the heap is an empty array. `sbrk(0)` tells us where it starts.
2.  Someone requests `n` bytes. We move the break `n` bytes upward with `sbrk(n)`. We return the base address.
3.  Someone requests `m` more bytes. We move the break again. We return the new base.
4.  Repeat.

In pseudocode:

```text
allocate(size):
    old_break = sbrk(size)
    if old_break == error:
        return NULL
    return old_break
```

That's a **bump allocator**, an allocator that only advances a pointer. It never goes back. It doesn't need to remember which blocks are free because _no block is ever freed_. It's the simplest possible allocation strategy, and it's exactly what we're going to implement.

Why start with something so limited? Because it makes the fundamental mechanics visible without noise. The bump allocator shows you what `sbrk()` is doing, how memory is organized, and where your data lives. Those intuitions survive intact when, in later posts, we add complexity on top.


## Section 2: The Bump Allocator — Code {#section-2-the-bump-allocator-code}


### The heap_t Struct {#the-heap-t-struct}

All of our allocator's state lives in one struct:

```c
typedef struct {
    void  *start;       /* base address of the sbrk'd region           */
    void  *brk;         /* current program break (start + capacity)    */
    size_t capacity;    /* total bytes requested from the OS           */
    size_t used;        /* bytes allocated (bump pointer offset)       */
} heap_t;
```

Four fields. Each one is there for a concrete reason, and each one will pay dividends in future posts:

`start` is the base address `sbrk()` gave us at initialization. It never changes. It's the heap's "zero", all internal positions are calculated as offsets from here. In Post 7, when we have multiple arenas with `mmap()`, each arena will have its own `start`.

`brk` is the current program break. Always equal to `start + capacity`. We keep it explicit rather than computing it each time because, in future posts, `brk` will decouple from `start + capacity` when we have non-contiguous regions.

`capacity` is how many bytes we've requested from the OS _in total_. Not how many we've handed out — how many we have available. When a `heap_alloc()` doesn't fit, we request more with `sbrk()` and update `capacity`.

`used` is the bump pointer itself: how many bytes of `capacity` we've already handed out. Whenever someone requests memory, `used` advances. It never goes back. The current free position is always `start + used`.

Why a struct instead of four global variables? It's a design decision that seems unnecessary now, but avoids a full rewrite later. In Post 7, `heap_t` will evolve into an `allocator_t` that can manage multiple arenas, each with its own `start` and `capacity`. Starting with globals would make that transition painful. With a struct, the refactor will be mechanical.


### heap_init() {#heap-init}

```c
heap_t heap_init(size_t initial_size)
{
    heap_t h;
    h.start = sbrk(0);               /* where is the break right now? */
    if (h.start == (void *)-1) {
        h.start = NULL;  h.brk = NULL;
        h.capacity = 0;  h.used = 0;
        return h;
    }
    void *result = sbrk((intptr_t)initial_size);  /* move the break    */
    if (result == (void *)-1) {
        h.start = NULL;  h.brk = NULL;
        h.capacity = 0;  h.used = 0;
        return h;
    }
    h.brk      = (char *)h.start + initial_size;
    h.capacity = initial_size;
    h.used     = 0;
    return h;
}
```

The flow is straightforward. First we ask where the break is with `sbrk(0)`. If that fails (returns `(void *)-1`), the situation is catastrophic enough that we return a null heap, the caller must check it. Then we move the break `initial_size` bytes with `sbrk(initial_size)`. If _that_ fails, same treatment.

Why two calls instead of one? Because `sbrk(n)` returns the _previous_ break, not the new one. We need to know where our region _started_, not just that it grew. The first call with `sbrk(0)` gives us that base; the second does the real work.

Error handling here is deliberately simple. A production allocator would do more sophisticated things, `jemalloc`, for example, has multiple fallbacks. We return NULL and trust that the caller checks. It's a pedagogical hack, and we acknowledge it. In later posts, error handling will improve.


### heap_alloc() {#heap-alloc}

```c
void *heap_alloc(heap_t *h, size_t size)
{
    if (h->start == NULL) return NULL;
    if (size == 0) return NULL;
    /* Does it fit in current capacity? */
    if (h->used + size > h->capacity) {
        size_t grow = size;
        if (grow < h->capacity)
            grow = h->capacity;       /* double as a heuristic */
        void *result = sbrk((intptr_t)grow);
        if (result == (void *)-1)
            return NULL;
        h->capacity += grow;
        h->brk = (char *)h->start + h->capacity;
    }
    void *ptr = (char *)h->start + h->used;
    h->used += size;
    return ptr;
}
```

This is the central function, and it fits in 15 meaningful lines.

The first two guards are defensive: if the heap wasn't initialized correctly or if someone requests zero bytes, we return NULL without doing anything. The zero-byte case deserves a note, `malloc(0)` has _implementation-defined_ behavior per the C standard. It can return NULL or a unique pointer you must not dereference. We choose NULL for simplicity.

The growth block is interesting. If what's requested doesn't fit in the current capacity, we need more memory. The heuristic: request at least what we need, but if the current capacity is larger, double it. Doubling capacity is a classic amortized strategy, the same one `std::vector` uses in C++, and for the same reason: avoid `O(n)` calls to `sbrk()` for `n` allocations.

The final two lines are the _bump_: we compute the current address (`start + used`), advance `used`, and return the address. That's all. No metadata, no lists, no per-block bookkeeping. That simplicity is the point, and also the limitation future posts will resolve.

Note the complete absence of **alignment**. If you request 3 bytes followed by an `int*`, the `int*` will start at a misaligned address. On x86-64 this _works_ (misaligned accesses are legal but slow), but on ARM or RISC-V it can cause a trap. Post 2 will introduce block headers that enforce 8 or 16 byte alignment.


### heap_dump() {#heap-dump}

```c
void heap_dump(const heap_t *h)
{
    if (h->start == NULL) {
        printf("  [heap not initialized]\n");
        return;
    }
    double pct_used = (double)h->used / (double)h->capacity * 100.0;
    size_t free_bytes = h->capacity - h->used;
    const int bar_width = 50;
    int used_bar = (int)((double)h->used / (double)h->capacity * bar_width);
    int free_bar = bar_width - used_bar;
    printf("╔══════════════════════════════════════════════════════╗\n");
    printf("║       HEAP DUMP — Post 1 (Bump Allocator)          ║\n");
    printf("╠══════════════════════════════════════════════════════╣\n");
    printf("║ Heap base:     %p                    ║\n", h->start);
    printf("║ Current break: %p                    ║\n", h->brk);
    printf("║ Capacity:      %-6zu bytes                         ║\n",
           h->capacity);
    printf("║ Used:          %-6zu bytes (%5.1f%%)                ║\n",
           h->used, pct_used);
    printf("║ Free:          %-6zu bytes (%5.1f%%)                ║\n",
           free_bytes, 100.0 - pct_used);
    printf("╠══════════════════════════════════════════════════════╣\n");
    printf("║ [");
    for (int i = 0; i < used_bar; i++) printf("=");
    printf(" USED ][");
    for (int i = 0; i < free_bar; i++) printf("-");
    printf(" FREE ]║\n");
    printf("╚══════════════════════════════════════════════════════╝\n");
}
```

The dump function is longer than the other two combined, and that's intentional. Visualization is pedagogy, it's what transforms abstract numbers into spatial understanding.


## Section 3: Visualization — Understanding Your Heap {#section-3-visualization-understanding-your-heap}


### Example 1: Basic Allocations {#example-1-basic-allocations}

```c
heap_t h = heap_init(1024);
int  *nums = heap_alloc(&h, sizeof(int) * 10);   /* 40 bytes */
char *msg  = heap_alloc(&h, 128);
for (int i = 0; i < 10; i++)
    nums[i] = i * i;
strcpy(msg, "Hello from our custom heap!");
heap_dump(&h);
printf("nums[5] = %d\n", nums[5]);
printf("msg     = \"%s\"\n", msg);
```

Output (addresses will vary):

<pre style="width: fit-content; margin: 0 auto;">
╔═════════════════════════════════════════════════════╗
║       HEAP DUMP — Post 1 (Bump Allocator)           ║
╠═════════════════════════════════════════════════════╣
║ Heap base:     0x555555576000                       ║
║ Current break: 0x555555576400                       ║
║ Capacity:      1024   bytes                         ║
║ Used:          168    bytes ( 16.4%)                ║
║ Free:          856    bytes ( 83.6%)                ║
╠═════════════════════════════════════════════════════╣
║ [======== USED ][--------------------- FREE ]       ║
╚═════════════════════════════════════════════════════╝
nums[5] = 25
msg     = "Hello from our custom heap!"
</pre>

168 bytes used out of 1024 available. `nums` and `msg` live side by side with no gap: `nums` starts at offset 0 and occupies 40 bytes, `msg` starts at offset 40 and occupies 128 bytes. No padding, no headers, no holes. Raw, contiguous memory.


### Example 2: Multiple Mixed Allocations {#example-2-multiple-mixed-allocations}

```c
heap_t h = heap_init(1024);
void *a = heap_alloc(&h, 16);     /* offset 0   */
void *b = heap_alloc(&h, 32);     /* offset 16  */
void *c = heap_alloc(&h, 400);    /* offset 48  */
void *d = heap_alloc(&h, 50);     /* offset 448 */
heap_dump(&h);
```

The offsets are predictable: 0, 16, 48, 448. Each block starts exactly where the previous one ends. And here's the problem the dump implicitly reveals: **if we called `free(b)`** (the 32 bytes of the second block), how would we recover that space? The bump allocator has no way to know. Post 2 adds block headers to solve exactly this.


### Example 3: Automatic Growth {#example-3-automatic-growth}

```c
heap_t h = heap_init(100);     /* small heap */
printf("Initial capacity: %zu\n", h.capacity);
void *big = heap_alloc(&h, 500);  /* doesn't fit → grows */
heap_dump(&h);
```

The capacity jumps from 100 to 600. This demonstrates the growth heuristic: since 500 &gt; 100 (the current capacity), we request exactly 500 rather than doubling. If we had requested 80 bytes with a heap of 100, the capacity would have jumped to 200 (doubling).


## Section 4: Tests and Edge Cases {#section-4-tests-and-edge-cases}


### What Happens if You Allocate More Than the Capacity? {#what-happens-if-you-allocate-more-than-the-capacity}

The heap grows automatically. `heap_alloc()` calls `sbrk()` to request more bytes. This works because `sbrk()` extends the heap contiguously. Is there a limit? Yes. The kernel enforces a per-process limit (configurable with `ulimit -v`), and eventually virtual pages run out. When `sbrk()` fails, it returns `(void *)-1` and our `heap_alloc()` returns NULL.


### What Happens if You Allocate 0 Bytes? {#what-happens-if-you-allocate-0-bytes}

```c
void *zero = heap_alloc(&h, 0);
// zero == NULL
```

Our implementation returns NULL. The C standard says `malloc(0)` may return NULL or a unique non-dereferenceable pointer. Both are legal. We choose NULL because it's simpler and safer.


### What Happens When the Program Exits Without free? {#what-happens-when-the-program-exits-without-free}

Nothing bad. When a process terminates, the kernel reclaims _all_ of its virtual memory, heap, stack, everything. It doesn't matter whether you called `free()` or not. For short-lived programs (command-line utilities, scripts, tools), not freeing memory explicitly is a legitimate strategy. The problem appears in long-running programs (servers, daemons) where unfreed memory accumulates until resources are exhausted.


### Contiguous Allocation: An Invariant That Matters {#contiguous-allocation-an-invariant-that-matters}

```c
heap_t h = heap_init(1024);
char *a = heap_alloc(&h, 100);
char *b = heap_alloc(&h, 200);
char *c = heap_alloc(&h, 300);
printf("a-base: %zu\n", (size_t)(a - (char *)h.start));  /* 0   */
printf("b-base: %zu\n", (size_t)(b - (char *)h.start));  /* 100 */
printf("c-base: %zu\n", (size_t)(c - (char *)h.start));  /* 300 */
printf("b-a:    %zu\n", (size_t)(b - a));                 /* 100 */
printf("c-b:    %zu\n", (size_t)(c - b));                 /* 200 */
```

The pointer differences are _exactly_ the block sizes. This behavior will change in Post 2, where each block carries an 8–16 byte header.


## Section 5: Explicit Simplifications {#section-5-explicit-simplifications}

This allocator has serious limitations. We list them explicitly, because each one is a _Post 1 feature_ that becomes a _later-post fix_:

**No freeing.** The bump pointer only advances. In Post 3, we'll add a **free list**: a linked list of freed blocks that `heap_alloc()` consults before requesting more memory from the OS.

**No alignment.** A 3-byte alloc followed by a `double*` alloc produces a misaligned pointer. In Post 2, every allocation will be automatically aligned to 8 or 16 bytes.

**No block metadata.** The allocator doesn't know where each individual block starts or ends. In Post 2, each block will carry a **block header**: a small struct at the start storing the size, state (free/occupied), and a pointer to the next block.

**Contiguous heap.** `sbrk()` can only extend the heap upward. In Post 7, we'll migrate to `mmap()`, which can request memory regions anywhere in the address space.

**Single-threaded.** No locks, no thread-safety. In Post 7, we'll acknowledge this problem; the full solution (per-thread arenas) is out of scope for this series, but we'll point to how `jemalloc` and `mimalloc` solve it.

| Limitation      | State in Post 1  | Resolved in            |
|-----------------|------------------|------------------------|
| No freeing      | Advances only    | Post 3 (free list)     |
| No alignment    | Raw offsets      | Post 2 (block headers) |
| No metadata     | Anonymous blocks | Post 2 (block headers) |
| Contiguous heap | sbrk() only      | Post 7 (mmap + arenas) |
| Single-threaded | No locks         | Post 7+ (acknowledged) |


## Section 6: Where We're Going {#section-6-where-we-re-going}

In **Post 2**, we'll add **block headers**. Each allocation will carry a small struct at the start with the block size and status flags. This transforms the heap from an opaque blob into a sequence of self-describing blocks we can traverse, inspect, and eventually free.

In **Post 3**, we'll implement `heap_free()` and a **free list**. When a block is freed, it's marked free and inserted into a linked list. Future allocations search the free list first before requesting more memory from the OS.

In **Post 4**, we'll tackle **coalescing and splitting**: when two free blocks are adjacent, we merge them into one large block; when a free block is much larger than requested, we split it. These two operations are what separates a toy allocator from a functional one.

**Post 5** explores **allocation policies**, first-fit, best-fit, next-fit, and how each affects fragmentation and performance.

In **Post 7** comes the big transition: we replace `sbrk()` with `mmap()`, refactor `heap_t` into an `allocator_t` with multiple arenas, and lay the groundwork for thread-safety.

**Posts 8–12** build a **garbage collector** progressively: reference counting, mark-and-sweep, conservative GC, and incremental GC.

**Post 13** is the capstone: copying collector, generational GC, and references to production implementations (Go GC, jemalloc, mimalloc).


## Conclusion {#conclusion}

Today we built a working memory allocator in ~80 lines of C. It requests memory from the operating system with `sbrk()`, hands it out with a bump pointer, and visualizes it with an ASCII dump. It's incomplete, it can't free, it doesn't align, it carries no metadata, but those limitations are pedagogical features, not bugs.

What matters isn't the code itself, but what it reveals: that `malloc()` isn't magic. It's a struct, a couple of syscalls, and some bookkeeping. The C runtime you use every day does exactly this, more sophisticated, more optimized, more robust, but the fundamental mechanics are the same.

In Post 2, we take the next step: **block headers and alignment**. The heap stops being an opaque blob and becomes a sequence of self-describing blocks. It's the first step toward `free()`.


## Key Takeaways {#key-takeaways}

-   `sbrk()` is the syscall underlying `malloc()`: it moves the program break to request contiguous memory from the OS.
-   A bump allocator is the simplest possible strategy: a pointer that only advances. No freeing, no fragmentation, no complexity.
-   `heap_t` is an architectural decision: using a struct instead of globals prepares the refactor to arenas in Post 7.
-   The limitations are deliberate: each simplification (no freeing, no alignment, no metadata) is resolved in a specific later post.
-   Unfreed memory is reclaimed by the OS on exit: for short-lived programs, not freeing is a legitimate strategy; for servers, it's a memory leak.
