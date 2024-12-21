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

    In a direct message, the `receiver_id` is used, while for a group or channel message, the `channel_id` is utilized. For each message, only one of these fields is populated and the other field will `remain null`, meaning a message can either be a direct message or a group message, but not both.

4. **Users_Channels Table:**
    - `user_id`, `channel_id`: Shows which users are in which channels, using a composite key for efficient querying.

## Improved Schema Design
The initial schema is functional for channels but becomes inefficient for DMs. To get the unique users a person has messaged, querying the **Messages** table becomes expensive. A better design would be to treat both DMs and group messages as "channels."

### Challenges in the current design:
1. The initial schema works well for channels since retrieving a list of channels a user belongs to is straightforward by querying the `Users_Channels` table. 

2. However, the schema becomes inefficient for direct messages (DMs). To find users with whom a person has previously chatted, we must scan the entire `Messages` table to identify unique `receiver_id` values for a given `sender_id` and vice versa. This process is computationally expensive.

3. Retrieving the complete conversation between two users (A and B) requires scanning the table for entries where sender_id and receiver_id are either A or B, which is also resource-intensive.

### Key Changes:

A better design would be to treat both DMs and group messages as "channels." Now instead of scanning for `sender_id` and `receiver_id` conditions, we can simply query for the `channel_id` that represents the conversation between both the users.


- **Messages Table:** Now contains `channel_id` only (no more `receiver_id`), treating DMs as channels between two users.
    - **Messages Table:** `id`, `sender_id`, `receiver_id`,`channel_id`, `message`, `created_at`


- **Channels Table:** Add a `type` column to distinguish between DM and Group channels.
    - **Channels Table:** `id`, `name`, `type`

This design simplifies querying:
```sql
SELECT * FROM messages WHERE channel_id = ? ORDER BY created_at DESC LIMIT 10;
```
The system can now query the same for both group and direct messages by using just `channel_id`.

- **Membership Table** (formerly `Users_Channels`): This table now handles both DM and group memberships. Add additional fields for `is_star`, `is_favorite`, `mute`, `preference` for each user’s relationship to the channel.

## Message Processing

Now, let's discuss the messaging part:

1. We have users exchanging messages.
2. Our API servers handle the requests.
3. The message database is partitioned, with the partition key based on `channel_id` or `workspace_id`.

### Read Path
#### Historical Messages
- Historical Messages are fetched from the DB using a **paginated REST API**. As users scroll, new messages are fetched in pages, for example 10 per page.

```ruby
GET /channels/:channel_id/messages?page=2
```
Messages are cached as they are immutable, improving efficiency.

#### Real-Time Messaging
- Real-time messaging is handled via websockets, not REST. The websocket connection is used for delivering live messages to the client.


When scrolling through historical messages, there is no need for data to be transmitted via websockets. Websockets are specifically designed for real-time message delivery to the frontend, ensuring immediate updates for active conversations. Historical messages can be fetched through other methods like REST APIs or pagination, reducing resource consumption from websockets.

### Write Path (Message Persistence)
All users connect via web sockets to an edge server, which handles sending and receiving messages in real-time. When a user sends a message, it is first routed to the edge server, and the server is responsible for storing this data in the database. There are two main methods to persist this data:

- **REST API:** This is more reliable, as retries are easier to implement. If a message fails to be sent (e.g., due to a network issue), the user can refresh and resend the message.

- **Websockets:** This method is preferred for real-time message delivery but offers less persistence guarantee. It's optimized for real-time communication.

## Message Delivery Types
1. Persistence > Delivery (Slack-like):

    - Messages are persisted first, then delivered to users.
    - Flow: User sends a message -> REST API persists the message -> Websocket delivers the message in realtime
    - Websocket allows the server to push data to the user without the need for the user to request it.

2. Real-time Communication (WhatsApp-like):

    - Delivery is prioritized, and persistence happens asynchronously.
    - Flow: Message is sent and delivered immediately via websocket, and then the message is stored asynchronously by the app and backend.
    - The app maintains a local state, allowing access to historical messages even when offline. When User A sends a message, it is stored in the app's local SQLite database, enabling asynchronous persistence. User A sends the message to User B, who receives it, while the message is also forwarded to a message broker, which handles its persistence asynchronously.

3. Real-time, No Persistence (Zoom-like):

    - Messages are delivered in real-time without any persistence (e.g., earlier version of Zoom chat).

In our scenario, when User A sends a message, it’s persisted via a POST request to a REST API. To deliver the message to User B in real-time, both users are connected to an edge server via WebSocket. After persistence, the API server notifies the edge server, which then pushes the message to User B.

## Scaling WebSocket Servers

The DB is horizontally partitioned, and the API servers are stateless, allowing them to scale horizontally by placing them behind a load balancer. However, scaling the edge (socket) server is more challenging due to their stateful nature:

- Each user maintains an open connection.

Each user establishes a single connection with the socket server, and the maximum number of connections per server is limited to 65,535 (2^16) at any given time. Connections are uniquely identified by a 5-tuple, consisting of (`src_ip, src_port, dst_ip, dst_port, protocol`). For the same protocol, this 5-tuple must remain unique. The `dst_ip`, `dst_port`, and protocol are fixed, so the uniqueness is maintained by varying the `src_ip` and `src_port`.

If the `src_ip` is fixed (i.e., for a single source system), the server can use different source ports (up to 2^16, due to the 16-bit port number) to create unique connections. Typically, socket servers are behind a load balancer, meaning connections come from a limited set of machines. However, if the socket servers are directly accessible over the internet, the system can handle a total of 2^32 * 2^16 unique connections for a destination.

As mentioned in a blog post, WhatsApp's edge servers, which are exposed to the internet (Naked Edge Servers), are designed to support up to 5 million TCP connections on a single server. However, this raises concerns about security, as exposing servers directly to the public internet can compromise sensitive information. To mitigate this, WhatsApp uses a **Dual NIC** setup, where one network card is public-facing and the other is private-facing. This configuration ensures that while the server is accessible on the internet, no sensitive information resides on it, as the communication between the public and private networks is securely relayed through the two network cards.

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
- all other edge server. ED2, ED3, and ED4, which then check which users on their servers belong to CH1 and deliver the message to those users.

**Membership DB:** Each edge server maintains a connection to the **Membership DB** to determine the users in a channel. This allows the edge server to deliver messages to the correct users. Additionally, each edge server maintains a list of users connected to it and the channels they are part of.

**Optimization:** In the case where a message is sent in a channel with fewer users, such as CH2 (with only A and E), the message is still sent to all edge servers, creating unnecessary network traffic. To optimize, we can adopt an alternative approach.

### Optimized Approach

#### Approach1: Managing Edge Server Subscriptions
To efficiently manage which edge servers are subscribed to which channels, we can maintain a dedicated table that maps each edge server to its subscribed channels. The table can be structured as follows:

- **ED1**: CH1, CH2
- **ED2**: CH1
- **ED3**: CH1, CH2
- **ED4**: CH1

When an edge server, such as **ED1**, receives a message, it will refer to this table before forwarding the message to other edge servers. For example, if ED1 receives a message for **CH2**, it will check the table and find that **ED3** is the only other server subscribed to **CH2**. Therefore, **ED1** will forward the message only to **ED3**, avoiding unnecessary forwarding to ED2 and ED4, which are not subscribed to CH2.

#### Approach2: STAR Topology with Redis Pub-Sub

Instead of maintaining seprate table and every edge server communicating with every other edge server, a **STAR topology** can be implemented. In this approach, all edge servers are connected to a central entity. This approach simplifies the management of subscriptions and makes it easier to add or remove edge servers. The central entity can store the edge server-to-channel mapping and handle the forwarding of messages.

Process Flow:

1. **Message Flow:**
    - When User A sends a message to **CH2**, edge server **ED1** receives the message and forwards it to the central entity. The central entity then forwards the message to **ED3**, which is the only edge server subscribed to **CH2** (based on the mapping). The message is not forwarded to **ED2** or **ED4**.

2. **Central Entity as a Pub-Sub Model:**

    - The central entity is a publish-subscribe (pub-sub) system based on a push-based approach, ensuring high throughput and real-time message delivery.
    
    - Unlike Kafka, which is pull-based (requiring consumers to request messages), the pub-sub system pushes messages to subscribers when they arrive, without needing the subscriber to poll for new messages.

3. **Using Redis Pub-Sub:**

    - Redis, with its pub-sub model, is ideal for this setup because it provides fast message delivery and doesn't require pre-configuration of channels.

    - Redis push messages to the edge servers without the edge server asking for it. This offers high throughput and real-time message delivery.

    - When a user joins an edge server, the edge server subscribes to the appropriate channels via Redis pub-sub. For example, when User B joins **ED1**, **ED1** subscribes to CH1, and when User A joins, **ED1** subscribes to **CH2**.

4. **Scaling with Redis Cluster Mode:**

    - To avoid a single point of failure, Redis should be run in cluster mode, distributing the load across multiple Redis instances and ensuring high availability.

**Disclamer:** Arpit shows the code for it, do it at your own.

## Connection Handling:
When users connect to the system, they need to connect to an edge server. How the user came to know the ip of the edge server? This can be handled using one of two approaches:

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