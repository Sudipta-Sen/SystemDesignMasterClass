# Distributed Task Scheduler (DTS)

A **Distributed Task Scheduler (DTS)** is designed to schedule and trigger tasks that need to be executed at specific times, ensuring tasks are performed within a Service Level Agreement (SLA). In this case, the SLA for executing the task is 30 seconds, meaning if a task is scheduled for execution at `10:00:00 AM`, it must start before `10:00:30 AM`.

This architecture caters to scenarios where high volume tasks (such as sending notifications to millions of users) need to be scheduled and executed on time, ensuring that the task execution is **distributed, reliable, and scalable**.

## Types of Tasks
The system primarily handles two types of tasks:

1. **One-Time Task:** A task that is scheduled to be executed once at a specific time.

2. **Recurring Task:** A task that repeats execution at a given interval.

Initially, the design will focus on **single-time execution** and later address recurring tasks.

## System Design Requirements
1. **Scale:** The system should handle 2 billion tasks per day, and each task is primarily an HTTP API call.

2. **Granularity:** The scheduling operates with minute-level granularity.

3. **Tenant-based Partitioning:** The system is designed to be multi-tenant, where each tenant (team or service) submits tasks through the DTS and a tenant ID is generated.

4. **Task Ownership:** The DTS does not execute the tasks directly; it only triggers them. The execution logic resides in the tenants' infrastructure, allowing them to handle their own business logic.

5. **API Exposure:** DTS exposes two APIs:

    - **POST /tasks:** For submitting a new task.

    - **GET /tasks/{id}:** For retrieving the status of a task.

## Database Design and Partitioning
- **Data Sharding and Partitioning:** Given the scale (2 billion tasks/day), a single database is insufficient. Hence, a **clustered database** setup with partitioning and sharding is used.

    - The partition key is **tenant-based** rather than time-based, to avoid "hot partitions" (e.g., one partition receiving more tasks than others).

    - The system employs **hash-based partitioning** to evenly distribute tasks across multiple partitions.

- **Configuration Management:**

    - **Zookeeper** is used to store partition and database configurations.

    - DTS API can cache the Zookeeper configuration locally for optimization, updating when changes occur.


## Task Execution Workflow

### Worker Responsibilities
1. **Pullers:** Servers responsible for pulling tasks from the database. They fire queries to fetch tasks scheduled for immediate execution.

2. **Executors:** Servers responsible for making HTTP API calls to execute tasks. To ensure separation of concerns, pullers and executors are different servers.

3. **Message Queue:** A message queue sits between pullers and executors. Pullers fetch tasks from the database and push them into the queue, while executors pull tasks from the queue to execute.

### Task Querying and Execution
- **Query:**

    - A puller queries the `Job` table for tasks whose scheduled time is within a 5-second buffer to ensure no task is missed due to slight delays in execution.

    - Query example:
        ```sql
        SELECT * FROM jobs WHERE tenant_id=? AND scheduled_at <= NOW() + INTERVAL 5 SECOND;
        ```

- **Concurrency Handling:**

    - To avoid multiple pullers picking the same task, the system uses a lock mechanism to ensure exclusivity:
        ```sql
        FOR UPDATE SKIP LOCKED
        ```
    
    - This ensures each puller skips rows already locked by another puller and only picks unlocked rows, maintaining concurrency.

- **Handling Task Failures**
    - **Retries and Observability:** If an API call fails, the executor retries the task. Clients also need visibility into task status (successful or failed). Each task execution status is tracked, and the client can retrieve this status via the GET /tasks/{id} API.

## Scaling the System
1. **Horizontal Scaling for Pullers:** The system can add multiple pullers, each responsible for a subset of tenants. **Zookeeper** stores metadata on which tenant each puller handles. An orchestrator manages the addition and removal of pullers based on load.

    - The orchestrator uses **leader election** to ensure high availability, with multiple worker instances and a master instance coordinating puller activity.

    - The orchestrator also monitors future job load and dynamically adjusts the number of pullers.

2. **Scaling Executors:** 
    - Executors can be scaled based on the number of tasks waiting in the queue. If the queue length increases, the orchestrator can add more executors to ensure tasks are executed promptly.

3. **Partition Shuffling:**
    - If a tenant starts generating a large number of tasks, it may affect other tenants on the same partition. The orchestrator can redistribute tenants to new partitions, potentially assigning new tenant IDs.

## Handling Failures and Recovery
- **Ensuring At-Least-Once Execution:** In the event of a failure (e.g., the system crashes after updating the `picked_at` column but before placing the task into the queue), the system will retry the task execution, ensuring at-least-once execution.

- **Transaction Handling:** To simplify recovery, task execution is performed within a single transaction. The task is pushed into the queue first, and then the `picked_at` column is updated. This approach prevents complex DB recovery processes and ensures SLA compliance.

## Advanced Considerations
1. **Priority-Based Execution:** The system can support priority queues. Higher-priority tasks can be assigned more executors, ensuring quicker execution, while low-priority tasks can be handled with fewer executors.

2. **Broker Scaling:** Multiple brokers can be used to handle different tenants or priorities. Each broker has its own set of executors to ensure efficient task handling.

3. **Orchestrator Monitoring:** The orchestrator continuously monitors the puller and executor nodes, redistributes tasks when nodes fail, and adjusts the system dynamically based on workload.

## Summary
The Distributed Task Scheduler (DTS) is a **highly scalable** and **fault-tolerant** system designed to schedule and trigger millions of tasks (HTTP API calls) daily with minute-level granularity. By partitioning tasks based on tenants, leveraging **Zookeeper** for configuration management, and **using multiple pullers and executors**, the system ensures reliable execution of tasks within defined SLAs.



