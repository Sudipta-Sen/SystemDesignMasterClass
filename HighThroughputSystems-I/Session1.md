#  LSM Trees and Bitcask Improvement through Write Optimization

## Introduction

We will discuss the design of LSM trees and how they can solve certain problems that arise in databases optimized for write-heavy operations. Our focus is on improving upon the Bitcask storage model by leveraging LSM trees, with a primary aim to enhance write performance while balancing durability, memory constraints, and read efficiency.

- **Quick Recap of Bitcask:** 
    Bitcask follows a simple design with:
    - A single append-only write file and multiple read files.
    - An in-memory index for quick lookups.

While effective for certain workloads, Bitcask can be slow in write operations as every write directly impacts the disk, which is relatively slow. Our goal is to improve this by introducing a buffering mechanism in RAM, enabling faster writes, batch processing, and optimizing disk I/O.

## Problem Definition: Can We Make Bitcask Better?

To improve Bitcask, we want to address the problem of **slow disk writes**:

- **Solution:** Use a RAM buffer to store writes temporarily and periodically flush them to disk in batches. This method optimizes both user experience (faster writes) and system performance (more efficient disk writes).

- Validation: Is This a Real Problem?

    Before jumping into a solution, it is crucial to validate whether users face performance issues during writes. The users of Bitcask are the best source for feedback on whether slow writes are problematic and worth solving.

## Key Challenges and Trade-offs

1. **Durability:** When data is stored in RAM temporarily, there’s a risk of data loss if RAM malfunctions before it is flushed to disk.
    - **Solution:** Write-Ahead Logging (WAL) can be used to mitigate this risk, though it introduces another append-only disk write.

2. **Memory Limitations:** RAM has a fixed size, meaning we must manage storage carefully until the data is flushed to disk.
    - We need a strategy to avoid running out of RAM while ensuring that the writes are as fast as possible.

## Write Optimization: RAM Buffering

The basic write optimization strategy involves:

1. Writing data into a RAM buffer.
2. Periodically flushing the RAM content to disk in batch mode for faster write operations.

To avoid blocking writes during flushes:
- Use a **dual-buffer strategy:** When flushing, a new buffer is allocated for incoming writes while the old buffer is flushed to disk.

### Durability with Configurable Options
For users concerned about durability, offer a configurable trade-off:

- **High-speed writes:** Users can prioritize write speed, understanding that some data loss (up to 1 minute of data) may occur if RAM fails.

- **Full durability:** Users who prioritize data integrity can opt for slower writes with guaranteed durability, using WAL for extra protection.

### Buffer and Operations

- The buffer is essentially a hashmap that stores key-value pairs.
- Operations on the buffers

    - **PUT(k, v):** Updates or adds the key-value pair to the hashmap.
    - **DEL(k):** Marks keys for deletion by writing a special value like -1 to signify that it should be deleted during flush.
    - **GET(k):** First checks RAM, and if the data isn’t present, retrieves it from disk.

## Optimizing Reads and Flushes

While flushing data to disk, we avoid updating existing disk files directly because it’s slow. Instead, a new file is created for each flush, allowing for quick writes. On read operations:

1. **Indexing:** An in-memory index (memtable) for each file helps locate keys efficiently.
2. **Compaction:** Multiple files are merged periodically to remove stale or deleted entries, a process known as compaction.

### Read Optimization with Bloom Filters

- Bloom filters are used to improve read performance by efficiently determining if a key might exist in a file.
- In a GET operation, the system first checks the RAM buffer. If the key isn’t there, it checks the Bloom filter before going to disk, reducing the number of disk accesses needed.

## LSM Tree Structure

We employ a two-level Log-Structured Merge (LSM) Tree:

- **Level-0 (RAM):** Newly written key-value pairs reside in RAM.
- **Level-1 (Disk):** Periodically, the contents of RAM are flushed to disk as files. A background thread merges files from Level-1 to create more compact representations in Level-2.

For writes, this structure offers **fast RAM writes**, and the most recent data is quickly accessible in Level-0. For reads, we start from the top (Level-0) and move down through the levels if necessary.

## Read and Write Trade-offs in LSM vs. Bitcask

1. **Write Performance:** LSM trees outperform Bitcask because writes initially go to RAM rather than disk.

2. **Read Performance:** LSM trees introduce potential slowdowns for reads, as stale data may exist in multiple levels, requiring more complex lookup logic. This can be mitigated with Bloom filters and careful indexing.

LSM trees favor **write-heavy systems**, while Bitcask might be better suited for read-heavy systems.

## Conclusion: Benefits of LSM Tree

- **Faster writes:** Recently written data resides in RAM, improving performance compared to Bitcask, which writes directly to disk.

- **Scalability:** LSM tree allows holding more keys in RAM and offers better management of large datasets through compaction and multiple levels.

LSM trees (as seen in implementations like RocksDB) provide a flexible, write-optimized solution for databases, while Bitcask and B+ trees might serve better in read-heavy environments. Each design has trade-offs, and the choice depends on system requirements.