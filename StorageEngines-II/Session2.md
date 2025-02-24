# Key-Value Store with Fast Read, Write, and Delete Operations

In this class, we’ll be building a key-value store on hard disk that supports extremely fast reads, writes, and deletes, all while ensuring full persistence. The solution involves optimizing operations for high performance, even when dealing with large datasets stored on HDDs (hard disk drives).

## Basic Operations:
- **Put Operation:** Adds a key-value pair to the storage.

    - Example:
        ```scss
        put (k1,v1), (k2,v2), (k3,v3), (K1,v1’)
        ```

        The storage would look like:
        ```scss
        K1, v1
        K2, v2
        K3, v3
        K1, v1’
        ```

        In this setup, `k1` is stored twice, once with `v1` and once with `v1’`. We don't overwrite the old value directly because random writes to HDD are slow, so the system appends new data. The Get request always fetches the latest value.

- **Delete Operation:** Simulates deletion by adding a "tombstone" entry.

    - Example:
        ```scss
        DEL(K2) -> Put (k2, -1)
        ```
        This way, the storage looks like:

        ```scss
        K1, v1
        K2, v2
        K3, v3
        K1, v1’
        K2, -1
        ```

## Handling Multiple Entries for the Same Key:

- For keys that appear multiple times, we always retrieve the most recent value by scanning the file in reverse. This approach, however, can be slow, so we create an index stored in RAM for quick random access.

    - The index holds the latest location of each key. For `put` requests, we update the index, and for `delete`, we remove the key from the index.

## Limitations of the Basic Approach:

1. **File Size:** Over time, the file becomes too large.

2. **Stale Values:** Old values are still stored even after updates.

3. **RAM Constraints:** The index is stored in RAM, which limits the number of keys we can store. Each entry requires about 13 bytes (5 bytes for the key, 8 bytes for the pointer), meaning for 1MB of RAM, we can store a maximum of ~76,923 keys.

## Improving File Management:

- **Sequential I/O Optimization:** HDDs perform better with sequential I/O compared to random I/O due to reduced head movement.

- **Compaction:** To prevent the file from growing too large, we can periodically run a **clean-up thread** to remove stale entries. This thread creates a new file containing only the latest values, but new data might still be added to the original file during the process, creating slight staleness.

- **Immutable Files:** To solve the issues with continuous appending and cleanup, we use immutable files. Once a file reaches a certain size (e.g., 1GB), it becomes read-only, and new writes are redirected to a fresh file. Periodic merge and compaction operations remove stale data across all immutable files, keeping storage optimized.

## Index Structure:

- The index now stores both the offset and file descriptor for each key. After a merge, the index must be updated, requiring a stop-the-world update to ensure consistency.

## Merge and Compaction Strategy:
- The frequency of merging depends on the system's put and delete operations, as well as the ingestion rate (average size of key-value pairs). When storage utilization reaches a threshold (e.g., 80% of disk capacity), merge and compaction are triggered to free up space.

## Efficient Reads:

- For a **GET** request, the system uses the index to find the file descriptor and offset. To optimize the number of system calls, each record in the file contains:

    - CRC (Cyclic Redundancy Check) for data integrity
    - Timestamp (TS)
    - Key Size (KSZ)
    - Value Size (VSZ)
    - Key
    - Value

- The index holds the **record size (RSZ)** along with the offset and file descriptor, allowing the system to read the entire record in one system call.

## Handling Data Corruption:
- If the data becomes corrupted, the system uses CRC to detect and flag the issue, ensuring that no incorrect data is returned to the user.

## Conclusion:

This approach resembles the **Bitcask** database model, one of the fastest embedded databases used in production systems like Uber. By leveraging techniques such as immutable files, merge/compaction, and efficient indexing, we achieve a scalable and high-performance key-value store.