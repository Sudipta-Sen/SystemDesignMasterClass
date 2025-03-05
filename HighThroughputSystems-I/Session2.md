# Building a Multi-Tiered Storage System for Analytics

In this note, we will explore the process of building a multi-tiered storage system for an e-commerce platform or any product that scales with time, the use cases around it, and how change data capture (CDC) and multi-storage solutions play crucial roles in data management and analytics.

## Initial Setup and Challenges

- **Multi-User Blogging App Example:** Consider a multi-user blogging application where data is initially stored in a NoSQL transactional database like MongoDB. In the early stages of the product, the product team manually queries MongoDB to generate insights and graphs, which guide business decisions.

- **Scaling Issue:** As the product and company grow, a dedicated insights team is formed. The insights team is proficient in SQL but not NoSQL. MongoDB, like other NoSQL databases, is not optimized for analytics and SQL querying, making it impractical for the team to work directly with MongoDB.

- **Proposed Solution:** A common solution is to copy data from MongoDB to an SQL database. The application sends data to the MongoDB transactional DB and also pushes events to Kafka. From Kafka, data can be written to a SQL database for the insights team to generate reports and analytics.

## Problems with Direct Event Propagation

- **Data Consistency Issue:** For every event in the application (e.g., updating a photo, deleting a photo, etc.), there must be a corresponding Kafka event. A potential risk arises when data is successfully written to MongoDB but fails to write to Kafka. This introduces inconsistency between the NoSQL and SQL databases.

## Change Data Capture (CDC) and ETL (Extract, Transform, Load)

- **Introduction to CDC:** To mitigate the issue of missing events in Kafka, an ETL process can be introduced. ETL involves extracting data from MongoDB, transforming it to a SQL-compatible format, and loading it into a SQL database. One of the best approaches to ETL is **Change Data Capture (CDC).**

- **CDC Mechanism:** CDC pulls updates periodically from the source database (MongoDB), applies necessary transformations, and writes them to the destination database (SQL). This periodic pull mechanism ensures no data is lost. Two popular CDC tools are **Airbyte and Debezium.**

## Multi-Tiered Storage Architecture

- **Hot vs. Cold Data:** In large-scale applications like e-commerce, orders are accessed frequently when they are recent but less frequently after several months. It doesn't make sense to store older, less accessed data in expensive transactional databases (e.g., MySQL). Instead, it should be moved to cheaper storage like S3.

- **Multi-Tier Architecture:**

    - **Hot Storage:** A high-cost, fast-access database like MySQL or DynamoDB.

    - **Warm Storage:** A cheaper, horizontally scalable, read-heavy storage like S3 with query capability using tools like PrestoDB or Hive.
    - **Cold Storage:** A highly cost-efficient storage solution (e.g., S3) where very old and infrequently accessed data can be stored as simple files.

## Data Movement Between Tiers

- **ETL for Tiered Storage:** The data is initially written to hot storage (MySQL/DynamoDB). As it becomes stale (e.g., after 6 months), it is moved to warm storage (e.g., S3). When moving data from hot to warm storage, we minimize the number of S3 calls by storing all related information (order, payment, logistics) in a single JSON object. This process involves:

    - **Staging Area:** Data from different services (order, payment, logistics) is first collected in a staging area (e.g., a structured S3 folder).
    - **Merging Process:** Data is merged using a distributed computing framework (e.g., ParquetSQL pipeline with Hive metadata) and written to warm storage.

- **Cold Storage Transition:** When data becomes older and is no longer accessed frequently (e.g., after a few years), it is moved to cold storage (e.g., Apache Hudi on S3). Cold storage supports queries using tools like PrestoDB, which allows querying CSV or Parquet data stored on S3.

## Strategies for Hot vs. Warm Data Access

- **Metadata Strategy:** One approach is to maintain metadata for all orders, which tracks the age of the data. Queries first check the metadata to decide whether to retrieve data from hot or warm storage.

- **Direct Query Strategy:** An alternative approach is to directly query the hot storage. If the data is not found in hot storage, the query then moves to warm storage. This approach eliminates the need for a metadata lookup, reducing system complexity, but may slightly increase query response time for warm data retrieval.

- **Timestamp in Order ID:** Another approach is to embed the timestamp in the order ID. By examining the order ID, the system can decide whether the data should reside in hot or warm storage. However, this requires coupling the ID generation logic with storage tier management.

## Cold Storage Queries and Optimization

- **Cold Storage Queries:** For cold storage, there are two types of queries:
    1. **Rare Queries:** A customer might request an old invoice after several years. Such rare queries can be handled by tools like PrestoDB, which query S3-based CSV or Parquet data. This setup is cost-efficient for occasional queries but increases cost when multiple queries are run.
    
    2. **Frequent or Investigative Queries:** In cases like legal investigations, where multiple queries need to be fired across a specific data range (e.g., data from 1st Nov to 8th Nov), it’s more efficient to download the data from S3 and load it into a temporary queryable database for intensive querying.

##  Building a Cost-Efficient E-Commerce Order System

- **Order Data Management:** In an e-commerce system, each order consists of multiple components (order details, payment, logistics). As data ages, it transitions from hot storage (MySQL/DynamoDB) to warm storage (S3). Warm storage contains pre-merged JSON objects with all relevant details (order, payment, logistics) for each order. Queries for warm data are less frequent but should still return the same consistent results as hot storage.

- **Benefits of Tiered Storage:**

    - Cost Efficiency: Older data is stored in cheaper storage solutions, reducing operational costs.
    
    - Scalability: Horizontal scalability in warm and cold storage ensures the system can handle large amounts of data as the business grows.
    - Consistency: The system provides consistent query results regardless of whether data is retrieved from hot, warm, or cold storage.

## Conclusion

Building a multi-tiered storage system is crucial for applications that generate and store large amounts of data over time. CDC tools like Airbyte and Debezium help ensure consistent data replication between NoSQL and SQL databases. By leveraging distributed computing tools and structured data pipelines, companies can maintain efficient, cost-effective storage solutions while providing consistent data access to users.