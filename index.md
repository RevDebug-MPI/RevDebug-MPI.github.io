---
layout: page
title: Overview
permalink: /
---

# RevDebug-MPI: Causally Consistent Reversible Debugger for MPI

**RevDebug-MPI** is a scalable, distributed reversible debugging tool specifically designed for Message Passing
Interface (MPI) applications operating in high-performance computing (HPC) environments. By leveraging coordinated
checkpoint-restore mechanisms alongside causal message tracking, RevDebug-MPI enables developers to deterministically
step backwards through parallel execution flows to isolate elusive concurrency bugs, non-deterministic race conditions,
and synchronization deadlocks.

> ⚠️ **Architecture Notice:** Only `x86_64` is currently supported.  
> When using the **CRIU** backend, root privileges are required; running inside a **container** or **virtual machine**
> is strongly recommended.  
> The **DMTCP** backend operates entirely in user space without root privileges.

---

## The Challenge of Parallel Debugging in HPC

As high-performance computing environments scale to thousands of interconnected nodes, the complexity of developing,
maintaining, and debugging reliable parallel applications grows exponentially. While MPI remains the de facto standard
communication framework for distributed scientific workflows and large-scale simulations, its highly concurrent nature
makes software execution inherently difficult to trace.

Traditional, forward-only debugging tools are notoriously inefficient for MPI workloads. In a distributed environment,
the mere act of attaching a standard forward debugger alters the timing of message deliveries, frequently masking the
very concurrency bugs developers are trying to resolve—a phenomenon widely known as **"Heisenbugs"**.

**Reversible debugging** resolves this fundamental limitation by enabling deterministic backwards execution. By allowing
developers to safely rewind the state of the entire distributed application from the moment a crash or erroneous state
is observed, RevDebug-MPI provides a direct, deterministic path to pinpointing the root causes of parallel execution
errors.

---

## What's New in the Latest Release?

Earlier iterations of our debugger established a viable foundation for reversible parallel debugging using
checkpoint-restore mechanisms and interchangeable backends (CRIU and DMTCP). However, applying traditional monolithic
checkpointing to long-running, state-heavy HPC workloads exposed two major scaling bottlenecks:

1. **"Stop-the-World" I/O Latency:** Pausing all MPI ranks globally to write full memory dumps directly to disk or
   Network File Systems (NFS) freezes interactive debugging sessions for seconds or minutes.
2. **Massive Storage Footprints:** Capturing complete process memory maps at every stepping interval rapidly exhausts
   disk and memory quotas.

Our latest architectural release resolves both bottlenecks through two synergistic advancements:

* 🚀 **[Multi-Level Checkpointing](architecture.html#1-multi-level-checkpointing):** Decouples immediate state capture
  from persistent disk I/O using a two-tier storage hierarchy. Process images are written instantaneously to a volatile
  Level 1 in-memory RAM-disk (`/dev/shm`), returning control to the developer in milliseconds while a dedicated
  background daemon asynchronously migrates snapshots to Level 2 persistent storage.
* 📉 **[Incremental Checkpointing](architecture.html#2-incremental-checkpointing):** Integrates kernel-assisted delta
  tracking via Linux `soft-dirty` memory bits. Following an initial baseline snapshot, subsequent checkpoints record
  only the specific memory pages modified during a stepping interval, achieving up to a **69% reduction in payload size
  **.
* 🔄 **[Modular Backend Engine](architecture.html#3-interchangeable-checkpointing-backends):** Supports both **CRIU** (
  kernel-assisted, ultra-fast with incremental support) and **DMTCP** (unprivileged user-space operation for restricted
  HPC clusters).
* 🖥️ **Dual Interface (CLI & Web GUI):** Provides both a low-overhead command-line orchestrator console and an
  interactive, WebSocket-driven Web GUI for visual inspection of distributed ranks.

---

## Core System Components

RevDebug-MPI compiles into three specialized binaries that coordinate distributed execution and state management:

| Binary              | Role & Description                                                                                                                 |
|:--------------------|:-----------------------------------------------------------------------------------------------------------------------------------|
| `bin/orchestrator`  | Central debugger process manager, CLI/WebSocket server, and L1-to-L2 tier migration controller.                                    |
| `bin/compiler`      | MPI wrapper compiler that instruments target C/C++ MPI programs with DWARF level 4 debug metadata.                                 |
| `bin/node-debugger` | Per-process debug agent attached to each individual MPI rank to control stepping, variable inspection, and local checkpoint dumps. |

---

## Quick Navigation

* 🏗️ **[Architecture & New Features](architecture.html):** Deep dive into Multi-Level Checkpointing, Incremental
  Checkpointing, DWARF instrumentation, and backend internals.
* 🛠️ **[Installation & Usage Guide](installation.html):** Complete step-by-step instructions for building the toolchain
  from source (GCC 8.5.0, MPICH 3.4.3, Go 1.21, CRIU 3.19, DMTCP), compiling target programs, and launching the CLI and
  Web GUI.
* 💻 **[Tool Demonstration & Walkthrough](demo.html):** Comprehensive empirical walkthrough using the `circle.c` MPI
  benchmark, complete with terminal logs and storage footprint analysis.
* 📚 **[Publications & Team](publications.html):** Academic papers, BibTeX citations, and research team details.