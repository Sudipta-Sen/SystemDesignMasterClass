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
        - **Limited TCP connections:** The number of TCP connections a server can handle is limited. For each backend server, two TCP connections are required: 
        
            - one for relaying messages
            - another for health checks. 
        
            This unnecessarily burdens the LB server. It should focus on routing requests efficiently rather than performing additional tasks like health monitoring.

        - **Redundant health checks from multiple LBs:** If multiple LB servers are performing health checks on the same backend server (Bservers), it leads to redundant work and unnecessary traffic. This increases the load on Bservers due to frequent health check requests. 
        
            Additionally, it exposes a leaky abstraction: for example, if a LB server is scheduled to perform one health check every 5 secs but if a Bserver receives 5 health check requests every 5 secs, it can infer that there are 5 LB servers, which should ideally remain abstracted. The backend should not be aware of the number of LB servers in the system.

### Orchestrator for Health Checks
- Instead of having the LB handle health checks, we introduce an **Orchestrator Server**. This server regularly checks the health of all backend servers and updates the central **Config DB** which is also used by LB server.

- Advantages:
    - Reduces the load on LBs.
    - Removes redundant checks by delegating to a single orchestrator that updates all LBs.
    - Allows efficient updating of Bserver health in a highly available manner.

#### Self-Healing and Scalability

- **Potential Issue:** If the orchestrator server goes down, we would need another monitor to check its status, leading to a potential chain of monitoring the monitor itself. 

    While we could consider having Bservers push health updates directly to the LB server, this approach is not feasible as it requires Bservers to be aware of the LB server’s IP addresses, which should ideally remain abstracted.

- **Solution:** Instead of relying on a single orchestrator, the system can have multiple orchestrators. There are two typer of orchestrator nodes - 
    1. Leader
    2. Worker

    Orchestrator workers work under the leader node. In the event of leader failure, these workers do not stop functioning. The system elects a new leader from the pool of workers. This way, we avoid needing continuous monitors for the monitors and ensure that the system is self-healing.

- **Leader Election:** The orchestrator system implements leader election to maintain high availability and self-healing capabilities. 

    If a worker becomes overloaded -- 
    - other workers can step in to assist. 
    - If none are available, the leader can dynamically spawn new workers and delegate tasks to them. 
    
    In case the leader goes down, another server within the worker pool will automatically take over leadership duties. Two popular leader election algorithms are -- 
    - LCR Algorithm with time complexity O(n²) 
    - HS Algorithm with time complexity O(nlogn)

### Handling Bserver Downtime Without Overburdening the LB Server

A common consideration arises: what if the orchestrator takes a few milliseconds (e.g., 5ms) to detect that a Bserver is down, and in that short window, the LB server still forwards a request to the downed Bserver? It seems logical to think that the LB could identify a failed Bserver if it receives consecutive 5XX errors. However, this method is flawed because not all 5XX errors are indicative of a Bserver failure. For example, a user may make a bad request, leading to 5XX responses even when the Bserver is functioning properly.

Relying on the LB to continuously call the Bserver's health API is also not a viable solution, as this brings us back to the initial problem: overloading the LB with responsibilities beyond request routing.

The LB and orchestrator should have distinct responsibilities. The LB should not be burdened with the task of detecting Bserver failures. Instead, health checks and failure detection must remain the orchestrator's duty, while the LB focuses on relaying requests to backend servers. Keeping these responsibilities separate ensures more efficient operations and avoids unnecessary interference between the two components.

###  Redis Pub-Sub Model for Configuration Updates
- A Redis Pub-Sub model is used to update LBs when there are configuration changes in the Config DB (e.g., adding or removing a backend server).
    - New LB servers subscribe to the Redis Pub-Sub channel, and whenever there’s a change in the Config DB, the update is pushed to all subscribed LBs.
    - This eliminates the need to manually track IP addresses of all LB servers.

### Ensuring Availability with Prometheus and DNS
- **Monitoring with Prometheus:** The orchestrator uses Prometheus to monitor average CPU and memory utilization and other necessary details to determine when to scale the number of LB servers up or down. Now, the orchestrator has two responsibilities.
    - First, it monitors the health check APIs of the Bservers to ensure that only healthy servers are handling traffic. 
    
    - Second, it continuously monitors Prometheus metrics to auto-scale the LB servers based on resource utilization.

- **DNS for Load Balancing of LB Servers:** When multiple LB servers are involved, traffic distribution among them becomes essential. It is not making sense of keeping another load balancer for the LB servers themselves. Instead of this task is handled by a DNS server. 

    The DNS server distributes traffic across multiple LB servers by resolving the domain name to different IP addresses based on predefined algorithms, ensuring efficient load distribution

    - When a user requests a domain (e.g., `google.com`), the DNS server returns the IP address of an available LB.
    
    - **Issue:** when a new LB server is added or an existing one is removed, who will update the core DNS? 
    
        **Solution:** The orchestrator ensures that new LBs are added to the DNS, and old LBs are removed automatically as part of scaling using DNS server api's.

### Core DNS with Virtual IP for High Availability

To ensure high availability for Core DNS, we can deploy multiple Core DNS servers. However, the challenge arises in distributing traffic among these servers while maintaining high availability. One approach is to use like an orchestrator with leader election and an active-passive architecture.

- Problem:

    - When the active DNS server goes down and the passive server takes over, the IP address changes. This poses an issue when users are querying the Core DNS for domain name resolution — they need a way to track the changing IP addresses.

- Solution:
    - The solution is to use a **Virtual IP (VIP)**. The VIP can be configured at the router level, where it is mapped to the physical IP address of the Core DNS. Externally, the VIP is broadcasted, and inside the network, it is mapped to a physical Core DNS IP. When the physical Core DNS IP changes (for example, during a failover to the passive server), the mapping changes, but the VIP remains constant.

        This ensures that users continue to query the same VIP, which abstracts the internal changes in the physical IPs of the DNS servers. This is why cloud platforms often provide a domain name for Core DNS that points to a virtual IP rather than a physical one, allowing for seamless failover without requiring users to track IP changes.

---------------------------

> Disclaimer:

> This lecture by Arpit demonstrates coding your own load balancer. Replicate the code on your own.

---------------------------