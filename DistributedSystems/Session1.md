# Distributed System Approach: Load Balancer Design and High Availability

## Introduction

In designing distributed systems, ensuring high availability is crucial. One commonly used component is the **Load Balancer (LB)**, which distributes client requests across backend servers. The LB abstracts the backend server topology from the client, allowing elasticity (scaling up/down) of servers without client awareness.

##  Basic Load Balancer Architecture

A load balancer forwards requests from users to backend servers and can be configured with **load balancer algorithms**, **health protocols**, and **IP addresses of backend servers**. Typically, the configuration of multiple LB servers is stored centrally in a **Configuration Database (Config DB)**.

### The Importance of Centralized Config DB in LB

We could consider storing the configuration directly within the Load Balancer (LB) itself, eliminating the need for a separate database. However, it’s important to think about **who** interacts with the LB. The primary users may seem to be the clients accessing backend servers, but they are not the only users. For instance, consider how AWS manages its load balancers—AWS developers configure the LB through internal tools like AWS Console, which communicates with the LB through APIs. This shows that the LB must serve both end users and the developers who manage it.

Therefore, we need a **central source of truth** for configuration data that can easily accommodate adding or removing LB servers and managing their settings. Storing configurations in the LB itself reduces flexibility and makes it difficult to handle dynamic scaling or changes.

**System design is not just about coding or engineering solutions**; it’s about understanding the needs of the users and designing a system that caters to those needs. Empathy toward how your users—whether developers or end users—interact with the system ensures that the design is practical and production-ready. This holistic approach leads to more reliable, maintainable systems.

### Basic Flow for LB Operation
Till now we can think of the request flow as -- 

1. **Client Request:** Users send their requests to the Load Balancer (LB) server.

2. **Configuration Fetch:** The LB server retrieves necessary information, such as backend server details, from the configuration database.

3. **Backend Selection:** Based on the configuration, the LB server selects a suitable backend server to handle the request.

4. **Response Handling:** The LB server forwards the client's request to the chosen backend server, retrieves the response, and then sends it back to the user.

### Issues with the Current Request Flow Architecture with LB

- **DB Overhead on Every Request:** Making a database call for each request increases the time it takes for the user to get a response. This extra call to the config DB adds unnecessary latency that doesn't directly benefit the user.

- **Response Time Delays:** The time spent calling the config DB for information is overhead, especially when the backend server (B server) is fast at processing requests. For example, if the B server responds in 5ms but the LB server takes 100ms to handle the communication, most of the delay is caused by the load balancer rather than actual request processing.

- **Non-Optimized LB Communication:** The developer's primary focus should be on optimizing the backend server's response time. However, if the LB server introduces additional delays due to constant DB lookups, it defeats the purpose of optimizing the B servers for faster responses.

- **Excessive Relay Time:** The overall time the LB server spends between receiving the request and forwarding it to the B server should not exceed much from the time taken by the B servers to compute and return a response.

##  Optimized Design
To optimize, we **cache the configuration in the LB server**, avoiding repeated calls to the config DB. However, this introduces **staleness**—when config DB updates, LB may not immediately reflect the changes. To resolve this, synchronization mechanisms are required.

## Synchronization Approaches

1. **LB API Notifies LB Servers:**

    - LB API informs/notify LB servers of changes by calling their endpoints.

    - **Challenge:** This requires knowing the IP addresses of all LB servers. Since the number of LB servers can scale up or down, and some may stop functioning, keeping track of active IP addresses becomes complex. The LB API must handle these dynamic changes and manage retries in case some notifications succeed while others fail. Additionally, ensuring all LB servers are up-to-date requires tracking their status continuously, which adds further complexity to the notification process.


2. **Polling from LB Servers:**

    - LB servers periodically poll the config DB for updates.

    - **Challenge:** Delays due to polling intervals; real-time updates may take longer to propagate.

3. **Gossip Protocol:**

    - LB servers communicate updates amongst themselves.

    - **Challenge:** Every LB server must know the IPs of other servers, leading to complexity.

---------------------------

> Disclaimer:

> The solution to the issues discussed above was explored in detail during last week's session on **non-relational databases**. Refer to the previous notes for a more comprehensive understanding. Try to find out the solution independently using those concepts before proceeding.

-------------------

4. **Redis Pub-Sub Model:**

    - LB servers subscribe to a central Pub-Sub system (e.g., Redis). When the config DB changes, LB servers are notified in real-time, without the need of polling the config DB.

    - Whenever a new LB server is added, it automatically subscribes to the Pub-Sub model. This ensures that when the configuration in the config DB is updated, the LB API pushes the changes to the Pub-Sub system. The Pub-Sub model then broadcasts those updates to all subscribed LB servers, eliminating the need to manually track or manage the IP addresses of individual LB servers.

    - **Limitations:** Redis cannot ensure guaranteed delivery, so missed updates are possible.

5. **Hybrid Approach: Redis Pub-Sub + Polling**

    To ensure reliability, we can use a hybrid approach:

    - **Primary Mechanism:** Use Redis Pub-Sub to notify LB servers of real-time updates.
    - **Backup Mechanism:** LB servers periodically poll the config DB to capture any missed updates.

    This ensures that updates are processed in real-time, but in the rare event of missed messages, polling ensures eventual consistency.

6.  **Using Zookeeper as an Alternative:**
    
    An alternative to Redis Pub-Sub + Config DB is Zookeeper:

    - Zookeeper provides a durable, fault-tolerant, distributed config store with built-in pub-sub capabilities.

    - It tracks servers listening to a particular configuration key, ensuring guaranteed delivery when config changes.
