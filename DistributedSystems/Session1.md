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

## Observability for LB Server
In a distributed system, monitoring backend servers is essential to maintain availability and performance. As we focus on LB servers in this discussion, we’ll explore how observability helps monitor LB performance, track backend health, and manage system scaling to ensure optimal reliability.

### Health Check and LB Monitoring

- LB forwards client requests to backend servers. To ensure efficient routing, it's important for the LB to know the current health of each backend server.

- **Health Check API:** Each backend server exposes a `/health` endpoint, which the LB could theoretically use to monitor server health.

- **Challenges:** Performing health checks directly from the LB is inefficient. Large numbers of backend servers would slow down the LB’s primary task of routing requests.

    - Technical Reasons:
        - Limited TCP connections: Each health check and request require separate connections.
        
        - Redundant health checks from multiple LBs: Multiple LBs querying the same server create unnecessary load.
        
        - **Staleness and delay:** If backend servers fail between health checks, this might not be reflected immediately in the LB’s decision-making process.


### Orchestrator for Health Checks
- Instead of having the LB handle health checks, we introduce an **Orchestrator Server**. This server regularly checks the health of all backend servers and updates the central **Config DB** accordingly.

- Advantages:
    - Reduces the load on LBs.
    - Removes redundant checks by delegating to a single orchestrator that updates all LBs.
    - Allows efficient updating of backend server health in a highly available manner.

### Self-Healing and Scalability
- The orchestrator system is designed to be self-healing. If an orchestrator worker goes down, another worker takes over its responsibilities. A leader is elected to distribute the monitoring tasks among workers.

- Leader Election: The orchestrator system includes a leader who delegates backend server monitoring to workers. If a leader fails, another worker can take over as the leader, ensuring continuous monitoring.

###  Redis Pub-Sub Model for Configuration Updates
- A Redis Pub-Sub model is used to update LBs when there are configuration changes in the Config DB (e.g., adding or removing a backend server).
    - New LB servers subscribe to the Redis Pub-Sub channel, and whenever there’s a change in the Config DB, the update is pushed to all subscribed LBs.
    - This eliminates the need to manually track IP addresses of all LB servers.

### Ensuring Availability with Prometheus and DNS
- **Monitoring with Prometheus:** The orchestrator uses Prometheus to monitor average CPU and memory utilization and to determine when to scale the number of LB servers up or down.

- **DNS for Load Balancing:** Instead of introducing another LB for the LBs, a DNS server is used to distribute traffic among multiple LBs.

    - When a user requests a domain (e.g., `google.com`), the DNS server returns the IP address of an available LB.
    
    - The orchestrator ensures that new LBs are added to the DNS, and old LBs are removed automatically as part of scaling.

### Core DNS with Virtual IP for High Availability
- To ensure high availability of the DNS server itself, multiple Core DNS servers are set up with a Virtual IP (VIP). This VIP is mapped to the actual DNS server's physical IP address at the router level.

- The virtual IP ensures that even if the physical DNS server IP changes due to failover, the external world continues to interact with the system through a stable IP address.