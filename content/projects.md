---
title: "Projects"
date: 2025-01-01
layout: "single"
ShowToc: true
TocOpen: true
ShowReadingTime: false
---

Active projects I'm building or maintaining, mostly in C.

---

## Zinc

A minimal, statically-typed language targeting a safe subset of C semantics.

Current state: interpreter + bytecode VM. Compiler in progress.

Goals:
- No undefined behavior by construction
- Explicit ownership without a runtime GC
- Readable output C as a compilation target (initially)
- Written entirely in C99, no dependencies

**Status:** Active — posts on this blog track the development.  
**Repo:** [github.com/ansuzgs/zinc](https://github.com/ansuzgs/zinc)

---
<!---->
<!-- ## liblog -->
<!---->
<!-- A structured logging library for C, designed as a vehicle for exploring every major -->
<!-- C design pattern: vtable-based polymorphism, arena allocators, tagged unions, and -->
<!-- compile-time configuration via macros. -->
<!---->
<!-- Features: -->
<!-- - Multiple backends (stderr, file, ring buffer) -->
<!-- - Log levels, filters, and per-module configuration -->
<!-- - Zero dynamic allocation in the hot path (arena-backed) -->
<!-- - Single-header option (`liblog.h`) -->
<!---->
<!-- **Status:** Active   -->
<!-- **Repo:** [github.com/YOUR_HANDLE/liblog](https://github.com/YOUR_HANDLE/liblog) -->
<!---->
<!-- --- -->

## bitstream.h

A single-header C library for reading and writing packed bitstreams. Useful for
protocol parsers, compression codecs, and anything that deals with sub-byte data.

**Status:** Stable / maintenance mode  
**Repo:** [github.com/ansuzgs/bitstream.h](https://github.com/ansuzgs/bitstream.h)

---

## nz

A terminal-first Markdown note-taking CLI. Written in Bash.

Not a C project, but it lives in the same workflow.

<!-- **Repo:** [github.com/YO/nz](https://github.com/YOUR_HANDLE/nz) -->

