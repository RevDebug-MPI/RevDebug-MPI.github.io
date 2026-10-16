---
layout: page
title: Installation
permalink: /installation.html
---

# Installation, Toolchain Build & Usage Guide

This guide provides complete instructions for building the required toolchain from source, compiling **RevDebug-MPI**,
and running interactive debugging sessions via the CLI and Web GUI.

---

## 1. System Requirements

RevDebug-MPI has been extensively tested with the specific toolchain versions listed below.

> ⚠️ **Important Note on Compiler Versions (Ubuntu 24.04+):**  
> Using newer compiler versions (such as GCC 11+ or GCC 13 shipped by default on Ubuntu 24.04) will cause compatibility
> issues with the debugger's DWARF metadata parser. GCC 11+ generates DWARF level 5 by default, whereas RevDebug-MPI
> requires **DWARF level 4** debug data produced by **GCC 8.5.0**. Furthermore, Ubuntu 24.04 does not include CRIU in its
> default apt repositories. Follow the source-build instructions below to set up a clean, compatible environment.

| Tool             | Required Version | Purpose & Compatibility Reason                                                 |
|:-----------------|:-----------------|:-------------------------------------------------------------------------------|
| **Architecture** | `x86_64`         | Currently supported target CPU architecture                                    |
| **GCC**          | `8.5.0`          | Must produce DWARF level 4 debug metadata for the binary parser                |
| **MPICH**        | `3.4.3`          | Tested MPI implementation (must be compiled with GCC 8.5.0)                    |
| **Go**           | `1.21`           | Required for compiling the Go-based orchestrator and debug agents              |
| **CRIU**         | `3.19`           | High-performance kernel-assisted checkpointing backend (Default / Recommended) |
| **DMTCP**        | `2.6+ / 3.0`     | User-space checkpointing backend (Alternative, runs without root)              |

---

## 2. Building the Toolchain from Source

First, install the base system dependencies required to compile all toolchain components:

```bash
sudo apt-get update
sudo apt-get install -y build-essential wget curl libgmp-dev libmpfr-dev \
    libmpc-dev flex bison pkg-config libprotobuf-dev libprotobuf-c-dev \
    protobuf-c-compiler protobuf-compiler python3-protobuf libnl-3-dev \
    libnet-dev libcap-dev bsdmainutils asciidoc npm git

```

---

### Step 2.1: Build GCC 8.5.0

The debugger's DWARF parser requires executables compiled with DWARF level 4. Build GCC 8.5.0 into
`/usr/local/gcc-8.5.0`:

```bash
mkdir -p ~/toolchain/gcc-build && cd ~/toolchain/gcc-build

# Download source
wget [https://ftpmirror.gnu.org/gcc/gcc-8.5.0/gcc-8.5.0.tar.gz](https://ftpmirror.gnu.org/gcc/gcc-8.5.0/gcc-8.5.0.tar.gz)
tar xzf gcc-8.5.0.tar.gz
cd gcc-8.5.0

# Download internal prerequisites
./contrib/download_prerequisites

# Configure and build (uses -j$(nproc) to utilize all available CPU cores)
cd ..
mkdir objdir && cd objdir
../gcc-8.5.0/configure \
    --prefix=/usr/local/gcc-8.5.0 \
    --enable-languages=c,c++ \
    --disable-multilib

make -j$(nproc)
sudo make install

```

Add GCC 8.5.0 to your `PATH` and verify:

```bash
export PATH=/usr/local/gcc-8.5.0/bin:$PATH
gcc --version
# Expected output: gcc (GCC) 8.5.0

```

---

### Step 2.2: Build MPICH 3.4.3

MPICH must be compiled using your newly built GCC 8.5.0 to ensure consistent DWARF metadata across MPI libraries:

```bash
cd ~/toolchain

# Download source
wget [https://www.mpich.org/static/downloads/3.4.3/mpich-3.4.3.tar.gz](https://www.mpich.org/static/downloads/3.4.3/mpich-3.4.3.tar.gz)
tar xzf mpich-3.4.3.tar.gz
cd mpich-3.4.3

# Point explicitly to GCC 8.5.0
export CC=/usr/local/gcc-8.5.0/bin/gcc
export CXX=/usr/local/gcc-8.5.0/bin/g++

# Configure and build
./configure \
    --prefix=/usr/local/mpich-3.4.3 \
    --disable-fortran

make -j$(nproc)
sudo make install

```

Add MPICH to your `PATH` and verify:

```bash
export PATH=/usr/local/mpich-3.4.3/bin:$PATH
mpicc -v
# Should reference GCC 8.5.0

```

---

### Step 2.3: Install Go 1.21

```bash
cd ~/toolchain

wget [https://go.dev/dl/go1.21.0.linux-amd64.tar.gz](https://go.dev/dl/go1.21.0.linux-amd64.tar.gz)
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz

```

Add Go to your `PATH` and verify:

```bash
export PATH=/usr/local/go/bin:$PATH
go version
# Expected output: go version go1.21.0 linux/amd64

```

---

### Step 2.4: Build CRIU 3.19 (Recommended Backend)

CRIU is used for kernel-assisted, multi-level, and incremental checkpointing. Build it from source using GCC 8.5.0:

```bash
cd ~/toolchain

# Download source
wget [https://github.com/checkpoint-restore/criu/archive/refs/tags/v3.19.tar.gz](https://github.com/checkpoint-restore/criu/archive/refs/tags/v3.19.tar.gz)
tar -xzf v3.19.tar.gz
cd criu-3.19

# Build using GCC 8.5.0
export CC=/usr/local/gcc-8.5.0/bin/gcc
export HOSTCC=/usr/local/gcc-8.5.0/bin/gcc

make
sudo make install

```

Verify installation:

```bash
criu --version
# Expected output: Version: 3.19

```

---

### Step 2.5: Install DMTCP (Alternative User-Space Backend)

Unlike CRIU, DMTCP operates entirely in user space without root privileges.

#### Option A: Install via Package Manager (Ubuntu/Debian)

```bash
sudo apt-get install -y dmtcp

```

#### Option B: Build from Source

```bash
cd ~/toolchain

git clone [https://github.com/dmtcp/dmtcp.git](https://github.com/dmtcp/dmtcp.git)
cd dmtcp
./configure --prefix=/usr/local/dmtcp
make -j$(nproc)
sudo make install

export PATH=/usr/local/dmtcp/bin:$PATH

```

Verify installation:

```bash
dmtcp_launch --version
# or
dmtcp_coordinator --version

```

---

## 3. Persistent Environment Variables

To avoid re-exporting paths in every new terminal session, append the following configuration to your `~/.bashrc`:

```bash
# GCC 8.5.0
export PATH=/usr/local/gcc-8.5.0/bin:$PATH

# Go 1.21
export PATH=/usr/local/go/bin:$PATH

# MPICH 3.4.3
export PATH=/usr/local/mpich-3.4.3/bin:$PATH

# DMTCP (if built from source)
export PATH=/usr/local/dmtcp/bin:$PATH

# Compiler flags for debugger-compatible MPICH compilation
export CC=/usr/local/gcc-8.5.0/bin/gcc
export CXX=/usr/local/gcc-8.5.0/bin/g++

```

Apply the changes immediately:

```bash
source ~/.bashrc

```

---

## 4. Building RevDebug-MPI

Once all toolchain prerequisites are installed and in your `PATH`, clone the debugger repository and run `make`:

```bash
git clone [https://github.com/RevDebug-MPI/RevDebug-MPI.git](https://github.com/RevDebug-MPI/RevDebug-MPI.git)
cd RevDebug-MPI

make

```

This compiles the three core binaries into the `./bin/` directory:

| Binary              | Description                                               |
|---------------------|-----------------------------------------------------------|
| `bin/orchestrator`  | Main debugger process manager and storage tier controller |
| `bin/compiler`      | MPI wrapper compiler for instrumenting target programs    |
| `bin/node-debugger` | Per-process debug agent                                   |

---

## 5. Usage Guide

### 5.1 Compile Your MPI Program

Target MPI programs must be compiled using the included compiler wrapper to instrument MPI calls and generate compatible
DWARF metadata:

```bash
bin/compiler <path-to-your-mpi-program>

```

The compiled executable is placed in `./bin/targets/<source-file-name>`. For example, to compile the circular
message-passing benchmark `circle.c`:

```bash
bin/compiler examples/circle.c
# Output binary: ./bin/targets/circle

```

---

### 5.2 Run the Debugger

The general syntax for launching the orchestrator is:

```bash
bin/orchestrator <num_processes> <path-to-binary> <criu|dmtcp>

```

#### Option 1: Running with CRIU (Requires Root)

Because CRIU requires kernel-level permissions to inspect process trees and track soft-dirty memory bits, run it with
`sudo env PATH=$PATH`:

```bash
sudo env PATH=$PATH bin/orchestrator 2 ./bin/targets/circle criu

```

#### Option 2: Running with DMTCP (Unprivileged User Space)

```bash
bin/orchestrator 2 ./bin/targets/circle dmtcp

```

Upon startup, the orchestrator initializes the RPC listeners, launches the MPI ranks, and starts the WebSocket server
for the Web GUI:

```text
rpc server listening on address: localhost:3490
executing ./bin/targets/circle as an mpi job with 2 processes
Starting the gui - /usr/bin/npm run start:open
starting websocket server for gui
1 - binary started, waiting for command
0 - binary started, waiting for command
1 - rpc server listening on address: localhost:3501
0 - rpc server listening on address: localhost:3500

```

---

### 5.3 Accessing the Web GUI

When `bin/orchestrator` starts, it automatically launches a web-based graphical interface communicating via WebSocket on
port **3496**.

* **Local Machine:** The GUI opens automatically in your default web browser at `http://localhost:3496`.
* **Remote Server (SSH / Cloud VM / Cluster Node):**

1. Forward port `3496` from the remote machine to your local workstation:

```bash
ssh -L 3496:localhost:3496 user@your-server

```

2. Open `http://localhost:3496` in your local browser.

You can verify that the WebSocket listener is active at any time using `curl`:

```bash
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Host: localhost:3496" \
  -H "Origin: http://localhost:3496" \
  -H "Sec-WebSocket-Key: dGhllHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  http://localhost:3496/

```

---

### 5.4 Important Operational Notes

* **Preserving `PATH` with `sudo`:** When running under `sudo` with CRIU, default security policies in `/etc/sudoers` (
  `secure_path`) often strip `/usr/local/...` from root's environment. Always use
  `sudo env PATH=$PATH bin/orchestrator ...` so that `mpirun`, `gcc`, and `criu` remain reachable.
* **Network Port Availability:** Ensure that TCP ports `3490` (Orchestrator RPC), `3496` (Web GUI WebSocket), and
  `3500+` (Per-Rank Debug Agents) are free and not blocked by local firewall rules.