# Designing a Distributed Cache

## Single Node Cache Implementation

We aim to implement a cache similar to Redis, which typically runs on port 6379. The cache allows external machines to connect and communicate over a simple TCP connection to enable request-response mechanisms with guaranteed message order delivery. For example, a user can send a query to the system’s IP (e.g., `10.0.0.1`) on port `6379`.

The cache operates as an **in-memory system**, relying on a large **hashmap** for data storage. However, since memory (RAM) is limited, features like *eviction* and **time-to-live (TTL)** mechanisms are required to manage memory effectively.

## Communication Protocols

When designing a distributed cache that communicates with the external world, the way machines connect and exchange data is crucial. There are several options for enabling communication:

1. **Using HTTP:** This is the most common communication protocol, where data is passed in HTTP format. For example, if we want to retrieve the value of a key `k1` via an HTTP GET request, the request might look like this:

    ```bash
    GET /hello HTTP/1.1
    X-accept: application/json
    ```

    This request is sent in a string format, but HTTP includes additional metadata and headers with every request. Although we are simply retrieving the value for k1, the actual payload contains a significant amount of extra data, leading to network overload and slower processing due to parsing all of this metadata.

2. **gRPC:** Another option is gRPC, which is a high-performance RPC (Remote Procedure Call) framework used to streamline communication between services. gRPC also uses HTTP/2 as its underlying protocol. 

3. **Custom Protocol:** Instead of using HTTP, we can define a custom protocol for communication. A custom protocol allows us to create a more efficient format for requests and responses, reducing unnecessary metadata. By defining our own rules for how the server interprets client requests and responds, we can optimize communication for specific use cases.

To optimize communication, Redis developed its own protocol called R**ESP (Redis Serialization Protocol)**. RESP minimizes overhead by reducing the amount of extra metadata that HTTP adds. This results in much faster and lighter communication. Although HTTP is a valid choice, using RESP (or any other custom protocol) reduces payload size, parsing time, and overall network overhead, making it more efficient.

## Data Storage in the Cache

In our cache implementation using C, the hashtable will store key-value pairs. Here's how we handle the key and value:

- The key can be a simple string, represented as `char *` in C.
- Redis supports various data types for the value, such as `hyperloglog`, `bloom filter`, `string`, `int`, `set`, `sorted set`, and more.

1. **Question:** How can we support multiple data types in value using C?

    **Answer:**  We use `void*` for the value. The advantage of `void*` is that it allows us to store any data type and we can typecast it to any other data type dynamically, enabling support for various types like strings, integers, sets, etc.

2. **Question:** How do we know which data type to typecast `void *` to?

    **Answer:** We can store the data type information along with the value by wrapping them in a struct. The struct will contain the value as `void *` and a field for the data type (e.g., int, hyperloglog, bloom filter, string), allowing the cache to know how to handle and typecast the value appropriately. Example:
    ```
    struct {
        void *val;
        type int8/hyperloglog/bloom_filer/string;   // Type indicates what the value is (string, set, etc.)
    }
    ```
By using this structure, the cache can determine the correct type and perform typecasting, allowing it to support different types of values.

## Command Execution Flow

For a command like `SADD k v1` (to add `v1` to the set associated with key `k`):

1. The command is received.
2. The server retrieves the value for key `k` and checks its type.
3. If the type is not a set, an "invalid operation" response is returned.
4. If the type is correct, the value is typecast to a set, and `v1` is added to the set.

## Custom Memory Management in C

Since the cache is stored in RAM, which is limited, we need a way to determine when the cache has reached its maximum memory limit (e.g., 1GB). Two common approaches for monitoring memory usage are:

- Background process to query the OS
- Tracking memory manually inside cache

### Background process to query the OS

This method involves periodically retrieving memory usage from the OS. However, to retrieve data from the operating system, we must switch from user mode to kernel mode, fetch the data, and then switch back. This process requires system calls, which are costly because they cause interrupts, making them slow and resource-intensive. Thus, querying the OS frequently is inefficient.

### Tracking memory manually inside cache

A more efficient approach is to maintain the memory usage within the cache itself. This is done by introducing a `total_size` global variable, which is updated every time an insertion, deletion, or update is made. This allows the cache to accurately track memory usage in real-time without relying on costly system calls.

To maintain `total_size`, every memory allocation or deallocation must update this variable atomically. For instance:

- For a simple query like `SET USER:1:name arpit` how will we update the `total_size` variable? 
- Consider a query on set like `SADD group:1:member set("Arpit")`. How would we update the `total_size` in this case? The complexity increases because data structures like sets often store elements in trees, meaning we also need to account for the memory required to store pointers, not just the data itself.

How do we allocate or deallocate memory in C ?? The answer is `malloc`, `calloc`, and `free`. For example, when a node in a set or a string is allocated, we use `malloc` to allocate the necessary bytes and it returns a memory reference of type `void *`. For example, `void * malloc(int b)`, `b` is the number of bytes to allocate.

To track memory usage, every time we call `malloc`, we can also increment a `total_size` variable by the allocated number of bytes (`b`). However, doing this manually for every `malloc` call can be cumbersome. To simplify this, Redis wraps the `malloc` function with a custom function called `zmalloc`, which has the same signature but also handles updating the `total_size` variable internally. This ensures that memory usage is tracked efficiently without adding overhead to every memory allocation operation.

![](Pictures/1.png)

Similarly, Redis uses a `zfree` function to wrap `free`, which updates `total_size` by subtracting the released memory. The details of these functions can be found in Redis's source code (`zmalloc.c` and `zfree.c`). This internal tracking system ensures efficient and accurate memory management in the cache

By using these custom wrappers, Redis efficiently tracks memory usage without the need for expensive system calls, ensuring optimal performance.

**Question1**: How do we know how many bytes to allocate for each data type in C?

**Answer:** We use the `sizeof` operator to determine the exact amount of memory required for each data type. For example, when creating a tree node in C, we use `malloc(sizeof(struct node))`. This ensures we allocate the correct number of bytes, allowing us to keep the `total_size` variable updated and track memory usage accurately.

**Question2:** How do we ensure that our cache does not exceed the maximum memory limit?

**Answer:** Before allocating memory, we check if the current `total_size` plus the new memory to be allocated would exceed the maximum limit. If it does, we call the eviction mechanism to free up space by removing some keys until the required memory becomes availbale. This approach prevents the system from breaching the maximum memory limit.

![](Pictures/2.png)

### Eviction Techniques and Implementation in Cache

Two widely used eviction techniques are **LRU (Least Recently Used)** and **LFU (Least Frequently Used)**.

**Question:** How can we implement LRU in a hash table as an eviction policy?

**Answer:** To implement LRU, we need a linked list. Each node in this linked list should contain a pointer to the next node and the key, represented as a char *. The basic structure of a node could look like this:

```c
struct node {
    struct node *p;
    char *key;
}
```
This is the bare minimum, but additional metadata may be needed.

- **Size Calculation for Linked List Overhead:**

    Each node in the linked list contains two pointers (each 8B in size on a 64-bit system), so the total size of a node is 16B. For 1 million keys in the cache, we would need 16MB of RAM just to store the linked list. This overhead is significant since everything is stored in RAM.

- **Trade-offs:**

    The challenge is the overhead of metadata (linked list) for managing eviction. 
    - One alternative is random eviction, where no metadata is stored, and random keys are evicted. However, the downside is the potential eviction of keys that may be needed in the future. The benefit is using the extra memory (16MB in this case) for storing additional keys instead of metadata.

    - Another option is to implemented LRU without a linked list. Here we would need to iterate through each key and find out which was least recently used, which increases the time complexity for eviction.

- Three Considerations for any choice for system design:
    - **Space:** Linked list metadata consumes space.
    - **Time:** Eviction without metadata is slow. (Iterating through all keys)
    - **Correctness:** Random eviction risks evicting important keys.

- **Optimizing the Second Approach (random eviction):**

    Can we improve the 2nd approach without completely relying on randomness? We can, by introducing a sampling strategy while keeping space and time usage manageable. Instead of evicting a totally random key, we can take a sample of keys and evict those that are least recently used (LRU). This way, we're doing local optimization rather than global, focusing on a subset of keys that are good candidates for eviction.

    In this approach, we introduce a fixed space overhead with a **sample + eviction pool**. Redis implements this by maintaining an eviction pool with a fixed size of 15, regardless of the total number of keys. The eviction pool stores keys that are likely to be evicted. When the eviction condition is triggered using the LRU strategy, keys from the eviction pool are removed in sequence until enough memory is freed.

    To keep the eviction pool updated, Redis runs a background job periodically or during idle times. This job picks random keys from the hash table and adds them to the eviction pool, ensuring that the pool always contains the least recently used keys that the system has encountered.

    This approach offers a balance between space, time, and correctness, avoiding the full cost of randomness while still keeping eviction time and memory efficient.

### Implementing TTL (Time-to-Live) in Cache

**Question:** How can TTL (Time-to-Live) be implemented in cache?

**Answer:** Since we store the value as a struct, we can add a field for the expiration timestamp in the same struct. This timestamp is calculated by adding the current time to the TTL when the key is created or updated.

```c
struct {
    void *val;        // Pointer to the value stored (can be any data type, like string, set, etc.)
    int type;         // Indicates the type of the value (e.g., int8, hyperloglog, bloom_filter, string)
    time_t expire_at; // Timestamp indicating when the key will expire
};
```

Here hard deletion may seem costly but since everything is stored in RAM, it is not an issue. Hard deletion is costly when the data lies on disk. The main problem here is addressing expired keys that remain in memory but are never accessed. The challenge is identifying expired keys and removing them.

- Approaches for Removing Expired Keys:

    - **Approach-1:** When the eviction job runs, as discussed earlier, it will check all the keys, identify the expired ones, and remove them. However, this process is time-consuming and inefficient, as it involves scanning all keys to determine expiration.

    - **Approach-2:** Use a min-heap to store keys based on their expiration time. The key with the earliest expiration is at the top. When its timestamp is less than the current time, it gets removed. However, maintaining a min-heap requires significant memory.

    - **Approach-3:** We can run a background job at specified intervals, which randomly selects *n* keys and deletes those that have already expired. The value of n must be chosen carefully—if *n* is too large, the job will take longer to execute; if *n* is too small, the probability of selecting non-expired keys increases because non-expired keys outnumber expired ones. This can result in slower eviction of expired keys, as it becomes more likely to miss them. Balancing *n* is key to optimizing this process. This method avoids the memory overhead of a min-heap. 
    
    - **Approach 4:** Similar to sampling, we create a fixed-size priority queue regardless of the number of keys. A background job randomly picks a key, and if it’s expired, the key is deleted. If it’s not expired, the job attempts to insert the key into the min heap, which will hold the most likely keys to expire. While this approach improves upon the previous one, it introduces the overhead of managing the priority queue, making it less efficient. We often avoid this due to the extra complexity.
    
    - **Approach 5:** We can implement lazy deletion, where on each GET query, the cache server fetches the value (assuming the key exists), checks whether the key has expired, and if so, deletes it and returns a 404 response. If the key is still valid, it returns the value. The issue with this approach is that expired keys may remain in memory indefinitely, consuming space. If a key is never accessed after its expiration, it will persist forever in RAM.

    To address the limitations of **Approach 5**, we can combine it with either **Approach 3** or **Approach 4**. This combination optimizes the eviction process — **Approach 5** handles the majority of expired keys during read operations, while **Approach 3** or **4** takes care of any remaining expired keys in the background. While the challenges of **Approach 3** or **4** (sampling overhead, managing priority queues) still apply, these methods complement each other well. Given that caches are typically read-heavy, most expired keys will be cleaned up by **Approach 5**, with the remaining handled by the background process.

### Enhancing Cache Performance with Sampling and Probabilistic Theory

We can also improve cache performance using sampling and probabilistic theory. For example, if we sample 20 keys and find 4 that are expired but not yet deleted, this implies that 20% of the sampled keys are expired. We can infer that roughly 20% of the entire cache might also consist of expired keys. When this percentage exceeds a certain threshold, we can trigger a full cleanup job to scan the entire cache and delete all expired keys by doing a stop the world, ensuring optimal memory usage.