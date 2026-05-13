# Experiment 1 — MemTable Insert Statistics (Observability)

### Concept
Understanding the internal state of the MemTable is critical for tuning RocksDB. While RocksDB provides a comprehensive `Statistics` object, this experiment implements a custom, high-frequency "live" reporting engine directly in the write path. This provides real-time visibility into the lifecycle of a MemTable—from insertion rate to occupancy growth to the eventual flush.

### Tradeoff
*   **Standard Stats (Default):** Collected at the end of a run or via specific API calls. Low overhead but high latency in reporting.
*   **Live Injection (Experiment 1):** Injected directly into the write thread. Provides instant feedback on how specific write patterns affect memory management and flush frequency. This version calculates **deltas** per second for highly accurate throughput measurement.

### Code Path
*   `db/internal_stats.h/cc`: Added `kIntStatsNumFlushes` to track the total number of MemTable switches.
*   `db/db_impl/db_impl_write.cc`: 
    - Incremented flush count in `SwitchMemtable`.
    - Injected a 1-second interval reporting block to calculate `inserts/sec`, `AvgSize`, `Flushes`, and `Occupancy %`.

### Setup
*   **Workload:** `fillseq` and `fillrandom` (4M ops for small, 500K for large)
*   **Configuration:** 16MB Buffer (`--write_buffer_size=16777216`), WAL disabled, Compression disabled.
*   **Environments:** Sequential vs. Random, Small (128B) vs. Large (8KB) values.

### Results
| Write Pattern | Value Size | Inserts/sec (Peak) | Avg Entry Size | Total Flushes | Peak Occupancy |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sequential** | 128B | ~3,941,378 | 160B | 40 | 71.8% |
| **Sequential** | 8KB | ~105,196 | 8.2KB | 236 | ~90% |
| **Random** | 128B | ~1,861,505 | 160B | 34 | 48.6% |
| **Random** | 8KB | ~51,713 | 8.2KB | 248 | ~95% |

### Analysis
**Sequential vs. Random Efficiency:**
Sequential writes are nearly **2x more efficient** at the MemTable level for small values. This confirms that the SkipList implementation benefits significantly from sequential keys, likely due to pre-fetching and reduced CAS contention in the insertion path.

**The "Large Value" Throughput Collapse:**
Transitioning from 128B to 8KB values results in a catastrophic performance drop (~40x). The "AvgSize" metric reveals a constant **32-byte overhead** per entry. For 8KB values, the MemTable fills so rapidly that the system triggers nearly **1 flush for every 2,000 keys**, turning the storage engine into an I/O-bound process.

**Occupancy Dynamics:**
The custom observability captured the "sawtooth" pattern of MemTable occupancy. In random large-value workloads, occupancy frequently hit >90%, followed by a sharp drop in `inserts/sec` (as low as 800 ops/sec) while the background flush thread worked to clear the memory.

### Conclusion
Experiment 1 provides a high-fidelity window into the internal mechanics of RocksDB. By tracking live insertion rates and occupancy, we can mathematically prove the relationship between data shape and storage engine performance. This visibility is essential for detecting "micro-stalls" and understanding the exact overhead of metadata in modern LSM-tree implementations.
