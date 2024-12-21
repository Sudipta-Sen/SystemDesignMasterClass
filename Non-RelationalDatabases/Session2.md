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