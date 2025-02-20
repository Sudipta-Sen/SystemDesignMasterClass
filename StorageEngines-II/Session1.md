# Designing a Scalable Word Dictionary Using a File-Based Approach

## Problem Overview:

We are tasked with designing a scalable word dictionary that can return the meaning of words via an API call. The dictionary must be contained within a single file, and words/meanings will be updated weekly. The dictionary file will be 1TB in size, containing approximately 170,000 words. The solution needs to be efficient in handling large datasets and offer reasonable response times.

- Key Requirements:
    - **Dictionary stored in one file** (1TB size, 170,000 words, no repetitive entries).
    - **Scalability:** Dictionary needs to scale for large datasets, potentially containing millions of words.
    - **Portability:** The dictionary should be contained within a single file, making it easy to move.
    - **Weekly Updates:** Every week, a changelog containing new words, updates, or deletions will be applied.
    - **Efficient Search and Update Operations:** The design should support quick lookup of words and allow updates without significant overhead.

## Approach 1: Linear Search

For a GET request with a word like "apple," we would search the file line by line until we find the word, then return the meaning. This approach, however, has a linear time complexity (O(n)), meaning that for a 1TB file, the average case would still require scanning 0.5TB of data. This is not efficient, especially for large files.

## Approach 2: Indexed Search with HashMap

To improve search performance, we can implement an indexed search by maintaining an index of words and their corresponding offsets within the file. This would allow us to perform a lookup using the index, greatly reducing search times.

- Index Structure:
    - The index will store words and their offsets (starting positions) in the file.

    - We can use a hashmap to store the index in memory for fast lookups.

- Calculating Index Size:
    - **Word Size:** The average length of an English word is ~4.7 characters (~5 bytes).

    - **Offset Size:** Each offset is an 8-byte integer (64-bit), which can support a file size of 1TB.

    - **Index Entry Size:** Each index entry would take ~13 bytes (5 bytes for the word, 8 bytes for the offset).

    - **Total Index Size:** For 170,000 words, the index size would be ~2.2MB (13 bytes * 170,000 words).

    We can store the index in RAM for fast lookups or persist it on disk to avoid reloading it every time the server starts. Persisting the index also avoids the need to recreate it during startup.

## File Structure: Embedding the Index in the Data File

Since we are constrained to using a single file, we can append the index to the dictionary file itself. Here’s the design:

1. **Header:** The first part of the file contains a fixed-size header with metadata.
    - First 4 bytes: Length of the index.
    - Next 4 bytes: Version information.
    - Last 8 bytes: Reserved for future use

2. **Index:** Immediately after the header, we store the index, which includes words and their offsets.

3. **Data:** The actual dictionary data (words and their meanings) is stored after the index.

- Loading the File:
    - Step 1: Load the 16-byte header.
    - Step 2: Read the index length from the first 4 bytes of the header.
    - Step 3: Load the index into a hashmap in memory.
    - Step 4: Start the web server to handle API requests.

## Handling Requests:

### GET Requests:

1. The API server first checks if the word exists in the in-memory index.
2. If the word exists, the server fetches the corresponding offset from the index.
3. It reads the word’s meaning from the file using the offset and returns it to the user.
4. If the word does not exist in the index, it returns a 404 Not Found.

### Update requests:

Each week, a changelog will update the dictionary:

- **New Words:** New entries can be appended to the dictionary file, and the index is updated accordingly.
- **Updated Meanings:** Existing words can be updated by modifying their associated meanings in the file.
- **Deleted Words:** Words to be deleted are simply removed from the index, though the data in the file remains.

## Alternative Approach: Content-Addressable File System

If the constraint of using a single file were removed, we could use a content-addressable file system, similar to how Git works. Each word would be stored in its own file, with the file name as the key (word) and the content as the value (meaning).

Advantages:
- **Efficient Lookups:** For a GET request, we can directly access the file for the word.
- **Easier Updates:** Updating or deleting a word would involve modifying or removing a single file.
- **Simplicity:** This approach avoids the need for a large index and can scale easily.

Disadvantage:
- **File Overhead:** Storing individual files for each word would introduce additional overhead in terms of file system management and storage.

## Scalability Considerations: Multiple API Servers

- **Multiple API Servers:** For large-scale applications (e.g., Oxford University data), a single API server may not be sufficient to handle all requests. To scale horizontally, we can deploy multiple API servers, each with a copy of the dictionary.

- **Data Synchronization:** Since the dictionary is updated only once per week, staleness is not a significant concern. However, we need to ensure that all API servers are synchronized after every update.

- **Cost Optimization:** To reduce infrastructure costs, we can use network-attached storage (NAS) or other storage solutions like HDFS or S3. The API servers would query this central storage system to fetch data, which allows for more lightweight servers.

## Storage-Compute Separation:
- **Storage:** The dictionary is stored in NAS, HDFS, or S3, which supports file operations such as reading from specific offsets.

- **Compute:** API servers are lightweight and can scale independently, accessing the dictionary via network calls. This reduces the cost of maintaining large servers.

## API Server Workflow:
1. The API server boots up and loads the header and index from the storage.

2. It loads the index into memory.

3. It starts the web server and begins serving requests.

4. For each request, the server checks if the word exists in the index and fetches the meaning from the storage.