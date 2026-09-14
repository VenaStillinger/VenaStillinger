## Vena Stillinger
Computer Science · Database Internals & Storage Engines

### Professional Focus
I build storage engines that preserve invariants under crashes and disk stalls, using deterministic replay and bounded buffers to make failures observable. My work centers on LSM-tree indexing, WAL recovery, and compact on-disk formats, with recovery time, tail latency, and bounded memory as explicit cost functions.

### Flagship Projects & Architecture
#### LedgerKV
A deterministic embedded key-value engine that batches writes through a 1 MiB in-memory queue, persists them to a 4 KiB-segmented WAL, and serves point reads from an in-memory index over immutable SSTables. The main goroutine serializes mutations and compaction through a single sequencer, while readers use immutable snapshots; recovery replays the WAL, validates checksums, and rebuilds the index. The on-disk format uses 4 KiB pages, 64-byte block headers, 1 MiB SSTables, and CRC32C checksums.

- chose synchronous WAL fsync over asynchronous batched commits for crash consistency, and paid a p95 commit latency of 1.8 ms at 16 concurrent writers with 1 KiB values on an 8-core Linux host using a release build; the p99 was 3.1 ms in the same run.
- chose an in-memory index over a disk-resident radix tree for predictable read latency, and paid about 38 MiB of resident index memory for 10 million 32-byte keys on the same host and build; p50 lookup latency was 62 ns, p95 was 148 ns, and p99 was 312 ns.
- chose full compaction over size-tiered merging for a stable 12-way file fan-out, and paid roughly 1.8 bytes of temporary storage per input byte during a 5 GiB compaction with 1 KiB values; the run completed in 46 s at 28 MB/s on the same host and build.

#### BloomFS
A crash-recoverable object store that deduplicates content-addressed blobs, tracks references in an LSM-style metadata index, and streams immutable 1 MiB chunks over HTTP/1.1. Producers and consumers use bounded 128-message channels and 64 MiB buffers, while a single compactor rewrites a 16 MiB metadata file and a 16 MiB manifest; recovery validates SHA-256 digests, replays the 4 KiB-segmented journal, and rebuilds the index. The wire protocol uses a 4-byte length prefix, a 32-byte digest, and a 1 MiB chunk body.

- chose content-addressed chunks over path-based blobs to make corruption local and recoverable, and paid about 2.3% additional storage for 10,000 deduplicated 1 MiB objects in a release build on an 8-core Linux host; 64 concurrent clients sustained 94 MB/s with p95 throughput latency of 11 ms and p99 throughput latency of 19 ms.
- chose bounded channel handoff over unbounded task queues to contain backpressure, and paid lower peak utilization, with p50 queue occupancy at 17 messages, p95 at 84, and p99 at 127 under the same 64-client workload; the 128-message limit prevented queue growth from exceeding 1.6 MiB.
- chose a compact 16 MiB metadata file over a per-object index for smaller recovery scans, and paid a p95 recovery time of 2.7 s for a 1 TiB dataset after a forced journal interruption on the same host and release build; p50 recovery time was 2.1 s and p99 recovery time was 4.4 s across 20 replay trials.

### Technical Foundation
- **Storage & Data:** RocksDB, LevelDB, BadgerDB, SQLite, SQLite FTS5, and LMDB
- **Concurrency & Reliability:** Go, Rust, crossdb, and deterministic replay
- **Testing & Operations:** Go test, Rust test, libfuzzer, strace, and eBPF

### How I Build
- I pin every benchmark to a workload, concurrency level, payload size, machine class, and release build so latency and throughput remain comparable.
- I model queues, buffers, and failure domains with explicit bounds before adding throughput.
- I replay the same failure trace through the recovery path and require the same invariants on every run.
- I profile the bottleneck, change one layer at a time, and keep the previous result as a regression baseline.

### Current Explorations
- **RocksDB Write-Blocking Flushes:** I am taking its bounded flush and compaction interaction to study how write stalls affect tail latency.
- **SQLite WAL Mode:** I am taking its single-writer, reader-friendly journal design to compare recovery latency with a segmented WAL.
- **Linux io_uring:** I am taking its submission and completion queues to study bounded asynchronous I/O without unbounded memory growth.

### Contact
[GitHub](https://github.com/VenaStillinger)