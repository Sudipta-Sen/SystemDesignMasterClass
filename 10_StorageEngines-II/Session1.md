# Designing a Scalable Word Dictionary Using a File-Based Approach

## Problem Overview:

We will design a scalable word dictionary that can return the meaning of words via an API call. The dictionary must be contained within a single file, and words/meanings will be updated weekly. The dictionary file will be 1TB in size, containing approximately 170,000 words. The solution needs to be efficient in handling large datasets and offer reasonable response times.

- Key Requirements:
    - **Dictionary stored in one file** (1TB size, 170,000 words, no repetitive entries).
    - **Portability:** The dictionary should be contained within a single file, making it easy to move.
    - **Weekly Updates:** Every week, a changelog containing new words, updates, or deletions will be applied.
    - **No dupliate entry:** The dictionary must ensure that no duplicate entries exist for any key. Each key should be unique.

    - **Efficient Search and Update Operations:** The design should support quick lookup of words and allow updates without significant overhead.
    - Every week, the dictionary will be updated with a change log that may include:
        - New words along with their meanings.
        - Existing words that may receive updated meanings.
        - Words that may need to be removed from the dictionary.
    
    ![](Pictures/1.png)

## Alternative Approach by relaxing constraint 

If the constraint of using a single file were removed, we can consider creating a separate file for each word, where the file name serves as the key, and the file's content contains the corresponding meaning of the word. This is called **content-addressable file system**, similar to how Git works.

Advantages:
- **Efficient Lookups:** For a GET request, we can construct the file path using the key, directly read the file, and return the response if the file exists. If the file doesn't exist, we return a 404 error.

- **Easier Updates:** Updating or deleting a word would involve modifying or removing a single file.
- **Simplicity:** This approach avoids the need for a large index and can scale easily.

Disadvantage:
- **File Overhead:** Storing individual files for each word would introduce additional overhead in terms of file system management and storage.

## Approaches for Optimizing GET Requests: Pros and Cons

Since we cannot use a traditional database, we need to design our own. However, instead of creating a fully-fledged database, we'll build one that efficiently solves the specific problem at hand.

Our word dictionary will be a simple text file, which we'll call `data.dict`. This file will contain entries like words and their meanings.

For example, a typical entry in `data.dict` might look like this:

![](Pictures/2.png)

What would a GET call look like? For example, if we receive a GET request for the word "apple," our system needs to search the dictionary and return the corresponding definition: "a fruit."

### Approach 1: Linear Search

For this GET request, we would search the file line by line until we find the word, and then return its meaning. However, this approach has a linear time complexity (O(n)), meaning that for a 1TB file, in the average case, we would still need to scan 0.5TB of data. This is inefficient, especially for large files. Additionally, there’s a risk that the word might appear within the definition (message, meaning) part, making it challenging to distinguish between the key and the value.

### Approach 3: Binary Search

 We can implement binary search in place of linear seach to improve the efficiency of searching for data. While binary search in a file can be challenging, it is possible with careful implementation. Try experimenting with binary search on a file to explore this solution further.

### Approach 3: Indexed Search with HashMap

To improve search performance, we can implement an indexed search by maintaining an index of words and their corresponding offsets within the file. This would allow us to perform a lookup using the index, greatly reducing search times.

- Index Structure:
    - The index will store words and their offsets (starting positions) in the file.

    - We can use a hashmap to store the index in memory for fast lookups like below,

        ![](Pictures/3.png)

- Calculating Index Size:
    - **Word Size:** The average length of an English word is ~4.7 characters (~5 bytes).

    - **Offset Size:** Each offset is an 8-byte integer (64-bit), which can support indexing a file size of 1TB.

    - **Index Entry Size:** Each index entry would take ~13 bytes (5 bytes for the word, 8 bytes for the offset).

    - **Total Index Size:** For 170,000 words, the index size would be ~2.2MB (13 bytes * 170,000 words).

    We can the index itself can be kept either in disk or RAM. However, if we don't persist the index on disk, the index would need to be recreated every time we load the dictionary, significantly increasing the startup time of the API server. To avoid this overhead, we can store the index on disk, allowing us to quickly load it into memory and begin serving requests when the server starts.

    Given the constraint that the dictionary must be stored in a single file, the challenge is how to persist the index within the same file as the dictionary itself.

#### File Structure: Embedding the Index in the Data File

- **Marker Based Approach:**

    A potential approach is to append the index to the data file itself and use a marker to identify where the index ends and the actual data (dictionary) begins. However, storing the index after the data would require scanning the entire file to locate the end marker, which would increase loading time. Instead, we can place the index at the beginning of the file, followed by a marker indicating the end of the index. This way, we only need to read the index and skip loading the actual data until needed.

    However, there is a challenge with using markers — if the chosen marker string appears inside a key, it may lead to incorrect identification of the index's end. 

    Also the marker-based approach (even if the marker never appears in a key) requires us to dynamically increase the size of the character array(index) as we progress through the read operation. At each step, we also need to check if the marker has been encountered, which adds overhead and increases the loading time.

- **Fixed Length Header:** 
    
    To avoid these issues, instead of relying on a marker, we can store the exact size of the index in the header. This way, we know exactly how many bytes to read to load the entire index. The process would follow this flow:
    1. Load the header.
    2. Retrieve the index length from the header.
    3. Read the index and load it into memory.

    But how do we determine the size of the header? Do we add another header to describe the length of the next one? Absolutely not! The header has a fixed length in all cases, so we know exactly how many bytes to read. We can structure these bytes to contain specific information.

    For example, let's use a 16-byte header. In this 16-byte structure,
    - First 4 bytes: Length of the index.
    - Next 4 bytes: Version information.
    - Last 8 bytes: Reserved for future use

    This makes the header-loading process simple — just read the first 16 bytes.

    The index starts at byte 17, and we can determine where it ends by reading the length of the index from the first 4 bytes of the header. The actual data (dictionary) begins right after the index. By reading the entire header in one system call, we can efficiently load the index into a character array of exactly the required size, improving speed in contrast of marker-based approach where at each step, we need to check if the marker has been encountered.

    When our API server boots up, it performs a well-optimized startup process.
    - Step 1: Load the 16-byte header.
    - Step 2: Read the index length from the first 4 bytes of the header.
    - Step 3: Load the index into a hashmap in memory.
    - Step 4: Start the web server to handle API requests.

    Now how the API server handles a GET request - 

    1. The API server first checks if the word exists in the in-memory index.
    2. If the word exists, the server fetches the corresponding offset from the index.
    3. It reads the word’s meaning from the file using the offset and returns it to the user.
    4. If the word does not exist in the index, it returns a 404 Not Found.

## API Server Configuration and Handling Read Requests

Given the scale of handling data as large as Oxford University's dictionary, a single API server is insufficient to manage all read requests effectively. In this case, multiple API servers are required.

- Key Considerations:
    1. Configuration Needs:

        -  Since each API server holds the entire dictionary and serves data from the disk on each request, the server should be equipped with high-speed SSDs for fast data retrieval.

        - To handle the index efficiently and serve requests quickly, each API server requires high RAM capacity to store the index in memory.
    
    2. Scaling and Infrastructure Costs:

        - As traffic increases, more API servers will be needed, each containing a full copy of the dictionary. However, this approach leads to high infrastructure costs as every server will need the same storage and computing power.

            ![](Pictures/4.png)

        - To reduce these costs, we can adopt a model where API servers do not store the full dictionary themselves. Instead, we can use **network-attached storage** (NAS) or a similar distributed storage system. In this model, API servers will query the central storage for data.

    3. Staleness and Data Synchronization:

        - One of the primary concerns with having multiple API servers is data synchronization. However, since the dictionary database is updated only once a week as per our prerequisites, the risk of data staleness is minimized. All API servers can operate with the same copy of the data during the week without the need for real-time synchronization, ensuring consistency across the system.

- Moving to a Storage-Compute Separation:

    By separating storage and compute, the dictionary is stored in a centralized network storage, and API servers only query the storage for the required data and does not contains any data within its own disk. This approach allows API server to be lightweight, so we can use low-configuration servers for the API, significantly reducing costs.

    - **Storage:** The dictionary is stored in NAS, HDFS, or S3, which supports file operations such as reading from specific offsets.

    - **Compute:** API servers are lightweight and can scale independently, accessing the dictionary via network calls. This reduces the cost of maintaining large servers.
    
    ![](Pictures/5.png)

    - Storage Requirements:

        - The storage system should support file access over the network.
        
        - It should have **key-value** semantics and support operations like reading specific **offsets** and fetching data in **byte ranges**.

        Systems such as NAS, HDFS, and S3 (with byte-range fetch capabilities) are ideal for this setup.
    
    - Benefits of Storage-Compute Separation:

        1. **Independent Scaling:** Compute nodes (API servers) can scale horizontally based on traffic without duplicating storage. All servers refer to the same centralized storage, which can also scale independently if needed.

        2. **Cost Efficiency:** While there is a network call involved in accessing data, the cost savings from using lightweight servers and centralized storage are substantial. Furthermore, this ensures all API servers have a **consistent view of the data**.
    
    This architecture ensures the system can handle high traffic efficiently while keeping infrastructure costs under control

    - So the API Server Workflow becomes:

        1. The API server boots up and loads the header and index from the storage.
        2. API server loads the index into memory its own memory.
        3. It starts the web server and begins serving requests.
        4. For each request, the server checks if the word exists in the index. 
            - If the word is not found in the index, it returns a 404 error. 
            - Otherwise the server retrieves the corresponding byte location from the index, reads the meaning from that position from network storage and sends it back to the user.

## Handling Updates and Writes in a Dictionary API

Now that we have sorted out the read operations for our API server, let’s shift focus to write operations, specifically handling weekly updates. As illustrated in below picture, we have a `data.dict` file representing the current dictionary, while the `changelog` contains updated meanings for existing words, as well as new words and their meanings. 

![](Pictures/6.png)

### Challenges in Updating an Entry:

Let's explore the challenges involved in updating a word, say "ball".

1. **Space Management for Updates:**

    Suppose the updated meaning for **"ball"** requires 10 more bytes than the existing entry. To accommodate the new meaning by overwriting the old one, we would need to right-shift every byte after the modified key by 10B to create space. The complexity of this operation is high because, in the worst case, it might involve shifting all the bytes in the dictionary, which could be up to 1TB in size. On average, we might shift 0.5TB per word. If the changelog contains 100 words, this could result in 50TB of disk I/O—a significant cost in performance. After each update, we would also need to update the index to reflect the new positions.

    Overwriting the new meaning without shifting might overwrite parts of the next word. In an IDE, it seems effortless to hit "enter" and insert a new line of code, allowing us to modify the content seamlessly. However, this is because the file we are editing resides in RAM, where inserting new content is straightforward. In contrast, our dictionary is stored on disk, and performing such in-place modifications in disk is not as simple. Unlike RAM, disk storage doesn't easily accommodate inserting new data in the middle of a file, making such changes more complex and time-consuming.

2. **Insertion of New Words:** 
    
    For new entries, we would need to shift bytes after the insertion point, similarly creating inefficiency.

3. **Index Updates:**

    After such operations, we would also need to update the index to reflect new positions.
    

### Improvement with Merge Sort:

Instead of right-shifting, a more efficient approach is to use merge sort:

- Both `data.dict` and the `changelog` are sorted, allowing us to merge them into a new `data.dict` file.

- The time complexity is `O(n+m)`, where `n` is the number of entries in `data.dict` and `m` is the number of entries in the changelog.

- We will iterate through all the entries in data.dict, and one might wonder where the improvement lies. The key difference is that in this approach, we iterate through the entire dictionary only once for the entire changelog, regardless of how many entries it contains. In contrast, with the previous approach, we would have to shift bytes (on average 0.5TB) for each individual entry in the changelog. This reduces the overall I/O operations significantly.

- How does merging work?

    We use a two-pointer approach. If both the dictionary and changelog contain a particular key, we take the updated entry from the changelog; otherwise, we take whichever file has the relevant data.

![](Pictures/7.png)

### **Weekly Update Process:**
    
Suppose this merging happens every Sunday. The process would be as follows:
1. Spin up a weekly update server.
2. Fetch the changelog.
3. Download the actual dictionary file.
4. Perform the merge operation.
5. Upload the latest dictionary file to the database and delete the older version.

### **Handling Index and API Server Synchronization:**

Where do we upload the updated file in the database? Do we simply replace the older one? The challenge is that the index for existing, unchanged words may shift in the new data file. As a result, the API server needs to be aware of these changes to load the updated index; otherwise, it could serve incorrect/garbage data. So, how does the API server detect that the data has been updated?

![](Pictures/8.png)

- **Pull-based Approach:**

    Who will the API server poll? The weekly update server? That’s not feasible because the weekly update server only spins up once a week and doesn't run 24/7.

    Instead, we can leverage a version field in the header. The API server checks the current version of the file in storage and compares it to the version in the header it has. If there’s a mismatch, it reloads the index.

    However, how frequently should the API server check for updates? Checking too often wastes bandwidth, while checking too infrequently could result in the server serving outdated or incorrect data for an extended period. Additionally, if we update the dictionary file with a new name, how will the API server detect that change? 
    
- **Push-based Approach:**
        
    The update server pushes notifications to API servers. However, some servers may receive the update while others might not due to failures. To avoid this, a new file name can be used. Servers that get the update will read from the new file, while others will continue reading from the old one. So some API servers may serve stale data temporarily, but they will not serve incorrect or corrupted data. However, the challenge is how the servers reading the old file will eventually switch. To address this, we could think of using a pub/sub model, but that increases infrastructure costs significantly for just a weekly update.

    ![](Pictures/9.png)
    
    - **The Solution - Meta.json File:**

        To address these challenges, we can implement a two-pointer or symbolic link approach using a `meta.json` file. This file will contain the storage path of the latest dictionary file. Whenever a new API server starts up, it will read the `meta.json` file to obtain the path of the latest dictionary file, access that path, read the file header, load the index, start the web server, and begin serving requests.

        **Handling Old API Servers:** The question arises: How do the older API servers know when the meta.json file has been updated? Should they poll the `meta.json` file at regular intervals? Polling frequently would be inefficient since the `meta.json` file is updated only once a week. Even if we set the polling interval to once per day, there's a chance that the `meta.json` file could be updated right after the server polls it, leaving the server to continue serving outdated data for another day.

        While this approach solves the problem for new API servers (which will always load the latest data), older servers will continue to serve slightly **outdated data** — though not garbage or corrupted data. To fully address this, we need to replace older API servers with new ones, which can be done through a rolling update deployment strategy.

        **Rolling Update Process:** Once the weekly update server finishes uploading the new dictionary file and updates the `meta.json` file, it can call a `Jenkins.deploy()` operation. This will initiate a rolling update that deletes the old API servers and spins up new ones. These new servers will then read the updated `meta.json` file, access the new storage path, and start serving the latest data.

        This solution is much more efficient than managing the IP addresses of all API servers and manually triggering reloads. It ensures a smooth and automated process for keeping all API servers up-to-date.

        ![](Pictures/10.png)

## Handling Deletions in Changelog:
        
The changelog can also contain deletion markers. We can use markers like 1 for new words, 2 for updated words, and 3 for deleted words. While merging, any word with a 3 marker will be skipped, effectively removing it from the new dictionary.

This approach optimizes the update process while keeping the API servers in sync with the latest data efficiently and cost-effectively.

## Building a Pointed Query System with S3

We have not just built a dictionary; we’ve created the ability to perform targeted data queries on any kind of storage. This isn’t a general query system, but rather a pointed read mechanism—given a specific word, return its meaning. So, where does this kind of system find application?

Consider a scenario where we have key-value pairs that are infrequently accessed. For frequently accessed data, we can store it in fast, low-latency databases like DynamoDB. However, for infrequent data, we can store it on S3, which is more cost-effective. This approach allows us to build a **multi-tiered solution**.

For example, imagine an e-commerce platform like Amazon. Recent orders (which are frequently accessed) can be stored in a fast database like DynamoDB or MySQL. Meanwhile, orders older than three months—rarely accessed—can be archived in S3. In this case, given an order ID, we can retrieve the details from S3 when needed. This way, we optimize storage costs while maintaining the ability to access historical data.

This foundational principle is also used by big data tools like Hive, Presto, and Pig, which are designed to query large datasets efficiently using similar mechanisms. By leveraging S3 for less frequently accessed data, we can significantly reduce infrastructure costs without sacrificing data accessibility.

![](Pictures/11.png)


