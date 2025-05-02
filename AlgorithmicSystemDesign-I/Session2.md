# Dropbox Remote File Sync

Dropbox's file sync system, one of its key features that drove its popularity in 2008-09, allows **seamless synchronization of files across multiple devices** (Windows, macOS, Android, etc.) by keeping a special folder (the “Dropbox folder”) in sync with the cloud. Let’s break this down step-by-step.

## Goal

Keep a common folder synchronized across all devices linked to a Dropbox account. Any changes (create, update, delete) made on one device should reflect on all others with efficient bandwidth usage and fast updates.

## Core Components

1. **Dropbox Client:** Installed on each device. Watches a designated folder for changes.
2. **FSNOTIFY:** Filesystem-level notification API to detect changes in real time (additions, deletions, modifications).
3. **Cloud Storage (e.g., S3):** Stores file chunks.
4. **Metadata DB:** Stores file paths, chunk references, and versioning info.

## Workflow Overview

1. Detecting File Changes (FSNOTIFY)

    When a file (e.g., video.avi) is added/modified:
    - Dropbox client detects it using FSNOTIFY.
    - It breaks the file into chunks.

2. Chunking the File

    - Chunk size = Max 4MB
    - E.g., A 14MB file is split into 4 chunks: 3 of 4MB, 1 of 2MB
    - SHA256 hash is calculated for each chunk → used as Chunk ID
    - Chunks are content-addressed (like Git):
        ```java
        Chunk ID = SHA256(chunk_data)
        ```

3. Uploading Chunks to Cloud (Resumable Upload)

    - Requirements:
        - Upload only the chunks that don’t exist.
        - Avoid re-uploading if upload partially failed earlier.
    
    - Strategy:
        - Client calls API → “I want to commit file /video.avi with blocklist: [h1, h2, h3, h4]”
        - Meta Server (API) checks S3 for which chunks are missing
        - Meta Server responds with missing chunks list
        - Client uploads only those missing chunks using presigned URLs
        - Once all chunks are uploaded, client calls API again
            - If all chunks exist → entry is made in DB (commit)
    
        This ensures resumable uploads and avoids duplicate uploads.

4. Metadata DB Design

    - Table Schema
        - `account_id:` User account ID
        - `path:` File path relative to Dropbox folder (e.g., /exam/random.avi)
        - `blocklist:` Ordered list of chunk IDs (e.g., h1,h2,h3)
        - `version_id(vid):` Monotonically increasing version number per user
        - `timestamp:` File update time if not using vid

    - Path is logical, not physical.
    - Chunk names are not tied to file names, making rename efficient.
    - Storing hash in blocklist allows detecting file corruption and deduplication.

## Sync Across Devices

#### How do other devices know if a file changed?

- Every client keeps track of latest vid it has synced.
- Periodically, client polls Meta Server:
    ```arduino
    "Give me entries after vid = X for account_id = Y"
    ```
- New entries (with higher vid) are sent.
- Client compares chunk hashes with local ones to:
    - Download only new chunks
    - Reconstruct file from chunks

####  Lazy vs Eager Download
- Eager: All updated chunks downloaded immediately.
- Lazy: Only metadata pulled; chunks are downloaded when user accesses the file (0-byte placeholder). 
- For lazy load:
    - When user clicks a file → download chunks → reconstruct file → temporary cache → serve to user

####  File Versioning

Instead of updating DB row, insert a new row for every file update.

| version_id |	path	| blocklist	| account_id |
|---|---|---|---|
| 12	| /video.avi |	h1,h2,h3,h4	| A123 |
| 13 |	/video.avi |	h1,h2,h3',h4 |	A123 |

- Client fetches latest version by max `vid`.
- Easy to revert to older version.
- Avoids need for `is_active` boolean column.

## API Design

1. Commit API (PUT /<file_path>)

    Client calls:

    ```bash
    PUT /video.avi
    Payload: blocklist=[h1,h2,h3,h4]
    ```

    Server Flow:
    - Check which chunks exist in S3
    - Return missing chunk list
    - Client uploads those chunks using presigned URLs
    - Client retries the same PUT call
    - If all chunks exist → server commits entry in DB

2. Get Updates API
    ```bash
    GET /sync?account_id=A123&after_vid=42
    ```
    Returns all new versions for that account.

## Multi-versioned File Consistency
- New device joins → gets all metadata
- Syncs chunk-by-chunk only if required
- Chunks are shared across files if identical (de-duplication)

## Related Systems
- Git: Uses content-addressed storage (hash-based chunking)
- Google Drive: Also supports multiversioning (but serves full files for in-browser playback)
- S3: Native versioning support (not used here due to chunking design)

##  Considerations
- CDN Not Required:
    - Since it's personal sync (user uploads & downloads their own data), CDN isn’t needed.

- Hash Collisions:
    - Use SHA-256 for minimal collision risk
    - Avoids uploading duplicate content

- Cache Management:
    - Temporarily store and reconstruct full files on client during lazy access
    - Delete after use