+++
title = "Hello, Array: malloc, free and Manual Bookkeeping"
author = ["pablo"]
date = 2026-05-24
draft = false
translationKey = "post-01-hello-array"
series = ["Dynamic Arrays in C"]
seriesPost = 1
+++

## The Probem No One Starts With {#the-probem-no-one-starts-with}

You have fve integers. You put them in an array:

```c
int numbers[5] = { 10, 20, 30, 40, 50 };
```

Done. C gives you a contiguos chunk of 20 bytes on the stack indexed from 0 to 4, and life is good.

Now your user wants to add a sixth integer. What do you do?

You can't resize a stack array. Its size was baked into the binary at compile time, the compiler saw `5`, calculated 20 bytes, and that's the space your function's stack frame has. There's no negotiation. You could declare `int numbers[1000]` and hope it's big enough, but hope is not a memory management strategy.

You could use a variable-length array (`int numbers[n]`), but that just shifts the problem: n is still fixed once you enter the function. Worse, VLAs live on the stack, which is limited to a few megabytes. Store a million integers and you blow the stack with no graceful recovery.

The real solution lives on the heap. `malloc` lets you ask the operating system for a chunk of memory at runtime, any size you want, limited only by available RAM. But malloc gives you raw bytes and a pointer. No size tracking. No bounds checking. No "how full am I?" bookkeeping. You get the memory and the responsibility.

This is where every dynamic array begins: not with a clever data structure, but with a basic question of bookkeeping. Who tracks how many elements you've stored? Who tracks how many you could store? Who makes sure the memory gets freed when you're done?

In C, the answer is always the same: you do.

This post builds the simplest possible dynamic array, one that holds integers, has a fixed capacity, and does exactly three things: create, push, and destroy. No automatic growth (that's Post 2), no generics (Post 4), no error recovery (Post 6). Just the raw skeleton that everything else builds on.

By the end, you'll understand two things most C tutorials skip: why the metadata struct exists, and why the order in which you call `free` matters.

Let's allocate some memory.


## The Struct: What an Array Knows About Itself {#the-struct-what-an-array-knows-about-itself}

A raw `malloc` call returns `void *`, a pointer to bytes with no meaning attached. If you allocate space for 10 integers, nobody remembers that number except you. The instant you lose track of the capacity, you're writing bugs.

So the first thing a dynamic array needs isn't data. It's _metadata_: a small struct that sits alongside the data and remembers the bookkeeping details.

```c
typedef struct {
    int    *data;       /* Pointer to the heap buffer holding elements  */
    size_t  size;       /* How many elements have been stored           */
    size_t  capacity;   /* How many elements the buffer can hold        */
} IntArray;
```

Three fields. This is the minimum viable bookkeeping.

`data` is a pointer to the actual heap allocation where elements live. It's the result of a `malloc(capacity * sizeof(int))` call. When you access `arr->data[3]`, you're reading the fourth integer in that allocation.

`size` tracks how many elements the user has actually pushed. It starts at 0 and increments with every `array_push`. It is not the same as capacity, this distinction is the single most important concept in dynamic array design.

`capacity` tracks how many elements the allocation can _hold_. If you malloced space for 10 integers, capacity is 10, even if size is only 3. The gap between size and capacity is wasted memory, we allocated it but aren't using it yet. Managing that gap is the art of dynamic arrays.

Think of it like a parking garage. `capacity` is the number of parking spots. `size` is the number of cars currently parked. The garage exists at a specific address (`data`). You can have an empty garage (size=0, capacity=100) or a full one (size=100, capacity=100), but you can never park more cars than spots, unless you build a bigger garage.


## The Full Code {#the-full-code}

Here is the complete, standalone source file. It compiles with zero warnings under `gcc -Wall -Wextra -Wpedantic -std=c11`, runs without leaks, and generates both ASCII visualization to stdout and a Graphviz DOT file.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

/* --- The struct ---------------------------------------------------------- */

typedef struct {
    int    *data;
    size_t  size;
    size_t  capacity;
} IntArray;

/* --- Lifecycle ----------------------------------------------------------- */

IntArray *array_create(size_t capacity)
{
    if (capacity == 0) {
        fprintf(stderr, "array_create: capacity must be > 0\n");
        return NULL;
    }

    IntArray *arr = malloc(sizeof(IntArray));
    if (!arr) {
        fprintf(stderr, "array_create: failed to allocate struct\n");
        return NULL;
    }

    arr->data = malloc(capacity * sizeof(int));
    if (!arr->data) {
        fprintf(stderr, "array_create: failed to allocate buffer\n");
        free(arr);
        return NULL;
    }

    arr->size     = 0;
    arr->capacity = capacity;
    return arr;
}

void array_destroy(IntArray *arr)
{
    if (!arr) return;
    free(arr->data);
    arr->data = NULL;
    free(arr);
}

/* --- Operations ---------------------------------------------------------- */

int array_push(IntArray *arr, int value)
{
    if (!arr) return -1;
    if (arr->size >= arr->capacity) {
        fprintf(stderr,
                "array_push: full (size=%zu, capacity=%zu)\n",
                arr->size, arr->capacity);
        return -1;
    }

    arr->data[arr->size] = value;
    arr->size++;
    return 0;
}

int array_get(const IntArray *arr, size_t index, int *out)
{
    if (!arr || index >= arr->size) return -1;
    *out = arr->data[index];
    return 0;
}

size_t array_size(const IntArray *arr)     { return arr ? arr->size     : 0; }
size_t array_capacity(const IntArray *arr) { return arr ? arr->capacity : 0; }
```


### The Visualization Functions {#the-visualization-functions}

The ASCII visualizer prints a complete snapshop of the array's state, metadata memory layout with box drawing, index labels and utilizacion statistics:

```c
void array_visualize_ascii(const IntArray *arr, const char *label)
{
    if (!arr) { printf("(NULL array)\n"); return; }

    size_t cap = arr->capacity;
    size_t sz  = arr->size;

    printf("\n╔══════════════════════════════════════════════════════╗\n");
    printf("║  %-50s  ║\n", label ? label : "ARRAY STATE");
    printf("╠══════════════════════════════════════════════════════╣\n");
    printf("║  size = %-5zu  capacity = %-5zu  elem = %zu bytes    ║\n",
           sz, cap, sizeof(int));
    printf("║  data = %-14p  (heap)                   ║\n",
           (void *)arr->data);
    printf("╠══════════════════════════════════════════════════════╣\n");

    size_t show = cap <= 16 ? cap : 16;

    /* Top border */
    printf("║  ");
    for (size_t i = 0; i < show; i++) printf("┌──────");
    printf("┐  ║\n");

    /* Values */
    printf("║  ");
    for (size_t i = 0; i < show; i++) {
        if (i < sz) printf("│%5d ", arr->data[i]);
        else        printf("│  ·   ");
    }
    printf("│  ║\n");

    /* Bottom border */
    printf("║  ");
    for (size_t i = 0; i < show; i++) printf("└──────");
    printf("┘  ║\n");

    /* Index labels */
    printf("║  ");
    for (size_t i = 0; i < show; i++) printf(" %3zu   ", i);
    printf("   ║\n");

    /* Stats */
    size_t used_bytes  = sz  * sizeof(int);
    size_t alloc_bytes = cap * sizeof(int);
    printf("╠══════════════════════════════════════════════════════╣\n");
    printf("║  %zuB used / %zuB allocated = %.1f%% utilization\n",
           used_bytes, alloc_bytes,
           cap > 0 ? 100.0 * (double)sz / (double)cap : 0.0);
    printf("╚══════════════════════════════════════════════════════╝\n\n");
}
```

The DOT generator writes a Graphviz file showing the structural relationship between the metadata struct and the heap buffer:

```c
void array_generate_dot(const IntArray *arr, const char *filename)
{
    FILE *f = fopen(filename, "w");
    if (!f) return;

    fprintf(f, "digraph IntArray {\n");
    fprintf(f, "  rankdir=LR;\n");

    /* Metadata node */
    fprintf(f, "  metadata [shape=record, style=filled, "
               "fillcolor=\"#FFF3CD\",\n");
    fprintf(f, "    label=\"{IntArray|data: %p|size: %zu|"
               "capacity: %zu}\"];\n",
            (void *)arr->data, arr->size, arr->capacity);

    /* Buffer node */
    fprintf(f, "  buffer [shape=record, style=filled, "
               "fillcolor=\"#D1ECF1\",\n");
    fprintf(f, "    label=\"{Heap Buffer (%zu bytes)|{",
            arr->capacity * sizeof(int));
    for (size_t i = 0; i < arr->capacity; i++) {
        if (i > 0) fprintf(f, "|");
        if (i < arr->size)
            fprintf(f, "[%zu]=%d", i, arr->data[i]);
        else
            fprintf(f, "[%zu]=·", i);
    }
    fprintf(f, "}}\"];\n");

    fprintf(f, "  metadata -> buffer [label=\"owns (heap)\"];\n");
    fprintf(f, "}\n");
    fclose(f);
}
```


### The Main Function {#the-main-function}

The `main()` drives the demonstration: create an array, push elements one by one (visualizing after each push), fill it to capacity, try to exceed it, generate the DOT file, and destroy:

```c
int main(void)
{
    IntArray *arr = array_create(5);

    /* Push values one at a time, visualize after each */
    int values[] = {10, 20, 30, 40, 50};
    for (int i = 0; i < 5; i++) {
        array_push(arr, values[i]);
        array_visualize_ascii(arr, "After push");
    }

    /* This will fail — array is full */
    int rc = array_push(arr, 60);  /* returns -1 */

    /* Generate Graphviz and clean up */
    array_generate_dot(arr, "outputs/post_01_state.dot");
    array_destroy(arr);
    return 0;
}
```


## Walking Through the Code {#walking-through-the-code}


### Two Allocations, Two Frees {#two-allocations-two-frees}

The most important pattern in this entire file is the symmetry between `array_create` and `array_destroy`.

`array_create` performs two allocations

```text
malloc(sizeof(IntArray))        -> the metadata struct
malloc(capacity * sizeof(int))  -> the element buffer
```

`array_destroy` performs two frees, in reverse order:

```text
free(arr->data)   -> element buffer first
free(arr)         -> metadata struct second
```

The order is not negotiable. If you reverse it, `free(arr)` first, then `free(arr->data)`, you're dereferencing `arr` after freeing it. The memory at `arr` has been returned to the allocator. Reading `arr->data` at that point is undefined behavior: it might work, it might crash, it might silently corrupt your heap. Valgrind would flag it as an invalid read.

The defensive `arr->data = NULL` after the first free is optional but cheap. It ensures that if anything accidentally touches the struct between the two frees (in more complex code with callbacks or error handling), the invalid access produces a NULL dereference, a loud, debuggable crash, instead of a silent use-after-free.


### Why Push Checks `size >= capacity` {#why-push-checks-size-capacity}

The push function's guard clause is simple:

```c
if (arr->size >= arr->capacity) return -1;
```

This is the bookkeeping paying for itself. Without the `capacity` field, you'd have no way to know whether `arr->data[arr->size]` is a valid write or a buffer overflow. The check costs one comparison per push, a few nanoseconds, and prevents the most common class of heap corruption bugs.

Notice that push doesn't grow the array when it's full. It just refuses and returns -1. This is a deliberate design choice for Post 1: we're building understanding of the static case first. Post 2 introduces `realloc` and automatic growth. If you're itching to grow the array, that frustration is pedagogically intentional, you're feeling the same pain that motivates dynamic growth.


### The Visualization as a Debugging Tool {#the-visualization-as-a-debugging-tool}

`array_visualize_ascii` isn't just a pretty-printer for the blog. It's a debugging tool you can embed in any project. Every time the array's state changes, you can print it:

<pre style="width: fit-content; margin: 0 auto;">
╔════════════════════════════════════════════════════╗
║  After push(30)                                    ║
╠════════════════════════════════════════════════════╣
║  size = 3      capacity = 5      elem = 4 bytes    ║
╠════════════════════════════════════════════════════╣
║  ┌──────┌──────┌──────┌──────┌──────┐              ║
║  │   10 │   20 │   30 │  ·   │  ·   │              ║
║  └──────└──────└──────└──────└──────┘              ║
║     0      1      2      3      4                  ║
╚════════════════════════════════════════════════════╝
</pre>

Three integers stored, two slots still empty (shown as `·`). The metadata at the top tells you everything: size, capacity, pointer address, element size. The stats at the bottom quantify waste. You can pipe the output to a file (`./post_01 > trace.txt`) and scroll through the entire lifecycle of your array.

This is something most tutorials don't give you: the ability to see memory. We'll build on this visualization in every post, adding Graphviz diagrams for pointer relationships, and eventually an interactive HTML dashboard for performance analysis.


## Key Concepts and Tradeoffs {#key-concepts-and-tradeoffs}


### Stack vs Heap for the Metadata Struct {#stack-vs-heap-for-the-metadata-struct}

Our `array_create` allocates the IntArray struct on the heap:

```c
IntArray *arr = malloc(sizeof(IntArray));
```

Why not put it on the stack? You could write:

```c
IntArray array_create_stack(size_t capacity) {
    IntArray arr;
    arr.data = malloc(capacity * sizeof(int));
    arr.size = 0;
    arr.capacity = capacity;
    return arr;  /* returns a copy */
}
```

This works, but it changes the API contract in subtle ways. The caller receives a copy of the struct. If they pass it to a function that modifies it (say, `push`), they need to pass a pointer to their local copy. And they can't return it from a function that creates it conditionally, because the stack frame disappears. Heap allocation gives you a stable pointer that survives function boundaries and can be stored in other data structures. The tradeoff is that you now need to explicitly free it, heap memory never cleans itself up.

For a library API, heap allocation is the standard choice. It gives callers a single pointer to manage, and the destroy function handles the internal cleanup. You'll see this pattern in virtually every C library: `thing_create()` returns a pointer, `thing_destroy()` frees it.


### Memory Waste: The Capacity Problem {#memory-waste-the-capacity-problem}

When you create an array with capacity 10 and store 3 elements, you're wasting 7 slots × 4 bytes = 28 bytes. That's a 70% waste ratio. Is this bad?

It depends. For small arrays, the waste is negligible, 28 bytes is nothing on a modern machine with gigabytes of RAM. For millions of small arrays, it adds up. For one large array, the waste percentage drops as you fill it.

The real question is: what capacity should you start with? If you pick too small (capacity=1), you'll need to reallocate constantly as the array grows (Post 2). If you pick too large (capacity=10000), you waste memory on arrays that only hold 5 elements. There's no universal answer, it depends on your usage pattern. This is why production dynamic arrays let you specify an initial capacity hint.


### Why Return Codes, Not Assertions {#why-return-codes-not-assertions}

`array_push` returns 0 on success and -1 on failure. It doesn't abort the program. This is a conscious design decision: the caller should decide what to do when a push fails. Maybe the caller wants to log and continue. Maybe they want to resize and retry. Maybe they want to exit. By returning an error code, we give the caller that choice.

The alternative, `assert(arr->size < arr->capacity)`, kills the program with no recovery. That's appropriate for programmer errors (like passing NULL where you shouldn't), but not for expected conditions like "the array is full." We'll discuss error handling strategies in depth in Post 6.


## Try This and Watch It Fall {#try-this-and-watch-it-fall}

Before moving on, try these experiments with the code:

**Experiment 1: The Memory Leak**. In `array_destroy`, comment out `free(arr->data)`. Compile and run under valgrind (`valgrind --leak-check=full ./post_01`). You'll see "definitely lost: 20 bytes", that's the orphaned buffer. The struct was freed, but the buffer it pointed to was not. This is exactly the bug the knowledge test asks about.

**Experiment 2: Use After Free**. In `main`, add `printf("%d\n", arr->data[0])`; after `array_destroy(arr)`. Compile and run. It might print `10`. It might print garbage. It might crash. That's undefined behavior, the data is freed, but the memory hasn't necessarily been overwritten yet. Valgrind would flag this as "Invalid read of size 4."

**Experiment 3: Buffer Overflow**. Change the push guard to always succeed (remove the capacity check). Push 100 elements into a capacity-5 array. Run under AddressSanitizer (`gcc -fsanitize=address`). Watch it detect the heap-buffer-overflow.


## Knowledge Test {#knowledge-test}

> What happens if you call `free(arr)` but forget to call `free(arr->data)` first?

The struct is returned to the heap allocator, but the buffer it pointed to, the `capacity * sizeof(int)` bytes at `arr->data`, remains allocated. No pointer to it exists anymore (the struct that held the pointer is freed), so the memory is leaked. It will never be freed for the rest of the program's lifetime. On a long-running program, repeated leaks like this accumulate and eventually exhaust memory. Valgrind would report "definitely lost: N bytes in 1 blocks."


## What's Next {#what-s-next}

Our array works, but it has a crippling limitation: when it's full, push fails. The user has to guess the right capacity upfront, and if they guess wrong, they're stuck.

In **Post 2: "Growing Pains: realloc and Automatic Capacity Management"**, we remove this limitation. We'll introduce `realloc`, the syscall that says "give me more space, and copy my data to the new location if needed." You'll learn why old pointers become invalid after a realloc, why the growth factor matters (spoiler: it determines your amortized cost), and why you must never write `arr->data = realloc(arr->data, new_size)` directly.

The fixed-capacity array you built today is the foundation. Everything from here builds on it.
