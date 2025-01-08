# Distributed System Synchronization with Remote Locks
In distributed systems, synchronization is a common challenge, requiring different approaches depending on the entities involved:

1. **Threads within the same process** use mutexes or semaphores.
2. Processes across the same machine often synchronize using disk-based locks (e.g., preventing concurrent execution of certain system commands like `apt-get upgrade`).
3. **Distributed systems across multiple machines** employ **remote locks** for synchronization.

## Example: A Hypothetical Queue for Synchronization

Consider a hypothetical queue system that doesn't inherently provide isolation, data protection or locking mechanisms. The queue simply processes messages by firing a request, fetching data, executing the task, and then deleting the message without ensuring any concurrency control. This means that consumers need to manage synchronization themselves.

In this scenario, each consumer must implement logic to determine who will access the queue, retrieve a message, process it, and delete it, ensuring that only one consumer processes a message at a time.

![](Pictures/16.png)

**Solution: Centralized Lock Management** To ensure proper synchronization, a **lock manager** can be introduced. This lock manager acts as a central entity where only one consumer can acquire a lock, enter the critical section (process a message), and then release the lock. Only the consumer holding the lock is authorized to process the queue or perform any operations on it. The requirements for this lock manager are:

1. **Atomicity:** Only one consumer should be able to hold the lock at any given time.

2. **TTL (Time-to-Live):** If a consumer crashes or fails to release the lock, the lock must expire, preventing perpetual starvation.

A popular solution for implementing these distributed locks is **Redis**, which provides atomic operations and TTL. The widely-used **RedLock algorithm** ensures high availability and distributed lock functionality.

DynamoDB, a fully managed service by AWS, is another option for distributed locks, though Redis remains the more popular choice for such implementations.

## Flow of Distributed Locks in Redis
In a distributed system, multiple consumers compete to acquire a lock for critical sections. The general flow is as follows:

1. **Lock Acquisition:** n consumers try to acquire the lock. One of them successfully obtains the lock, allowing it to enter the critical section and perform its operation.

2. **Critical Section:** The consumer that holds the lock proceeds to perform its operation, ensuring that no other consumer can interfere with this process.

3. **Lock Release:** Once the operation is complete, the consumer releases the lock, enabling other consumers to attempt acquiring the lock and enter the critical section.

This process is widely used in distributed transactions, where remote locks are essential for data integrity. However, it’s important to note that while distributed locks are heavily used, they reduce throughput, so their usage is often minimized for performance reasons.

![](Pictures/7.png)

Example: Consider a queue named `Q7`. Suppose `Consumer2 (C2)` wants to acquire a lock on `Q7`. In Redis, `C2` will set a value with the key `Q7` and value `C2`, along with a TTL (time to live). This indicates that `C2` holds the lock on `Q7`. If the key `Q7` already exists, `C2` will not set anything and will wait for the lock to be available again.

## Basic Redis Locking
To acquire a lock in Redis:

- A consumer sets a key using the `SETNX` command, which ensures the key is only set if it doesn't exist.
- If the lock is acquired, the consumer enters the critical section, processes the task.

    ![](Pictures/8.png)
- After the task got finished the consumer releases the lock.
    ![](Pictures/9.png)

## Issue with Release Lock implementation

In distributed systems, while only one transaction can acquire the lock and enter the critical section, there are concerns with the current lock release implementation:

1. **No Ownership Check for Lock Release:** If there's no check for ownership, any consumer could release the lock, even if they didn't hold it. Although the system ensures only one consumer can enter the critical section, it's still possible for a developer to forcefully delete the lock without proper validation. However, as developers, we must avoid such practices to ensure system integrity.

2. **Expired Lock Scenario:** Suppose transaction T1 acquires the lock and is performing an operation that takes longer than expected, surpassing the TTL of the lock. Once the TTL expires, the lock is automatically deleted, and transaction T2 may acquire it. Now, when T1 finishes its operation, it attempts to release the lock—potentially releasing the lock held by T2.

**Solution:** To address this, we must ensure that only the consumer holding the lock can release it. The correct approach is to check if the lock belongs to the consumer before releasing it, ensuring that no other process unintentionally interferes.

![](Pictures/10.png)

Can you figure out any issue with the above release lock implementation?

**Problem:** One issue arises between the `redis.get` and `redis.delete` operations. Suppose T1 acquires the lock and performs `redis.get` to confirm it holds the lock. If a context switch occurs before T1 completes its critical section, the lock's TTL (Time-to-Live) may expire, causing the key to be evicted from Redis. As a result, T2 could acquire the lock and enter the critical section. Once T1 resumes execution, it may attempt to release the lock it originally acquired, as it previously received confirmation from `redis.get`, but this would mistakenly release the lock held by T2.

**Solution:** To prevent this race condition, atomicity between `redis.get` and `redis.delete` is required. Redis provides a solution using LUA scripts to ensure that both operations occur atomically, meaning they execute as a single unit. The command `eval` can be used to pass the `LUA script`, ensuring the lock is only deleted by the transaction that holds it. This guarantees that the lock management remains consistent even during context switches.

![](Pictures/17.png)

![](Pictures/11.png)

---------------------------

> Disclaimer:
> Arpit shows the code for this acquire and relase lock, do it by yourself

---------------------------


## Distributed Lock for High Availability and Fault Tolerance

When using a single Redis server for remote locks, it introduces a single point of failure. To improve the system's fault tolerance and availability, a distributed lock approach is often used. Here’s how it works:

1. **Multiple Redis Servers:**

    - Instead of a single lock manager, we use `n independent` Redis servers.
    - These Redis servers are not replicated, meaning each server operates independently.

2. **Locking Mechanism:**

    - A transaction acquires a lock if it can secure the lock on **more than 50% of the servers.**

    - If a transaction fails to secure the majority of the locks, it releases all the acquired locks and reports failure, preventing deadlocks or starvation.

        ![](Pictures/12.png)

3. **Fault Tolerance:**

    When using distributed locks across multiple Redis servers, the system ensures fault tolerance by requiring a transaction to acquire a lock on more than 50% of the total Redis servers. For example:

    - In a system with **5 Redis servers**, a transaction must secure a lock on **at least 3 servers**.

    - If **1 or 2 servers go down**, the transaction can still proceed by acquiring locks on the remaining servers, maintaining the ability to operate.

    - However, even if one server goes down, the calculation of 50% is still based on the total number of Redis servers, not the active number. In the case of **5 servers**, even with **1 or 2 servers down**, the system still calculates the 50% threshold on the original total of 5 servers, meaning **3 locks** are always required.

    - **If 3 servers go down**, the system cannot meet the majority requirement, and lock acquisition fails.

    This fault tolerance provides resilience, with the ability to withstand failures of up to 2 servers while continuing to function normally. If higher fault tolerance is required, the number of Redis servers can be increased. The algorithm for the same -- 

    ![](Pictures/13.png)

4. **No State Maintenance for Redis Servers:**
In this distributed lock system, Redis servers do not maintain state or replication. Even if a Redis server goes down while a transaction holds a lock, no conflict arises:

    - For instance, if T1 acquires the lock on 3 servers and then one of the servers fails, the transaction continues its operation in the critical section with the remaining 2 locks.

    - Other transactions cannot proceed until T1 releases all its locks, ensuring consistency.

    - The calculation of 50% continues to be based on all 5 Redis servers, not the remaining active ones, to prevent potential inconsistencies.

## Comparison of Approaches

1. **Single Redis Node:** Simple but introduces a single point of failure.

2. **Multiple Independent Redis Servers:** Ensures high availability and fault tolerance, but reduces throughput.

3. **Redis Read Replicas:** 

    - **Asynchronous updates:**
        
        In a **read replica setup**, correctness can be compromised due to the delay in replication from the master node to the replicas. This creates a window where multiple transactions might incorrectly perceive the lock as available and acquire it simultaneously, leading to conflicts. Additionally, if the **master node fails** before the changes propagate to the replicas, inconsistencies can arise, as some replicas may not yet have the updated state.

        If asynchronous updates are used between the master and replicas, this issue becomes more pronounced, with potential correctness violations during the replication lag.

    - **Synchronous updates:**

        To mitigate this, synchronous updates can be employed, ensuring that all replicas are updated simultaneously. However, this comes with a tradeoff: if one replica node goes down during the synchronous update, the transaction could be blocked, waiting for the failed node to recover, which in turn **decreases availability**. Thus, while synchronous replication ensures consistency, it can impact the system's availability during node failures.

![](Pictures/14.png)

---------------------------

> Disclaimer:
> Arpit shows the code for Synchronous and Asynchronous, do it by yourself

---------------------------