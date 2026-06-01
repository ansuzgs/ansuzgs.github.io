---
title: "About"
date: 2026-05-22
layout: "single"
ShowToc: false
ShowReadingTime: false
description: "Pablo González, telecommunications engineer and PhD, writing about low-level C programming, allocators, data structures, and language design."
---

I'm Pablo, telecommunications engineer with a PhD in electromagnetic simulation,
currently working in systems engineering for the defense and space sector (satellite
infrastructure and mission-critical systems).

I got into low-level programming because I wanted to understand what was actually
happening. Not the abstraction, not the API, the thing itself. How memory gets
handed out. How a dynamic array decides to grow. What `free()` actually does with
those bytes. This blog is where I work through those questions by building the
answers from scratch, in C.

## What I write about

Allocators, data structures, OOP patterns in C, and a small language called
**Zinc** that I'm building from the ground up (interpreter, bytecode VM, and
eventually a compiler) all in C99. Two ongoing series right now: one building a
[memory allocator and garbage collector](/series/memory-allocation-and-gc-from-scratch/),
another implementing
[dynamic arrays](/series/dynamic-arrays-in-c/) with all the edge cases most
tutorials skip.

## Projects

- **[Zinc](https://github.com/ansuzgs/zinc)**: a minimal, statically-typed
  language targeting a safe subset of C semantics. Explicit ownership, no runtime
  GC, readable C output as compilation target.
- **[bitstream.h](https://github.com/ansuzgs/bitstream.h)**: single-header C
  library for reading and writing packed bitstreams. Useful for protocol parsers
  and compression codecs.
- **[dynamic-arrays-c](https://github.com/ansuzgs/dynamic-arrays-c)**: companion
  repo for the Dynamic Arrays series. Every post compiles and runs.

## The code

Everything on this blog compiles and is tested. If it doesn't, that's a bug, open
an issue on [GitHub](https://github.com/ansuzgs).

## Elsewhere

When I'm not writing C I'm usually on a road bike, on a climbing wall, or somewhere
in the mountains with a camera.

GitHub: [@ansuzgs](https://github.com/ansuzgs)
LinkedIn: [pablogonzalez21191](https://www.linkedin.com/in/pablogonzalez21191/)
