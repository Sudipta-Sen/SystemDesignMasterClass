# Designing Slack’s Realtime Communication System

## Requirements
1. **Multiple Users:** Support for users across different channels and direct messages (DMs).

2. **Multiple Channels:** Users can create/join multiple communication channels.

3. **Real-Time Messaging:** Messages sent and received instantly.

4. **Historical Messages:** Ability to scroll back and view previous conversations.

## Similar Systems
Realtime polling, creator tools, interactive chat, and any other message exchange system with real-time behavior.

## Initial Schema Design
1. **Users Table:** `id`, `name`, `email`
2. **Channels Table:** `id`, `name`
3. **Messages Table:**
    - `id`: Unique message identifier
    - `sender_id`: User who sent the message
    - `receiver_id`: For DMs
    - `channel_id`: For group messages
    - `message`: The message content
    - `created_at`: Timestamp of message creation
4. **Users_Channels Table:**
    - `user_id`, `channel_id`: Shows which users are in which channels, using a composite key for efficient querying.

## Improved Schema Design
The initial schema is functional for channels but becomes inefficient for DMs. To get the unique users a person has messaged, querying the **Messages** table becomes expensive. A better design would be to treat both DMs and group messages as "channels."

Key Changes:
- **Messages Table:** Now contains `channel_id` only (no more `receiver_id`), treating DMs as channels between two users.

- **Channels Table:** Add a `type` column to distinguish between DM and Group channels.

This design simplifies querying:
```sql
SELECT * FROM messages WHERE channel_id = ? ORDER BY created_at DESC LIMIT 10;
```
The system can now query the same for both group and direct messages by using just channel_id.

- **Membership Table** (formerly `Users_Channels`): This table now handles both DM and group memberships. Add additional fields for `is_star`, `is_favorite`, `mute`, `preference` for each user’s relationship to the channel.

## Message Processing
### Read Path
#### Historical Messages
- Historical Messages are fetched from the DB using a paginated API. As users scroll, new messages are fetched in pages, typically 10 per page.

```ruby
GET /channels/:channel_id/messages?page=2
```
Messages are cached as they are immutable, improving efficiency.

#### Real-Time Messaging
- Real-time messaging is handled via websockets, not REST. The websocket connection is used for delivering live messages to the client.

### Write Path (Message Persistence)
When a user sends a message, there are two options:

- **REST API:** This is more reliable, as retries are easier to implement. If a message fails to be sent (e.g., due to a network issue), the user can refresh and resend the message.
- **Websockets:** These provide faster communication but with less guaranteed persistence.

## Message Delivery Types
1. Persistence > Delivery (Slack-like):

    - Messages are persisted first, then delivered to users.
    - Flow: User sends a message -> REST API persists the message -> Websocket delivers the message.

2. Real-time Communication (WhatsApp-like):

    - Delivery is prioritized, and persistence happens asynchronously.
    - Flow: Message is sent and delivered immediately via websocket, and then the message is stored asynchronously by the app and backend.

3. Real-time, No Persistence (Zoom-like):

    - Messages are delivered in real-time without any persistence (e.g., Zoom chat).

## Scaling WebSocket Servers
Scaling WebSocket servers is the main challenge due to their stateful nature:

- Each user maintains an open connection.
- TCP connection limit is approximately 65535 connections per server.
- **Solution:** Socket servers need to be load balanced behind multiple servers. When scaling to millions of users, as in WhatsApp, they use Dual NIC (Network Interface Cards). One NIC connects to the public network while another to the private network, ensuring secure communication.

## Horizontal Scalability of Edge Servers

In this design, we focus on scaling the edge servers horizontally to handle multiple users while optimizing real-time communication across channels.

**Scenario:**

You have multiple users connected to different edge servers (ED) but interacting in the same Slack-like channel. For example:

- ED1: Users A, B
- ED2: Users C, D
- ED3: Users E, F
- ED4: Users G, H

All users are part of Channel 1 (CH1). If user A sends a message, it must reach all users connected to the same channel, regardless of which edge server they are connected to.

### Solution 1: MESH Topology

Each edge server is connected to every other edge server. When A sends a message on CH1, ED1 delivers the message to:

- B (since B is on the same edge server as A),
- ED2, ED3, and ED4, which then check which users on their servers belong to CH1 and deliver the message to those users.

**Membership DB:** Each edge server maintains a connection to the **Membership DB** to determine the users in a channel. This allows the edge server to deliver messages to the correct users. Additionally, each edge server maintains a list of users connected to it and the channels they are part of.

**Optimization:** In the case where a message is sent in a channel with fewer users, such as CH2 (with only A and E), the message is still sent to all edge servers, creating unnecessary network traffic. To optimize, we can adopt an alternative approach.

### Optimized Approach: STAR Topology with Redis Pub-Sub

Instead of every edge server communicating with every other edge server, a **STAR topology** can be implemented. In this approach, all edge servers are connected to a central entity (a Redis Pub-Sub server).

- Redis Pub-Sub: The central entity holds a mapping of which edge servers are subscribed to which channels.
- When A sends a message to **CH2**, **ED1** sends the message to the central entity, which then forwards it only to **ED3** (since **E** is connected to **ED3**). This minimizes unnecessary communication with **ED2** and **ED4**.

**Pub-Sub Model:** Redis is ideal for this approach as it uses a push-based system, where the central entity pushes messages to the edge servers without needing polling. This offers high throughput and real-time message delivery.

- **Cluster Mode:** Redis Pub-Sub can be run in cluster mode to avoid a single point of failure, ensuring redundancy and high availability.

## Connection Handling:
When users connect to the system, they need to be assigned an appropriate edge server. This can be handled using one of two approaches:

1. **DNS Resolver:** Assigns users to random edge servers.
    - **Problem:** This approach can result in users from the same organization being connected to different edge servers, leading to excessive cross-server communication.

2. **Connection Balancer:** Instead of random assignment, a **connection balancer** determines the optimal edge server for a user. This component ensures that users from the same organization are grouped on the same edge server (or a subset of edge servers), reducing cross-server communication.

**Resilience:** If an edge server goes down, the connection balancer assigns the user a new edge server to connect to. The balancer aims to pack users from the same organization as closely as possible to minimize latency and cross-server overhead.

## Key Takeaways:
1. **Star Topology:** Minimizes connections between edge servers and improves scalability by using a central Redis Pub-Sub server.

2. **REST API for History:** Historical messages are fetched using REST API, not websockets. This allows for paginated fetching and caching of immutable messages.

3. **Websockets for Real-Time Delivery:** Websockets are used solely for real-time message delivery, ensuring efficient communication between users without waiting for REST API responses.

4. **Connection Balancer:** Ensures users from the same organization are connected to the same edge server to reduce cross-server communication.

5. **Handling Edge Server Failures:** If an edge server fails, users reconnect via the connection balancer, ensuring continuous communication.

By using Redis Pub-Sub, a connection balancer, and efficient topologies, we can scale edge servers horizontally while maintaining efficient and real-time communication across channels.