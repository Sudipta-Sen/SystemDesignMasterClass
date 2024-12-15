# Non-Relational Databases

## Document Databases
- **Definition:** 
    
    Document-oriented databases primarily store, retrieve, and manage data in the form of documents, often using formats like JSON or BSON.
- **Complex Queries:**

    Document DBs support complex queries, such as aggregations (grouping, summing, averaging), allowing for more advanced data analysis directly within the database.
- **Partial Document Updates:**

    These databases allow for updating a portion of a document without needing to read and rewrite the entire document, which can greatly save network bandwidth. For example, updating only the number of times a document was accessed, instead of sending and updating the whole document every time.

    - **Benefits:**
    
        This approach reduces network traffic, saves bandwidth, and increases efficiency, particularly useful when dealing with large documents where only specific fields need to be updated frequently.

- **Examples:**
    - MongoDB
    - Elasticsearch

## Key-Value Stores
- **Definition:** 

    In these databases, data is stored as key-value pairs. All interactions (read, write, update, delete) revolve around keys.

- **Access Pattern:**

    Every operation is performed by key—either retrieving, updating, or deleting the value associated with a particular key.

- **Horizontal Scaling:** 

    These databases are naturally suited to horizontal scaling (adding more servers), as keys can be easily partitioned across different servers, distributing data evenly.

- **Limitations:**

    Since the operations are key-based, more complex queries, such as joins or aggregations, are generally not supported.

- **Examples:**
    - Redis
    - DynamoDB
    - Aerospike

## Column-Oriented Databases
- **Definition:** 

    These databases store data in columns rather than rows, making them well-suited for analytical queries that only need to access specific columns of a dataset.

- **Row-Oriented Databases (OLTP):**

    Traditional databases store data by row, meaning each row contains all data fields for a transaction.

    All columns of a row are stored together in a single disk block. This design minimizes the number of I/O operations needed to access or modify the entire row, making it ideal for transaction-based queries(OLTP) where you typically need to retrieve or update most columns in a row. (e.g., `SELECT * FROM table`)

- **Column-Oriented for OLAP:**

    In contrast, column-oriented databases are designed for scenarios where you only need to read a subset of columns (e.g., CPU usage, timestamps).

    Column-oriented DBs store the values of each column in separate disk blocks. This approach optimizes performance when querying specific columns across multiple rows, as it minimizes the number of I/O operations required to retrieve all values for a specific column (e.g., `SELECT cpu_usage, timestamp FROM table`). This makes column-oriented DBs more efficient for analytical queries (OLAP) such as data mining and reporting that focus on few columns rather than entire rows.

- **Example:** 
    - Apache HBase
    - Google Bigtable.

## Wide-Column Databases (Hybrid Approach)
- **Definition:** 

    Wide-column stores blend the advantages of row and column-oriented databases. They group related columns into "column families" that can be stored and accessed together.

- **Use Case:**

    Imagine a people table where each row represents a person. Some columns (e.g., `first_name`, `last_name`, `bio`) might be grouped into one column family (`about`), while others (e.g., `followers_count`, `following_count`, `posts_count`) might be grouped into another family (`stats`). If a query frequently accesses the `stats` column family, wide-column databases optimize disk I/O by storing these related fields together.

- **Advantages:** 

    By packing the same column families together and store it in the same disk block, wide-column databases reduce the number of disk reads and optimize data retrieval. This structure is particularly useful when different queries target different sets of columns within the same dataset. So while reading the `stats` column family, this arrangement minimizes disk reads by storing all related data in the same disk block until the disk block is filled.

- **Example:**
    - Apache Cassandra

## Graph Databases
- **Structure:**

    Stores data in nodes (entities) and edges (relationships between entities), making them highly effective for relationship-based queries.

- **Best Use Cases:**

    Ideal for scenarios requiring the modeling of complex relationships, such as social behaviors, recommendations, and collaborative filtering.

- **Applications:**

    Commonly used in fraud detection to identify anomalous patterns in transactions. By analyzing connections and relationships in real time, it helps flag suspicious activities which could indicate fraud.

- **Examples:**

    - Neo4j
    - Amazon Neptune: A fully-managed graph database by AWS.
    - DGraph: Written in Go, suitable for small to mid-scale graph use cases.
    - TigerGraph: A proprietary graph database known for enterprise-level 
    deployments.

- **Notable Research:** The paper "WTF" (Who to Follow) by Twitter showcases the use of graph algorithms in making social network recommendations.

## Why Non-Relational Databases Scale:
- **No Relations:**

    While non-relational databases (NoSQL) like graph DBs or key-value stores may have relationships between data (e.g., nodes connected by edges in graph DBs or keys pointing to values in key-value stores), the term "**no relation**" refers to the **lack of enforced constraints**. Unlike SQL databases, which use strict **foreign key constraints** (e.g., ensuring an "author" exists in an "authors" table before adding a "book" entry in a "books" table), NoSQL databases don’t require these relationships to be enforced. This lack of **orphan row prevention** in NoSQL databases enables greater flexibility and scalability, as there’s no need to maintain complex relationships or constraints between table

- **Denormalization:**

    Denormalization allows storing related data in a single document or entry, avoiding complex joins. This is useful for speeding up queries at the cost of data redundancy.

- **Built for Sharding:**

    NoSQL databases are designed for horizontal scalability through sharding. For example, when we create a DynamoDB table, it requires us to define a partition key (also called a hash key) and an optional range key to automatically shard data across multiple nodes, enabling better distribution and management of large datasets.

## When to Use SQL:
- **ACID Properties:** It helps to maintain strict transactional consistency.

- **Relationships and Constraints:** Ideal where enforcing relationships between data is crucial.

## Post-Class Reading:
- **C-Store:** Learn more about column-oriented databases like C-Store, a pioneer in this space.

- **Graph Databases Pagination:** Explore how to paginate works in a graph database after inserting large amounts of data (e.g., 10,000 entries).

- **Amazon Redshift:** Study how Redshift, a fully managed data warehouse service, organizes and stores data for analytics.

- **MongoDB and Elasticsearch:** Practice creating aggregations and facets in both MongoDB and Elasticsearch to perform advanced data retrieval.