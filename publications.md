---
layout: default
title: Publications & Team
permalink: /publications.html
---

# Publications & Research Team

**RevDebug-MPI** is an ongoing collaborative research project between the **Institute of Computer Science at the University of Tartu** (Estonia) and the **Sharif University of Technology** (Iran).

---

## Research Publications

### 1. Latest Developments of a Reversible Debugger for MPI
* **Authors:** Danial Darroudy, Eero Vainikko, and Stefan Kuhn
* **Focus:** Introduces the Multi-Level Checkpointing architecture (two-tier L1 RAM-disk to L2 persistent storage pipeline with asynchronous background migration) and kernel-assisted Incremental Checkpointing.

### 2. Extending Reversible MPI Debugging with Flexible Checkpointing Backends (2026)
* **Authors:** Stefan Kuhn, et al.
* **Journal:** *Future Generation Computer Systems (FGCS)*, Vol. 179, 2026, 108418.
* **Focus:** Extends the reversible MPI debugging architecture to support modular, interchangeable checkpointing backends including CRIU and user-space DMTCP.

```bibtex
@article{KUHN2026108418,
  title   = {Extending reversible MPI debugging with flexible checkpointing backends},
  author  = {Kuhn, Stefan and others},
  journal = {Future Generation Computer Systems},
  volume  = {179},
  pages   = {108418},
  year    = {2026}
}

```

### 3. A Reversible Debugger for MPI Applications (2024)

* **Authors:** Stefan Kuhn, Eero Vainikko, et al.
* **Conference:** *Proceedings of the ACM*, 2024.
* **DOI:** [10.1145/3678720.3685316](https://www.google.com/search?q=https://doi.org/10.1145/3678720.3685316)
* **Focus:** Foundational work introducing the causally consistent checkpoint-restore mechanism and orchestrator-agent model for reversible debugging of MPI programs.

```bibtex
@inproceedings{10.1145/3678720.3685316,
  title     = {A Reversible Debugger for MPI Applications},
  booktitle = {Proceedings of the ACM},
  year      = {2024},
  doi       = {10.1145/3678720.3685316}
}

```

---

## Research Team & Contributors

* **Danial Darroudy**
  *Sharif University of Technology, Tehran, Iran*
  Email: `danialdarroudy1383@gmail.com`
* **Prof. Eero Vainikko**
  *Professor of Distributed Systems, Institute of Computer Science, University of Tartu, Estonia*
  Email: `eero.vainikko@ut.ee` | Web: [www.ut.ee/~eero](https://www.google.com/search?q=https://www.ut.ee/~eero)
* **Dr. Stefan Kuhn**
  *Institute of Computer Science, University of Tartu, Estonia*
  ORCID: `0000-0002-5990-4157`

---

## Acknowledgments

We gratefully acknowledge the developers and open-source communities behind **CRIU (Checkpoint/Restore In Userspace)** and **DMTCP (Distributed MultiThreaded Checkpointing)** for providing the foundational process checkpointing frameworks utilized by RevDebug-MPI.

