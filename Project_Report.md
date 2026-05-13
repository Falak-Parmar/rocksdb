# DS614 Project Report: RocksDB Internal Experiments

This project explores the internal mechanics of RocksDB through three isolated experiments focusing on observability, concurrency, and memory management.

## Experiments Overview

### 1. [Experiment 1: Observability](./reports/experiment_1_observability.md)
Implementation of a live reporting engine in the write path to monitor real-time insertion rates, entry sizes, and MemTable occupancy.

### 2. [Experiment 2: Concurrency Collapse](./reports/experiment_2_concurrency_collapse.md)
A comparative study of RocksDB's default lock-free SkipList versus a coarse-grained mutex implementation to demonstrate the impact of lock contention on multi-core performance.

### 3. [Experiment 3: Adaptive Flush](./reports/experiment_3_adaptive_flush.md)
Analysis of memory management stability by modifying the MemTable flush trigger to be proactive (75% occupancy) rather than reactive (100% occupancy).

---
*Refer to the individual reports in the `reports/` directory for detailed setups, results, and architectural analysis.*
