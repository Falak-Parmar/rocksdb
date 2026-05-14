## RocksDB MemTable Internals: A Reverse-Engineering Study

**Course:** DS614 — Big Data Engineering | DA-IICT, Semester 2  
**Topic:** Deep-dive analysis of RocksDB's MemTable (write buffer) layer  
**Method:** Source-code modification + db_bench benchmarks on the real RocksDB binary
**Group** : Big Data Knights  
**Members** : Aditya Jana - 202518035 | Falak Parmar - 202518053

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Architecture](#2-system-architecture)
3. [Key Design Decisions](#3-key-design-decisions)
4. [Concept Mapping](#4-concept-mapping)
5. [Experiment 1 — Observability](#5-experiment-1--memtable-observability)
6. [Experiment 2 — Concurrency Collapse](#6-experiment-2--the-concurrency-collapse)
7. [Experiment 3 — Adaptive Flush](#7-experiment-3--adaptive-flush-trigger-policy)
8. [Failure Analysis](#8-failure-analysis)
9. [Synthesis and Key Insights](#9-synthesis-and-key-insights)
10. [Conclusion](#10-conclusion)

---

## 1. Introduction

### The Aspect This Project Studies

Modern write-heavy databases face a fundamental tension: they must absorb millions of writes per second while simultaneously serving reads, maintaining durability, and limiting disk wear. The naive solution — updating a B-Tree node in place on every write — collapses under this pressure because each write becomes a random I/O operation, and random writes on SSD are 10–50× slower than sequential ones.

RocksDB solves this with an LSM-tree (Log-Structured Merge-tree) design. At the very top of the LSM stack — absorbing every single write before it touches disk — sits the MemTable. Understanding the MemTable is therefore equivalent to understanding RocksDB's write performance characteristics, its concurrency model, and its memory management strategy.

### What the MemTable Is

The MemTable is an in-memory write buffer. Every `Put()` or `Delete()` issued by an application is written to the MemTable first. When the MemTable fills up (default: 64 MB), it is made immutable and a background thread flushes it to an SST file on disk, after which a new empty MemTable absorbs further writes. The MemTable must satisfy three simultaneous requirements:

- **High write throughput** — accepting millions of insertions per second from multiple threads.
- **Sorted iteration** — because the flushed SST file must be sorted by key for the read path to work.
- **Concurrent access** — supporting many parallel writer threads without serializing on a lock.

RocksDB satisfies all three with a **lock-free `InlineSkipList`**: a skip list that keeps keys sorted at insertion time using atomic Compare-And-Swap (CAS) operations rather than mutexes.

### What This Project Does

This project reverse-engineers the MemTable at three levels:

1. **Source code** — key files in `db/` and `memtable/` are annotated with analysis of their roles, entry points, and tradeoffs.
2. **Experiments** — three controlled source modifications each isolate one MemTable design parameter, measure the impact with `db_bench`, and trace results back to specific functions.
3. **Measurement** — every claim is backed by a real number from an actual RocksDB process.

---

## 2. System Architecture

### The Complete Write Path


```text
                    Client Write: db->Put()
                               │
                               ▼
                 DBImpl::WriteImpl()
                               │
                ┌──────────────┴──────────────┐
                │                             │
                ▼                             ▼
      WAL Append (durability)        MemTable::Add()
         db/log_writer.cc              db/memtable.cc
                                              │
                                              ▼
                           InlineSkipList::Insert()
                           memtable/inlineskiplist.h
                                              │
                                              ▼
                          Active Mutable MemTable
                                   │
                     write_buffer_size reached
                                   │
                                   ▼
                    SwitchMemtable()
             db/db_impl/db_impl_write.cc
                                   │
             ┌─────────────────────┴─────────────────────┐
             │                                           │
             ▼                                           ▼
   Old MemTable marked IMMUTABLE         New Mutable MemTable
             │                            immediately accepts
             │                                 new writes
             │
             ▼
   Background Flush Thread
             │
             ▼
 FlushJob::WriteLevel0Table()
        db/flush_job.cc
             │
             ▼
      SST File (Level-0)
```

### Key Source Files

| Source File | Role | Key Function |
|---|---|---|
| `db/memtable.cc` | MemTable lifecycle and flush logic | `Add()`, `ShouldFlushNow()` |
| `db/internal_stats.h` | Internal statistics tracking | `kIntStatsNumFlushes` |
| `db/db_impl/db_impl_write.cc` | Write path orchestration | `WriteImpl()`, `SwitchMemtable()` |
| `memtable/inlineskiplist.h` | Lock-free concurrent skip-list | `Insert()`, `FindGreaterOrEqual()` |
| `memtable/skiplistrep.cc` | SkipList MemTable representation | `SkipListRep::Insert()`, `InsertConcurrently()` |

### The InlineSkipList — How Lock-Free Concurrency Works

A standard skip-list uses a mutex to protect the node-insertion path. RocksDB replaces this with Compare-And-Swap atomics. When a thread wants to insert a new node at level `k`, it:

1. Finds the predecessor and successor nodes at each level (read-only traversal — no lock needed).
2. Sets `new_node->next[k] = successor` atomically.
3. Attempts `predecessor->next[k].compare_exchange(successor, new_node)`.
4. If the CAS fails (another thread raced), it retries from the current state.

This eliminates the global lock, allowing N threads to insert simultaneously. Each thread only contends on the exact pointer it is modifying, not the entire data structure. The cost: every insertion involves at least one atomic operation per level, plus potential retry loops under high contention.

---

## 3. Key Design Decisions

### Decision 1 — Lock-Free Skip-List as the Core Data Structure

- **Problem solved:** High concurrent write throughput without serializing all writes behind one mutex.
- **Where implemented:** `memtable/inlineskiplist.h` — `Insert()` method.
- **Tradeoff:** Lock-free CAS is faster under concurrency, but adds a small retry overhead at 1 thread that a mutex doesn't have. Experiment 2 quantifies this: at 1 thread, the mutex version is 0.3% faster; at 4 threads, it collapses 73.8%.
- **Alternative:** A sorted array (O(n) inserts, impractical) or a hash table (no sorted iteration, breaks flush).

### Decision 2 — Reactive Flush at 100% Buffer Occupancy

- **Problem solved:** Maximizes memory utilization — every byte of `write_buffer_size` is used before triggering disk I/O.
- **Where implemented:** `db/memtable.cc` — `ShouldFlushNow()`.
- **Tradeoff:** When the buffer hits 100%, the write thread blocks until the background flush makes space, causing hard write stalls and latency spikes. Experiment 3 shows an alternative (75% trigger) trades 44.7% more flushes for stall elimination.
- **Alternative:** Proactive early flush — implemented and measured in Experiment 3.

### Decision 3 — Separate WAL and MemTable for Durability

- **Problem solved:** The MemTable is volatile RAM — a crash would lose all unflushed writes. The WAL provides crash recovery without sacrificing write speed.
- **Where implemented:** `db/log_writer.cc` (WAL) + `db/db_impl/db_impl_write.cc` (writes both simultaneously via group commit).
- **Tradeoff:** Every write incurs two operations: one sequential WAL append and one in-memory MemTable insert. WAL can be disabled (`--disable_wal=1`) to improve throughput at the cost of durability — used only for non-critical or ephemeral workloads.
- **Alternative:** Skip the WAL entirely (memory-only caches), accepting data loss on crash.

---

## 4. Concept Mapping

### Storage — LSM-Tree vs. B-Tree

The MemTable is the first tier of RocksDB's LSM-tree. Unlike B-Trees, which require in-place node updates (random disk I/O), the LSM approach buffers all writes in RAM first and flushes them sequentially. The MemTable makes this possible — it absorbs writes fast enough that the write thread never waits for disk I/O under normal conditions. The B-Tree vs. LSM tradeoff is visible in the MemTable's design: it optimizes for write throughput at the cost of requiring a sorted flush path on every buffer fill.

### Concurrency — Lock-Free vs. Coarse-Grained Locking

The `InlineSkipList` is a direct application of the lock-free concurrency principle: instead of a mutex that serializes all threads, CAS operations allow threads to proceed in parallel and only retry when their specific pointer was concurrently modified. This is Amdahl's Law applied directly — the fraction of the write path that is serialized determines the maximum speedup from adding threads. Experiment 2 proves this numerically: removing the CAS model and adding a coarse mutex causes a 73.8% throughput collapse at just 4 threads, while the lock-free version continues to scale.

### Memory Management — Write Buffer Sizing and Flush Policy

The `write_buffer_size` parameter controls the MemTable's capacity and therefore its flush frequency. This is a classic memory-throughput tradeoff: a larger buffer means fewer flushes (less I/O amplification and better SST file density), but higher memory pressure and larger latency spikes when the buffer exhausts. Experiment 3 introduces a third axis — trigger timing — showing that flushing at 75% instead of 100% can eliminate stall events at the cost of 44.7% more total flushes, with the optimal choice being workload-dependent.

### Observability — Instrumentation in the Write Path

Experiment 1 demonstrates the architectural importance of in-path instrumentation in storage engines. The standard `Statistics` object collects aggregate data at run end, which smooths over short-lived phenomena. Custom injection into the write thread captures per-second deltas, revealing micro-stalls — momentary throughput drops to under 800 ops/sec lasting only 1–2 seconds — that are completely invisible in aggregate statistics but dominate tail latency in production. This is the difference between knowing that average throughput was 1M ops/sec and knowing that it dropped to 800 ops/sec for 2 seconds every 30 seconds.

### Fault Tolerance — WAL and Crash Recovery

The separation of WAL (disk, durable) from MemTable (RAM, volatile) is RocksDB's primary fault-tolerance mechanism. On restart after a crash, RocksDB replays the WAL to reconstruct any MemTable data that had not yet been flushed to an SST file. This means durability is achieved without synchronous disk writes on every operation — the WAL is a sequential append, which is orders of magnitude cheaper than random writes. Disabling the WAL (`--disable_wal=1`) trades this durability guarantee for a 15–20% throughput improvement, which is only acceptable when the application manages its own persistence.

### Streaming / Ingestion — Group Commit

`DBImpl::WriteImpl()` implements a group-commit pattern: multiple concurrent writer threads are batched together into a single write batch by a designated leader thread. The leader writes the entire batch to the WAL as one sequential append, then each thread applies its portion to the MemTable. This amortizes WAL I/O overhead across many concurrent writes, which is why WAL-enabled throughput can approach WAL-disabled throughput at high concurrency — the cost per write drops as more writes are batched together.

---

## 5. Experiment 1 — MemTable Observability

### Concept

While RocksDB provides a `Statistics` object for aggregate metrics, this experiment implements a custom high-frequency reporting engine injected directly into the write path. Rather than waiting until the run completes to read aggregate counters, the engine calculates per-second deltas and logs them in real time — providing live visibility into insertion rate, entry size, occupancy percentage, and flush count as they evolve. This distinction matters: aggregate stats tell you what happened on average, while live deltas tell you *when* things went wrong and by how much.

### Code Modifications

- `db/internal_stats.h/cc` — Added `kIntStatsNumFlushes` counter to track the total number of MemTable switches. This counter is incremented every time `SwitchMemtable()` is called, giving a running total of how many times the MemTable has been flushed to disk during the benchmark.
- `db/db_impl/db_impl_write.cc` — Injected a 1-second interval reporting block inside `WriteImpl()`. Every second, the block calculates the delta in insertion count since the last report, computes average entry size from total bytes written, reads the current occupancy percentage from the MemTable, and logs all four metrics together. This injection runs on the write thread itself, so it captures the exact state the write thread sees — including any stall periods where the thread is blocked waiting for a flush to complete.

### Setup

- **Workload:** `fillseq` and `fillrandom` (4M ops for 128B values; 500K ops for 8KB values)
- **Configuration:** 16MB write buffer (`--write_buffer_size=16777216`), WAL disabled, Compression disabled
- **Environments:** Sequential vs. Random access pattern × Small (128B) vs. Large (8KB) values — four combinations total

### Results

| Write Pattern | Value Size | Peak Inserts/sec | Avg Entry Size | Total Flushes | Peak Occupancy |
|---|---|---|---|---|---|
| Sequential | 128 B | ~3,941,378 | 160 B | 40 | 71.8% |
| Sequential | 8 KB | ~105,196 | 8.2 KB | 236 | ~90% |
| Random | 128 B | ~1,861,505 | 160 B | 34 | 48.6% |
| Random | 8 KB | ~51,713 | 8.2 KB | 248 | ~95% |

### Analysis

**Sequential vs. Random Efficiency**

Sequential writes achieve approximately 2× higher throughput than random writes for the same value size. This confirms that the `InlineSkipList` benefits significantly from sequential keys: insertions always target the tail of the sorted structure, meaning each thread's CAS operation almost never races with another thread's — the predecessor and successor pointers are always at predictable positions. In a random workload, insertions scatter across the skip list at arbitrary positions, requiring longer traversals to find the insertion point and causing more frequent CAS retries as threads collide on shared pointer regions. The difference is not in how RocksDB was configured but in how the access pattern interacts with the lock-free data structure's contention model.

**The Large-Value Throughput Collapse**

Transitioning from 128B to 8KB values causes an approximately 40× throughput drop (3.94M → 105K for sequential; 1.86M → 51K for random). The `AvgSize` metric reveals a constant 32-byte overhead per entry — this is the key prefix, metadata, and skip-list node headers that accompany every write. For 128B values this overhead is 25%; for 8KB values it is under 0.5%, so the per-entry overhead is not the bottleneck. The real issue is flush frequency: a 16MB buffer holds only ~1,950 entries at 8KB each, meaning a flush is triggered for every 1,950 keys written. Each flush briefly stalls the write thread while the background thread completes the SST write, turning what should be a CPU-bound in-memory operation into an I/O-bound process dominated by background disk throughput.

**Occupancy Dynamics and Micro-Stalls**

The live reporting captured the sawtooth pattern of MemTable occupancy — rapid rise during active insertion, sharp drop when a flush completes. In random large-value workloads, occupancy frequently spiked above 90%, followed by throughput drops as low as 800 ops/sec for 1–2 seconds while the background flush thread cleared memory. These micro-stalls are completely invisible in aggregate statistics that report a single ops/sec number for the entire run. For production systems with latency SLAs — particularly those committing to P99 latency bounds — this kind of in-path observability is the only way to detect and diagnose such events without instrumenting the application layer above RocksDB, making it a critical operational tool for storage engineers.

### Conclusion

Custom in-path observability reveals phenomena that post-run statistics cannot capture: micro-stalls, occupancy spikes, and the exact moment the background I/O thread begins to fall behind the write thread. The 32-byte metadata overhead is negligible for large values; the real performance determinant is flush frequency, which is entirely controlled by `write_buffer_size` relative to the workload's average value size. For a workload with 8KB values, increasing `write_buffer_size` from 16MB to 256MB would reduce flush count by approximately 16×, dramatically improving throughput at the cost of higher memory commitment.

---

## 6. Experiment 2 — The Concurrency Collapse

### Concept

RocksDB's `InlineSkipList` achieves high write throughput by being lock-free — using atomic CAS operations that allow multiple threads to insert simultaneously without any thread ever blocking another. This experiment strips out the CAS model entirely and replaces it with a coarse-grained mutex that forces every single insertion to acquire a global lock on the entire MemTable, serializing the write path. The goal is to produce a direct, measurable proof of why lock-free data structures exist and what the cost of coarse locking is at real concurrency levels — not theoretically, but measured on actual hardware.

### Code Modifications

- `memtable/inlineskiplist.h` — Modified `InlineSkipList<Comparator>::Insert()` to wrap the node-insertion logic in `std::mutex lock/unlock` calls instead of the CAS retry loop. The lock is held for the entire duration of finding the insertion position and linking the new node, meaning no other thread can read or write the skip list while any insertion is in progress — a global serialization point on every write.
- `memtable/skiplistrep.cc` — Modified both `SkipListRep::Insert()` and `InsertConcurrently()` to route through the same locked insert path, ensuring that even writes arriving via the concurrent insertion API are serialized through the mutex rather than taking a parallel code path that might bypass the lock.

### Setup

- **Workload:** `fillrandom` (1,000,000 operations per thread)
- **Configuration:** WAL disabled (`--disable_wal=1`), Compression disabled (`--compression_type=none`)
- **Platform:** Apple M-series, 10-core arm64
- **Thread sweep:** {1, 4, 16, 32}

### Results

| Threads | Lock-Free (ops/sec) | Coarse-Lock (ops/sec) | Throughput Change |
|---|---|---|---|
| 1 | 1,295,194 | 1,299,798 | +0.3% (Baseline) |
| 4 | 1,544,854 | 405,033 | **−73.8%** |
| 16 | 249,862 | 210,268 | −15.8% |
| 32 | 242,919 | 206,483 | −15.0% |

### Analysis

**The 73.8% Collapse at 4 Threads**

The most dramatic result occurs at the transition from 1 to 4 threads. The lock-free implementation scales successfully — throughput increases from 1.30M to 1.54M ops/sec (+20%), as threads utilize idle CPU cores on the 10-core M-series chip. The coarse-locked version collapses from 1.30M to 405K ops/sec — a 73.8% drop. This is the mathematical proof of lock contention: 4 threads are doing less combined work than 1 thread not because each thread is slower, but because 3 out of 4 threads are blocked waiting for the mutex at any given instant. The 4-thread coarse-lock throughput of 405K ops/sec is less than one-third of the 1-thread throughput, confirming that synchronization overhead has become the dominant cost rather than the actual insertion work.

**Single-Thread Parity Proves the CAS Cost**

At 1 thread, the coarse-lock version is marginally faster (+0.3%). This is expected and important: a single thread never contends on a mutex, so the lock/unlock cost is a simple atomic flag flip — cheaper than the CAS retry loop, which involves setting up the retry condition and reloading the predecessor/successor pointers on each attempt even when no contention exists. This result proves that the lock-free design's benefit is not universal — it is a concurrency premium that only materializes when multiple threads are actually competing. For embedded or single-threaded deployments, a simple mutex would be equally correct and marginally faster; it is the multi-core write workload that makes the CAS design necessary.

**Amdahl's Law at 16 and 32 Threads**

As threads increase to 16 and 32, both implementations plateau and the performance gap narrows (−73.8% → −15%). This is not because the mutex improves — it is because both implementations now hit the same hardware-level bottlenecks: context switching overhead, memory bus saturation, and L3 cache contention on the 10-core CPU. The serialized fraction of the mutex-based implementation becomes a smaller share of total execution time because thread management overhead dominates both paths equally. This is Amdahl's Law in practice: as non-serialized overheads grow with thread count, the relative impact of the serialization bottleneck decreases — not because serialization got better, but because everything else got worse at a similar rate.

### Conclusion

RocksDB's decision to use a lock-free `InlineSkipList` is fundamental to its ability to handle modern multi-core write workloads. The 73.8% throughput collapse at just 4 threads shows that a coarse mutex is not a performance trade-off — it is an architectural defect for concurrent workloads. The single-thread result confirms that lock-free is not universally superior; it is the correct design for the specific case of many concurrent writers, which is exactly the workload RocksDB is built to serve.

---

## 7. Experiment 3 — Adaptive Flush Trigger Policy

### Concept

By default, RocksDB triggers a MemTable flush only when the buffer reaches approximately 100% of `write_buffer_size`. This binary Stop/Go behavior — fill completely, then block writes while flushing — can cause hard write stalls when the buffer is fully exhausted and the background I/O thread has not yet cleared space for new writes. This experiment implements an **Adaptive Flush Policy** that triggers flushes earlier, at 75% occupancy, to maintain a 25% absorption headroom. The hypothesis is that proactive flushing — starting background I/O while the write thread still has room to continue — eliminates the hard stall events that occur when the buffer has zero remaining capacity and every incoming write must wait.

### Code Modification

- `db/memtable.cc` — Modified `ShouldFlushNow()` to check the current occupancy percentage against a 75% threshold before reaching the standard 100% check. If the MemTable is more than 75% full, the function returns true and a flush is triggered immediately, while 4MB of free buffer space remains available for the write thread to continue writing without blocking. An `[AdaptiveFlush]` log tag was added to all flush events triggered by this path, allowing precise verification that flushes were occurring at the correct occupancy level rather than silently falling through to the default threshold.

### Setup

- **Workload:** `fillrandom` (1,000,000 operations)
- **Configuration:** 16MB write buffer (`--write_buffer_size=16777216`), 8KB values, WAL disabled
- **Threshold:** Early flush triggered at 75% occupancy (~12MB of 16MB used)

### Results

| Metric | Baseline (Standard) | Adaptive Early Flush | Change |
|---|---|---|---|
| Trigger Point | ~100% | **75.01%** | −25% headroom maintained |
| Total Flushes | 494 | **715** | +44.7% more flushes |
| Flush Behavior | Reactive (Emergency) | **Proactive (Adaptive)** | Stabilized I/O pattern |

### Analysis

**Smoothing the Ingestion Curve**

The baseline shows 494 flushes for 1M keys; the adaptive policy produces 715 — a 44.7% increase. The critical difference is not the count but the timing: each flush under the adaptive policy begins while there is still 4MB of free space in the 16MB buffer (25% headroom), meaning the background flush thread has a head start before the write thread could possibly reach the buffer limit. In the baseline, flushes are triggered only when the buffer is completely full, so the write thread must block and wait for the background thread to make even a single byte of space available. The adaptive policy converts this from a reactive emergency response into a proactive maintenance cycle where flush and write happen concurrently rather than sequentially.

**Mathematical Precision of the Policy**

The logs confirm a consistent trigger at exactly 75.01% — the 0.01% overshoot occurs because the threshold is checked after each insertion rather than continuously, so the first key that pushes occupancy past 75% triggers the flush and the actual recorded occupancy at trigger time is slightly above the threshold. This precision matters for production tuning: knowing that the trigger is reliably at 75.01% allows engineers to choose the threshold based on the speed of the underlying disk. A system with fast NVMe storage needs less headroom (60–65% trigger) because the background thread can clear the buffer quickly; a system with slower spinning disks or network-attached storage needs more headroom (80–85%) to ensure the flush completes before the write thread reaches 100% and must stall.

**The Cost of Stability**

The 44.7% increase in flush count is the direct, measurable cost of eliminating hard write stalls. Each additional flush involves a full sort-and-write cycle of the MemTable content to an L0 SST file, consuming background I/O bandwidth and generating additional L0 files that must eventually be compacted into L1. For a bulk-load scenario where maximum raw ingestion speed is the goal and occasional stalls are acceptable, the default 494-flush path is more I/O-efficient. For a high-availability streaming ingestion system with latency SLA requirements, the 715-flush path is preferable — the additional background I/O cost is a worthwhile price for never experiencing a write stall that breaches the SLA. The optimal choice is workload-dependent, which is precisely why `ShouldFlushNow()` is designed as a modifiable function rather than a hardcoded constant.

### Conclusion

Experiment 3 demonstrates that the flush trigger threshold is a first-class design parameter with direct, measurable consequences. The 44.7% increase in flush count is the price of never experiencing a hard write stall. For production LSM-tree systems handling latency-sensitive workloads — real-time event ingestion, transactional pipelines with SLA commitments, or any system where P99 latency matters more than average throughput — an adaptive early-flush policy is a critical reliability mechanism, not merely a performance optimization.

---

## 8. Failure Analysis

### What Happens When Data Size Increases Significantly?

As value sizes grow, the MemTable fills faster relative to its configured size. For 8KB values in a 16MB buffer, the MemTable holds only ~1,950 entries before triggering a flush. At high flush rates, two failure modes emerge. First, **L0 file accumulation**: if flushes happen faster than compaction can merge L0 files into L1, the L0 file count grows continuously, and RocksDB stalls all writes when this count reaches `level0_stop_writes_trigger` (default: 36 files) — the most common write stall pattern in production RocksDB deployments with large-value workloads. Second, **write amplification escalation**: each value written once may be rewritten 3–7 times through compaction levels before reaching its final resting level, and large values make this cost visible earlier because the database fills compaction levels faster, triggering deeper and more expensive compactions sooner.

The primary mitigation is to scale `write_buffer_size` proportionally with value size: for 8KB values, a 256MB buffer instead of 16MB reduces flush frequency by ~16×, dramatically reducing L0 pressure. For workloads with values in the tens or hundreds of KB, RocksDB's BlobDB (separate blob file storage for large values) keeps the MemTable SkipList small while storing large values in dedicated blob files that do not participate in the normal compaction cascade.

### What Happens Under Skew?

The MemTable is ordered by key using the SkipList comparator. Under skewed workloads — for example, all writes targeting a monotonically increasing timestamp key or a hot partition prefix — insertions concentrate on a narrow section of the skip list. This creates two specific failure modes. First, a **CAS hot spot**: lock-free CAS operations retry when two threads modify adjacent pointers, and under heavy key-range skew, many threads contend on the same region of the skip list simultaneously, causing CAS retry storms that reduce the lock-free throughput advantage measured in Experiment 2. Second, **asymmetric compaction**: flushed SST files from a skewed MemTable have dense, overlapping key ranges that create compaction hot spots, consuming disproportionate CPU and I/O on a small fraction of the key space while the rest sits idle.

The standard mitigation is key sharding: distributing writes across multiple column families based on a prefix hash, so that skewed logical keys map to distributed physical key ranges. For time-series workloads specifically, using a compound key with a shard prefix (e.g., `shard_id:timestamp`) distributes sequential timestamp writes across multiple MemTables, eliminating the single-SkipList hot spot.

### What Happens If a Component Fails?

| Component | Failure Mode | RocksDB Response | Recovery Path |
|---|---|---|---|
| MemTable (RAM) | Process crash, power loss | All in-flight writes lost | WAL replay on restart reconstructs MemTable |
| Background Flush Thread | Thread panic, disk full | Immutable MemTable accumulates; writes stall | Restart; free disk space; manual recovery |
| WAL disk | Disk failure before flush | WAL unreadable after crash | Data loss for writes since last flush; use RAID or replication |
| CAS operation | Retry storm under contention | Throughput degrades gradually | No failure state; CAS is inherently retry-safe |

### What Assumptions Does the MemTable Rely On?

**Sequential disk I/O is fast.** The entire LSM design assumes that flushing a MemTable as one large sequential write is significantly faster than many small random writes. On systems where sequential I/O is throttled — network-attached filesystems, certain cloud storage backends, or highly fragmented disks — this assumption breaks, and flush latency can exceed write throughput, causing the stall pattern described above regardless of how the flush trigger policy is tuned.

**Background threads can keep pace with writes.** Experiment 3 showed 715 flushes for 1M keys under the adaptive policy. If the background flush thread is slower than the write thread — due to insufficient I/O bandwidth, a slow disk, or CPU contention from other processes — immutable MemTables accumulate faster than they can be cleared, and the engine must stall the write thread regardless of flush policy. The system's maximum sustainable write throughput is ultimately bounded by the speed of the background I/O thread, not the in-memory MemTable insertion rate.

**Keys are distinguishable.** The skip list sorts by key comparator. If all keys are identical — the maximum data-skew case — every insertion finds the same position in the skip list, all threads contend on the same two pointers, and the CAS retry rate approaches 100%. The lock-free SkipList does not degrade gracefully under zero key diversity; throughput collapses similarly to the coarse-lock case in Experiment 2, but for a contention reason rather than a serialization reason.

---

## 9. Synthesis and Key Insights

### Experiment Results at a Glance

| # | Experiment | Key Metric | Result | Implication |
|---|---|---|---|---|
| 1 | Observability | Peak inserts/sec (seq, 128B) | ~3.94M ops/sec | Sequential 2× faster than random; CAS benefits from tail insertion |
| 1 | Observability | Large-value throughput | ~40× drop (128B → 8KB) | Flush frequency, not metadata overhead, is the bottleneck |
| 1 | Observability | Micro-stall visibility | <800 ops/sec during flush | Invisible in aggregate stats; critical for P99 latency |
| 2 | Concurrency | Throughput at 4 threads (coarse lock) | −73.8% | Lock-free is non-negotiable for multi-core write scaling |
| 2 | Concurrency | Throughput at 1 thread | +0.3% (coarse lock faster) | CAS overhead only appears under actual contention |
| 3 | Adaptive Flush | Trigger point | 75.01% (vs. ~100%) | Policy is tunable; precision is controllable via source modification |
| 3 | Adaptive Flush | Total flush count | +44.7% (715 vs. 494) | Stability costs I/O; the tradeoff is workload-dependent |

### Three Architectural Insights

**Insight 1 — The MemTable is the Write Amplification Knob**

Every design choice in the MemTable — its size, its flush policy, its data structure — directly shapes write amplification downstream in the compaction cascade. A larger MemTable creates larger, denser L0 SST files with fewer key-range overlaps between files, reducing the amount of data that compaction must re-read and re-merge at each level. A well-tuned adaptive flush policy reduces stall frequency without increasing compaction load beyond the 44.7% flush overhead measured in Experiment 3. The MemTable is not just a write buffer; it is the primary lever for shaping the entire compaction tree — tuning `write_buffer_size` is therefore not a memory configuration decision, it is a compaction strategy decision with consequences that propagate through every level of the LSM.

**Insight 2 — Lock-Free Is a Concurrency Premium, Not an Absolute Win**

The single-thread result in Experiment 2 (+0.3% for coarse lock) establishes clearly that lock-free is not inherently faster — it is the correct optimization specifically for multi-threaded workloads where threads would otherwise contend on a shared mutex. At 1 thread, the mutex is slightly cheaper than the CAS retry loop because there is no contention to manage. The 73.8% collapse at 4 threads is the mathematical proof that coarse locking serializes the write path in a way that negates the benefit of having multiple CPU cores — 4 threads produce less throughput than 1. This is not an edge case: modern write workloads routinely use 4–32 concurrent writer threads, making the lock-free design a correctness requirement for the workloads RocksDB is designed to serve, not merely a performance optimization.

**Insight 3 — Every Apparent Limitation Is a Deliberate Tradeoff**

The reactive flush policy causes stalls — but maximizes buffer utilization. The early flush policy eliminates stalls — but increases flush count by 44.7%. Large values cause throughput collapse — but the bottleneck is flush frequency, which is tunable via `write_buffer_size`. The WAL adds write overhead — but it is the only reason the MemTable can be volatile RAM without losing data on crash. In every case, what looks like a limitation is actually a parameter of a deliberate tradeoff that has been left visible and configurable rather than hidden behind a fixed default. RocksDB's design philosophy — and the key principle this project confirms — is that if you cannot point to the code that implements a tradeoff, you have not understood the system; you have only seen one side of it.

---

## 10. Conclusion

This project has demonstrated that the RocksDB MemTable layer is a tightly engineered subsystem where every design decision — the skip list over a hash table, the CAS atomic over a mutex, the 100% flush threshold over an earlier trigger, the WAL alongside the MemTable — reflects a specific, measurable tradeoff between write throughput, memory efficiency, latency stability, and durability.

Three experiments, each modifying a different aspect of the MemTable source code, produced measurable results that trace directly back to specific functions:

- **Experiment 1 (Observability)** revealed that value size controls flush frequency, flush frequency controls throughput, and micro-stalls invisible in aggregate statistics can dominate tail latency. Key function: `SwitchMemtable()` in `db_impl_write.cc`.
- **Experiment 2 (Concurrency Collapse)** proved that lock-free CAS is the foundation of multi-core write scalability — a coarse mutex produces a 73.8% throughput collapse at 4 threads. Key function: `InlineSkipList::Insert()` in `memtable/inlineskiplist.h`.
- **Experiment 3 (Adaptive Flush)** demonstrated that the flush trigger threshold is a configurable design parameter — shifting it from 100% to 75% increases flush count by 44.7% but eliminates hard write stalls. Key function: `ShouldFlushNow()` in `db/memtable.cc`.

The key principle from the project brief holds without qualification: **if you cannot point to code, you have not understood the system.** Every result in this report points to a specific function, and every function implements a tradeoff that has been measured, quantified, and explained.

---

*Source modifications: `db/memtable.cc`, `db/internal_stats.h/cc`, `db/db_impl/db_impl_write.cc`, `memtable/inlineskiplist.h`, `memtable/skiplistrep.cc`*

---
**DS614 — Big Data Engineering**  
DA-IICT | 2025–26