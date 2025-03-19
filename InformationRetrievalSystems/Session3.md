# Instagram Live Reactions System Design

## Problem Overview:

When users react during a live stream, their reactions should be visible to others watching the same stream. The system should handle multiple types of reactions, scale for 50 million users, and process up to 5 billion reactions. The design should meet the following goals:

1. **Horizontal Scalability:** The system should handle peak loads during live streams, but also scale down to reduce infrastructure costs when the stream ends.

2. **Good User Experience (UX):** Users should experience minimal delays in seeing reactions from others.

## Reaction Handling Flow:
When a user taps the reaction button, there are two ways this interaction reaches the backend:

1. **HTTP Post API:** Used for regular server-client interactions.
2. **WebSockets:** Used for real-time communication with lower latency.

To reduce the number of server calls, **client-side batching** is employed. If a user continuously taps the reaction button, the client batches multiple reactions and sends them together, avoiding individual API calls for every tap. This keeps the API server from being overwhelmed with unnecessary requests.

## Persisting Reactions:
Once the batched reactions reach the API server, they are persisted in the database and then forwarded to other users watching the stream. This process can be handled asynchronously to reduce latency.

Using **Kafka** ensures high availability and durability for asynchronous tasks, with Kafka consumer groups handling real-time delivery, analytics, and other services. The flow follows these steps:

1. **Client-side Batching:** Client batches reactions.
2. **Push to Message Queue:** The batched data is sent to Kafka, either synchronously or asynchronously, from the API server.

## Batching and Kafka Integration:
At the backend, batching is key to handling large volumes of data efficiently:

- **Batching Logic:** Either batch reactions based on a time window (e.g., 5 seconds) or a message limit (e.g., 20,000 messages per batch), whichever comes first.
- **Batch Size Optimization:** The batch size is determined based on Kafka’s operational payload size, which ensures efficient performance.

To minimize data size, reactions are encoded using protobuf or run-length encoding, reducing bandwidth and storage needs.

## Data Aggregation:
The backend processes reactions by aggregating data for each live stream:

- **Batch Processing:** Kafka consumers process batches and aggregate reaction data for each stream.
- **Micro-batching:** Technologies like Spark, Flink, or Kafka Streams can be used for micro-batching and real-time data processing.
Once the batch is processed, the results are written to another Kafka topic. This enables other services—like notifications, analytics, and trending data—to consume the aggregated data without re-computing it.

Once the batch is processed, the results are written to another Kafka topic. This enables other services—like notifications, analytics, and trending data—to consume the aggregated data without re-computing it.

## Fan-Out and Reaction Distribution:
Sending every reaction to all users would lead to unnecessary bandwidth consumption (O(n^2) complexity). Instead:

- **Sampling:** Only a sample of popular reactions (e.g., frequently used emojis) is sent to users.
- **Fan-out:** This sampling is performed by a service that consumes the final Kafka topic. The service decides which reactions are important to broadcast and forwards them to edge servers.
- **Edge Servers:** These edge servers then fan out the selected reactions to users.


## Final Architecture Flow:
1. **API Server:** Handles HTTP requests and batches reactions before publishing them to Kafka.

2. **Kafka:** Serves as the message queue for reactions. Kafka consumers aggregate data and publish it to another Kafka topic.

3. **Real-Time Processing:** The final data is consumed by services that distribute reactions to users. Edge servers sample and emit reactions in real-time using Redis pub/sub.

4. **Redis:** Stores and publishes reaction data to subscribed users in real-time, ensuring efficient fan-out of reactions.

## Key Takeaways:
1. **Reuse Real-Time Messaging Infrastructure:** From Redis pub/sub to user delivery.
2. **Efficient Reaction Distribution:** Not every event is sent to every user; reactions are sampled to minimize bandwidth.
3. **Leverage Expensive Computations:** Once reactions are processed, they are reused across multiple services (e.g., analytics, notifications) without needing to re-compute.
