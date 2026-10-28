# 🚀 Fault-Tolerant Topic-Based File Distribution System

[![Language: C](https://img.shields.io/badge/Language-C99%20%2F%20POSIX-blue.svg)](https://en.wikipedia.org/wiki/C_(programming_language))
[![Sockets: TCP](https://img.shields.io/badge/Networking-POSIX%20Sockets-orange.svg)](https://pubs.opengroup.org/onlinepubs/9699919799/)
[![Concurrency: pthreads](https://img.shields.io/badge/Concurrency-pthreads-green.svg)](https://en.wikipedia.org/wiki/Pthreads)
[![Python: 3.x](https://img.shields.io/badge/Benchmarks-Python%203-yellow.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-lightgrey.svg)](LICENSE)

A high-performance, fault-tolerant network file distribution service implemented in **C** using raw **POSIX TCP Sockets**. The system provides topic-based publish/subscribe semantics, binary protocol framing, concurrency models, probabilistic network fault injection, and stateful checkpoint-based session resumption with parallel worker streams.

---

## 📌 Table of Contents

- [Overview](#-overview)
- [Evolution of Implementations](#-evolution-of-implementations)
  - [Part I — Baseline (Single vs. Multi-Threaded)](#part-i--baseline-single-vs-multi-threaded)
  - [Part II: Case 1 — Naive Retry-from-Start](#part-ii-case-1--naive-retry-from-start)
  - [Part II: Case 2 — Checkpoint-Based Resumption](#part-ii-case-2--checkpoint-based-resumption)
  - [Part II: Case 2 Enhanced — Parallel TCP Workers](#part-ii-case-2-enhanced--parallel-tcp-workers)
- [Binary Wire Protocol](#-binary-wire-protocol)
- [Project Architecture & Sequence](#-project-architecture--sequence)
- [Repository Structure](#-repository-structure)
- [Prerequisites & Build Guide](#-prerequisites--build-guide)
- [Usage & Execution Examples](#-usage--execution-examples)
- [Benchmarking & Performance Evaluation](#-benchmarking--performance-evaluation)
- [Contributors](#-contributors)

---

## 📖 Overview

The server organizes datasets hierarchically where each subfolder under a designated root path acts as an isolated **topic**. Clients query a topic name and automatically receive all regular files contained within that topic.

### Key Capabilities:
- **Custom Framed Binary Protocol:** Fixed 28-byte headers preventing TCP stream fragmentation / boundary merging issues.
- **Dynamic Concurrency:** Sequential vs. worker-thread connection models.
- **Simulated Fault Injection:** Configurable per-chunk socket drop probabilities to test recovery resilience.
- **Smart Resumption:** Session tracking and byte-offset checkpoint validation ensuring zero redundant byte retransmission.
- **Parallel Multiplexing:** Client-side work queue distributing topic chunks across multiple concurrent TCP connections.

---

## 🔄 Evolution of Implementations

| Version | Directory | Concurrency | Fault Recovery Strategy | Redundant Byte Overhead |
|:---|:---|:---|:---|:---|
| **Part I** | `Part1/` | Single-Threaded / Multi-Threaded (`pthreads`) | None (Detects socket break and aborts) | N/A |
| **Part II (Case 1)** | `Part2_Case1/` | Single-Threaded with Fault Injection | **Restart from Beginning**: Discards progress and redownloads all topic files | High ($O(N)$ with failure rate) |
| **Part II (Case 2)** | `Part2_Case2/` | Stateful Server with Session Manager | **Byte-Offset Resume**: Validates last flushed disk offset and requests only remaining data | Minimal (Control messages only) |
| **Part II (Case 2 Enhanced)** | `Part2_Case2_Enhanced/` | Multi-Stream TCP Workers + Shared Work Queue | **Parallel Worker Failover**: Failed worker reconnects and resumes its assigned file while others continue | Lowest transfer latency |

---

## 🔌 Binary Wire Protocol

All control frames and data chunks use a unified **28-byte packed header** followed by a variable-length payload:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Magic Number (0x49543330 / "IT30")            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Msg Type             |         Status Flags          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Session ID (64-bit)                     |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Transaction ID / Counter (64-bit)           |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Payload Length (32-bit)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Payload (Variable Length)               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
