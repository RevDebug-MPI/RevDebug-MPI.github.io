---
layout: page
title: Architecture & Features
permalink: /architecture.html
---

# System Architecture & New Features

To achieve high responsiveness and scale to memory-intensive parallel workloads, **RevDebug-MPI** transitions from a
monolithic checkpointing model to a hierarchical, delta-based state tracking architecture.

---

## High-Level Execution Model

RevDebug-MPI operates through a coordinated client-agent topology:

1. **Instrumentation (`bin/compiler`):** Target MPI programs are compiled using our wrapper script around `mpicc` (
   backed by GCC 8.5.0), ensuring consistent DWARF level 4 debug metadata and interceptable MPI communication hooks.
2. **Central Orchestration (`bin/orchestrator`):** The orchestrator launches the MPI job across the specified number of
   processes, initializes an RPC server (default port `3490`), and starts a WebSocket server (port `3496`) to serve the
   interactive Web GUI.
3. **Per-Rank Debug Agents (`bin/node-debugger`):** Each MPI rank is paired with a dedicated debug agent listening on
   its own local RPC port (`3500`, `3501`, etc.). These agents execute fine-grained debugger commands (such as
   single-stepping, breakpoint management, call stack inspection, and variable evaluation) and coordinate with the
   underlying checkpointing backend.

---

## 1. Multi-Level Checkpointing

### The I/O Bottleneck in Distributed Debugging

In previous implementations, establishing a restoration point required all MPI ranks to pause synchronously while their
complete memory states were serialized and written directly to persistent disk storage or a Network File System (NFS).
Because physical disk write speeds are orders of magnitude slower than CPU and RAM operations, this blocking I/O
introduced severe latency. For applications managing gigabytes of state across multiple ranks, checkpointing interrupted
execution flow for several seconds or minutes per step, rendering interactive debugging nearly impossible.

RevDebug-MPI resolves this bottleneck through **Multi-Level Checkpointing**, utilizing a novel two-tier storage
hierarchy:

### Tier 1: In-Memory State Capture (Level 1)

Upon reaching a checkpoint barrier (either automatically or via the `cp` command), the central orchestrator directs the
per-node debug agents to write their process memory images directly to a local, in-memory `tmpfs` filesystem (
`/dev/shm/rev_mpi_deb_l1/`).

* Because volatile memory write throughput is vastly superior to physical disk I/O, process freeze times are reduced to
  a fraction of a second.
* The MPI application resumes execution—or returns control to the interactive debugger prompt—almost instantaneously,
  preserving the illusion of uninterrupted stepping for the developer.

### Tier 2: Asynchronous Persistent Migration (Level 2)

To prevent the volatile RAM-disk from rapidly overflowing and triggering Linux Out-Of-Memory (OOM) killer mechanisms
during extended debugging sessions, RevDebug-MPI incorporates a dedicated background worker daemon.

* Immediately after an L1 snapshot is finalized, the background worker asynchronously transfers the serialized snapshot
  images from the volatile L1 RAM-disk (`/dev/shm/rev_mpi_deb_l1/`) to permanent secondary storage (`bin/temp/` in Level
  2).
* This migration executes concurrently in a separate background thread, utilizing idle I/O cycles without blocking the
  active MPI processes, the CLI prompt, or the Web GUI.

### Automated Retention Policy and Seamless Fallback

An automated retention policy manages the L1 cache capacity by acting as a bounded ring buffer:

* It retains only the most recent snapshots (e.g., the last two or three stepping states) in volatile memory.
* Older snapshots are safely evicted from RAM only after their successful migration and integrity verification in L2
  storage are confirmed.
* **Reverse-Stepping Priority:** During a reverse-stepping (`rollback`) operation, the orchestrator prioritizes L1 for
  immediate, ultra-low-latency restoration. If the developer wishes to jump back to a deeper historical state that has
  already been evicted from RAM, the debugger transparently falls back to L2, retrieves the persisted snapshot, and
  orchestrates the process restoration seamlessly.

---

## 2. Incremental Checkpointing

### The Memory Footprint Challenge

Capturing complete process memory footprints at every debugging step—even when initially written to a fast
RAM-disk—rapidly consumes storage capacity and increases the data transfer overhead between L1 and L2. For a parallel
application utilizing 4 GB of RAM across 16 nodes, taking just ten full steps would generate hundreds of gigabytes of
redundant snapshot data. To mitigate this, RevDebug-MPI integrates **Incremental Checkpointing**.

### Kernel-Assisted Delta Tracking

RevDebug-MPI leverages the Checkpoint/Restore In Userspace (CRIU) framework's kernel-assisted memory tracking
mechanisms:

* **Baseline Capture:** The first checkpoint in a debugging chain establishes a full baseline image containing all
  process memory mappings, shared libraries, heaps, and execution stacks.
* **Soft-Dirty Bit Inspection:** For all subsequent checkpoints, RevDebug-MPI instructs the Linux kernel to reset and
  monitor the page-table `soft-dirty` bits for each MPI rank. When the next checkpoint is triggered, the tool bypasses
  read-only code segments and unmodified heap regions, dumping **only the specific memory pages modified** since the
  preceding snapshot.

### Hierarchical Parent Linking and Restoration

* Each incremental snapshot (delta) maintains a symbolic relative link to its parent checkpoint directory.
* By preserving these hierarchical parent links during the asynchronous L1-to-L2 migration, the overall storage
  footprint is kept tiny.
* When a restore command is issued by the developer, the orchestrator sequentially reconstructs the exact process memory
  state by re-applying the memory deltas on top of the baseline image. This drastically reduces the volume of data that
  must be serialized in L1, migrated to L2, and read back during reverse execution.

---

## 3. Interchangeable Checkpointing Backends

Recognizing that high-performance computing clusters operate under diverse security policies and administrative
constraints, RevDebug-MPI maintains modular support for two interchangeable checkpointing backends:

| Feature / Aspect              | CRIU (Recommended Default)                                                                                                                | DMTCP (User-Space Alternative)                                                                                     |
|:------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------|
| **Full Name**                 | Checkpoint/Restore In Userspace                                                                                                           | Distributed MultiThreaded Checkpointing                                                                            |
| **Tested Version**            | `3.19`                                                                                                                                    | `2.6+ / 3.0`                                                                                                       |
| **Operating Mechanism**       | Kernel-level `ptrace`, `/proc` inspection, and userspace dump                                                                             | Dynamic library injection (`LD_PRELOAD`) & user-space coordinator                                                  |
| **Root Privileges Required?** | **Yes** (`sudo` or `CAP_SYS_ADMIN` container capabilities)                                                                                | **No** (Runs 100% unprivileged in user space)                                                                      |
| **Incremental Checkpointing** | **Supported** (via Linux kernel `soft-dirty` page tracking)                                                                               | Not natively supported (full snapshots only)                                                                       |
| **Multi-Level L1/L2 Support** | **Supported**                                                                                                                             | **Supported**                                                                                                      |
| **Primary Use Case**          | Local workstations, dedicated nodes, Docker/Singularity containers, and VMs where maximum speed and minimum storage overhead are desired. | Shared multi-tenant supercomputers and restricted HPC clusters where escalating privileges is strictly prohibited. |