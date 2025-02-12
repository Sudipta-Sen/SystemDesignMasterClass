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

Similarly, Redis uses a `zfree` function to wrap `free`, which updates `total_size` by subtracting the released memory. The details of these functions can be found in Redis's source code (`zmalloc.c` and `zfree.c`). This internal tracking system ensures efficient and accurate memory management in the cache

By using these custom wrappers, Redis efficiently tracks memory usage without the need for expensive system calls, ensuring optimal performance.