# Module 14 — Design a Chat System (WhatsApp / Slack / Messenger) ⭐⭐

> Chat **inverts** the model every previous problem relied on. Until now the server **responded** to
> client requests (stateless, m02/m07). Chat needs the **server to push** to clients in real time, and
> it's **write-heavy** — so you need **persistent connections (WebSockets)**, which are **stateful** and
> break clean load balancing. The whole design is about taming that: **connection management at scale**,
> **routing a message to the right server**, **ordering & delivery guarantees**, and **presence**. This
> is the m03/m07 "persistent connections break statelessness" thread as the *main event*.

> **Format (Part 2):** worked m01 walkthrough; **connection management** + **message routing** are the
> deep-dives. `CROSS_QUESTIONS` drills the follow-ups.

---

## Step 1 — Requirements

**Functional (core):**
1. **1:1 messaging** in **real time** (low latency both directions).
2. **Group chat** (deliver to all members).
3. **Online/offline presence** (+ "last seen").
4. **Delivery & read receipts** (sent → delivered → read — the "double blue tick").
5. **Message history / persistence** (and delivery to users who were **offline**).
6. *(Stretch)* media, push notifications, end-to-end encryption, typing indicators.

**Non-functional (these drive it):**
- **Real-time, low latency:** messages must arrive in ~tens-to-hundreds of ms.
- **Bidirectional + server-push:** both sides send *and* receive frequently → **WebSockets** (not
  request/response). *This is the defining constraint.*
- **High availability + durability:** never lose a message; deliver even after disconnects (store-and-
  forward).
- **Ordered:** messages in a conversation must appear **in order** (per-conversation).
- **Massive concurrent connections:** the scaling dimension is **simultaneous open connections**, not
  just QPS.
- **Consistency:** per-conversation ordering matters; cross-conversation can be eventual (AP-leaning).

> **The defining insight:** the hard part isn't storing messages — it's maintaining **millions of
> stateful, long-lived connections** and **routing a message to whichever server holds the recipient's
> connection.**

---

## Step 2 — Estimation (connections are the key number) ⭐

Assume **500M DAU**, **~40 messages/user/day**, **~10% online concurrently at peak**.
- **Messages (writes):** 500M × 40 = 20B/day ÷ 10⁵ = **~230,000 msgs/sec** avg (~1M peak). Chat is
  **write-heavy** (unusual — most systems are read-heavy).
- **⭐ Concurrent connections:** 10% of 500M = **~50 million simultaneous WebSocket connections** — *the*
  number that shapes the design. A connection server holds ~**50k–100k** connections (WhatsApp famously
  did **~2M/server** with Erlang) → **~500–1,000 connection servers** (fewer if highly tuned).
- **Storage:** message ≈ ~200 B (text + metadata) × 20B/day ≈ **~4 TB/day** → ~1.5 PB/year → sharded,
  write-optimized store, with retention/tiering.
- **Fan-out (group):** a message to a group of K members = K deliveries → big groups multiply this (the
  m13 celebrity/group thread returns).

> **The estimate frames it:** ~50M **stateful connections** (→ connection servers + a session registry +
> a pub/sub backplane) and ~230k **writes/sec** (→ a write-optimized, sharded message store, m04/m05).

---

## Step 3 — Protocol / API
- **WebSocket** for the real-time channel: after auth, the client opens a **persistent, full-duplex**
  WebSocket to a **connection server**; messages flow both ways over it (m03). (Mobile clients sometimes
  use **MQTT** — Facebook Messenger does — a lightweight pub/sub protocol good for flaky mobile links.)
- **REST/HTTP** for non-real-time: fetch message **history** (paginated by cursor), create groups, upload
  media (to blob storage, m18).
- **Connection lifecycle:** connect + auth → heartbeat/ping-pong to detect dead connections → on
  disconnect, clean up the registry; client **reconnects** (with backoff) and **syncs** missed messages.

---

## Step 4 — Data model

```
User:     { user_id, ... }
Message:  { message_id (time-sortable), conversation_id, sender_id, seq_no, text/media_url, created_at,
            status }      ← sharded by conversation_id
Conversation: { conversation_id, type (1:1/group), member_ids[] }
Inbox / per-user queue (for offline): undelivered message_ids per user
Presence: { user_id → {status, last_seen, server_id} }   ← in Redis, ephemeral
Session registry: { user_id → connection_server_id }     ← in Redis, who holds whose socket
```
- **Message store:** **write-heavy + append-mostly + queried by conversation** → a **wide-column / LSM
  store** (Cassandra/HBase/ScyllaDB) or your **MongoDB**, **sharded by `conversation_id`** so a chat's
  messages are co-located and ordered (m04 LSM, m05 shard key). Time-sortable `message_id` (Snowflake)
  for ordering.
- **Presence & session registry** live in **Redis** (ephemeral, fast, m06) — "who's online" and "which
  server holds user X's connection."

---

## Step 5 — High-level design ⭐

```
                         ┌───────── Session registry (Redis): user → server ─────────┐
                         │                                                            │
  User A ══WS══▶ [Conn Server 1] ──▶ Chat/Message service ──▶ Message store (sharded)  │
                      │                       │                                        │
                      │                       └──▶ Pub/Sub backplane (Redis/Kafka) ────┤
                      │                                   │ "deliver msg to user B"     │
  User B ══WS══▶ [Conn Server 7] ◀────────────────────────┘  (B is registered here) ──┘
                      │ push down B's socket
                      ▼
                 (B offline? → store in B's inbox + send push notification via APNs/FCM)
```

**Send a 1:1 message (A→B):**
1. A sends over its WebSocket to **Conn Server 1**.
2. Server 1 hands it to the **chat service**, which **persists** it (durability + ordering: assign a
   per-conversation `seq_no`) and **ACKs "sent"** back to A.
3. The service **looks up B in the session registry** (Redis): which connection server holds B?
4. It **routes** the message to that server via the **pub/sub backplane** (Redis pub/sub or Kafka) — this
   is how a message produced on server 1 reaches B on **server 7**.
5. **Server 7 pushes** the message down **B's WebSocket** → B's client ACKs **"delivered"**; when B views
   it, the client sends **"read"**.
6. **If B is offline:** store the message in **B's inbox/queue** and send a **push notification**
   (APNs/FCM) to wake B's app; B syncs on reconnect.

> **The two pieces that make push work** (the m03/m07 payoff): a **session registry** (`user → server`)
> so you know *where* to send, and a **pub/sub backplane** so a message can cross from the sender's server
> to the recipient's server. Without these, persistent connections don't scale.

---

## Step 6 — Deep Dive #1: connection management at scale ⭐⭐

The hard part. Maintaining ~50M **stateful** WebSocket connections:
- **Connection servers** are **stateful** — each holds a set of live sockets in memory and is **pinned**
  to those users for the connection's life. So you **can't** load-balance them like stateless app servers
  (m07): you **L4-balance the initial connection** across connection servers, then that user stays on that
  server (forced affinity).
- **Session registry:** a fast store (Redis) mapping **`user_id → connection_server_id`**, updated on
  connect/disconnect. This is how the chat service finds *where* to route a message. (Alternatively, every
  server subscribes to a **per-user topic** on the backplane and you skip the explicit registry — a
  trade-off.)
- **Heartbeats / keepalive:** ping-pong frames detect dead connections (the **slow-vs-dead** problem,
  m09 — a silent socket might be a dead client or a network blip) and let you clean up the registry +
  mark presence offline.
- **The C10M problem:** holding millions of connections per fleet is an OS/runtime challenge (file
  descriptors, memory per connection, epoll). **WhatsApp** famously pushed **~2M connections per server**
  using **Erlang/BEAM** (lightweight processes); most stacks do tens of thousands. The point: **connection
  count is the scaling dimension** — provision connection servers by *connections*, not QPS.
- **Reconnection:** on server death/deploy, that server's connections **drop**; clients **auto-reconnect**
  (with backoff + jitter, m10) and the LB routes them to a healthy server, which re-registers them and they
  **sync missed messages**. Avoid a **reconnect stampede** (jitter, m06).

> **Say this:** *"Connection servers are stateful, so I L4-balance the initial WebSocket, keep a `user →
> server` **session registry** in Redis, and use a **pub/sub backplane** to route a message to whichever
> server holds the recipient. Heartbeats detect dead connections; clients reconnect with backoff and sync.
> The scaling dimension is concurrent connections — I size the connection fleet by connection count."*

---

## Step 6 — Deep Dive #2: ordering, delivery guarantees & receipts ⭐

- **Ordering (per conversation):** assign a **monotonic `seq_no` per conversation** (the server/shard
  owning that conversation assigns it) — **don't** order by client wall-clock (clock skew, m09). The
  client renders by `seq_no`, so messages always appear in the right order within a chat. Global ordering
  across conversations isn't needed (m08 ordering-vs-parallelism: scope ordering to the conversation =
  the partition key).
- **Delivery guarantee:** **at-least-once + client dedup = effectively-once** (m08). Each message has a
  unique **message_id**; the client **dedups** on it, so a re-delivered message (after a flaky ack) isn't
  shown twice. Never "exactly-once delivery" (impossible, m09) — idempotency on the client instead.
- **Receipts (sent → delivered → read):** these are **ACKs at each stage**: server persists → **"sent"**
  to sender; recipient's client receives → **"delivered"**; recipient views → **"read"** (the blue ticks).
  Each receipt is itself a small message routed back. (E2E-encrypted apps still do delivery/read receipts
  but can't read content server-side.)

---

## Step 6 — Deep Dive #3: group chat & presence ⭐

**Group chat = fan-out (the m13 thread returns):**
- **Small groups:** **fan-out on write** — when a message is sent, deliver (route + store) to **each
  member's** connection/inbox. Fine for typical group sizes.
- **Large groups / channels** (Slack/Discord with 10k–100k+ members): naive fan-out-on-write is a storm
  (one message = 100k deliveries) → use the **hybrid** (like celebrities, m13): don't push to every
  member; members **pull/sync** the channel on read, or fan out only to **online** members + store for the
  rest. Same celebrity/hot-key pattern.

**Presence (online/offline/last-seen) — deceptively expensive:**
- Track via **heartbeats** → mark online; missed heartbeats (timeout) → offline (+ `last_seen`). Store in
  **Redis** (ephemeral).
- **The fan-out trap:** broadcasting "I'm online/offline" to **all your contacts** every time = a
  **presence storm** (millions of status updates). Mitigations: **eventual consistency** (presence can be
  slightly stale), **on-demand/lazy** presence (fetch a contact's status when you open the chat, not
  push to everyone), batching, and only notifying users currently viewing you. "Last seen" is approximate
  by design.

---

## Step 7 — Bottlenecks, failure & edge cases

- **Connection servers (stateful) are the SPOF/scaling challenge:** size by connection count, handle
  reconnect storms (jitter), and accept that a server's death drops its connections (clients reconnect).
- **Pub/sub backplane** (Redis pub/sub / Kafka) is critical for cross-server delivery → make it HA;
  Kafka adds durability/replay (good for offline delivery), Redis pub/sub is lower-latency but fire-and-
  forget.
- **Offline delivery (store-and-forward):** persist undelivered messages in the recipient's **inbox**;
  deliver on reconnect; **push notifications** (APNs/FCM, → m15) wake the app. This is why messages are
  always persisted *before* ACKing the sender.
- **Hot conversation / huge group:** a hot key (m05/m06) → shard by conversation, hybrid fan-out for huge
  groups, cache recent messages.
- **Message store:** write-heavy (~230k/s) → LSM/wide-column, sharded by conversation_id, replicated;
  retention/tiering for the PB/year.
- **Media:** upload to **blob storage + CDN** (m18/m06); send only the URL in the message.
- **Security:** **end-to-end encryption** (Signal protocol — server can't read content, complicates
  search/multi-device), auth on connect, abuse/spam controls.
- **Observability (m10):** concurrent connections, message delivery latency, **pub/sub lag**, reconnect
  rate, undelivered-queue depth; alert on delivery latency / connection churn.

**Wrap-up:** *"A real-time, write-heavy system built on **stateful WebSocket connection servers**, a
**`user→server` session registry** (Redis), and a **pub/sub backplane** (Redis/Kafka) to route a message
to the recipient's server and push it down their socket. Messages are **persisted before ACK** (durable,
per-conversation `seq_no` for ordering), delivered **at-least-once with client dedup** (effectively-once),
with **store-and-forward + push notifications** for offline users. Group chat reuses **fan-out** (hybrid
for huge groups), and **presence** is heartbeat-based + eventually consistent to avoid a fan-out storm.
The scaling dimension is **concurrent connections**. Trade-offs: WebSockets' real-time push vs the
statefulness cost; Redis pub/sub (fast) vs Kafka (durable/replayable) for the backplane; fan-out-on-write
vs hybrid for groups."*

---

## State-of-the-art & real-world notes 📚
- **WhatsApp:** **Erlang/BEAM** for massive concurrency (~2M connections/server), store-and-forward,
  Signal-protocol E2E encryption. The canonical "few engineers, billions of users" chat story.
- **Slack/Discord:** channel-based; huge channels force **hybrid fan-out**; Discord uses **Elixir/Erlang**
  + Cassandra (then ScyllaDB) for the message store. **Messenger** uses **MQTT** for mobile.
- **Protocols:** **WebSocket** (web), **MQTT** (mobile/flaky links), **XMPP** (older standard). The shared
  need: a persistent, server-push channel.
- **The recurring pattern:** persistent connections → **session registry + pub/sub backplane** — the same
  shape for any real-time push system (collaborative editing, multiplayer, live dashboards, LLM streaming
  uses the simpler one-way SSE variant, m03/m31).

---

## From your systems 🏭
- **Pub/sub backplane = your Redis:** routing a message across connection servers via **Redis pub/sub**
  (or **Kafka**) is exactly your toolset — Redis for low-latency fan-out, Kafka for durable/replayable
  delivery (m08). You'd reach for both naturally.
- **Server-push decision, lived:** Questimate's **real-time trading** (live orders/holdings/positions) is
  the same "server needs to push updates" problem — a place you faced the WebSocket-vs-polling trade-off
  (you likely polled broker APIs; you can argue when WebSockets would've been worth the statefulness cost).
- **Effectively-once via dedup:** the at-least-once + message_id dedup is your **idempotent-processing**
  reflex (DCE claim reprocessing, m08) applied to message delivery.
- **Write-heavy sharded store:** a Cassandra/Mongo message store sharded by conversation_id is your m04/m05
  knowledge (you run Mongo at 1M/day; this is the write-heavy, partition-by-entity pattern).

---

## Key concepts (interview-ready)
- **Chat needs server→client push + is write-heavy** → **WebSockets** (bidirectional persistent
  connections), which are **stateful** and break stateless load balancing (m02/m07). That tension is the
  whole design.
- **Two enablers of push at scale:** a **session registry** (`user → connection_server`) and a **pub/sub
  backplane** (Redis/Kafka) so a message reaches whichever server holds the recipient's socket.
- **Connection management** is the core: **L4-balance the initial connection** (then affinity),
  **heartbeats** to detect dead sockets (slow-vs-dead, m09), **reconnect with backoff** + sync; **scaling
  dimension = concurrent connections** (C10M; WhatsApp ~2M/server).
- **Ordering:** per-conversation **`seq_no`** assigned server-side (not client clocks, m09); scope ordering
  to the conversation (partition key, m08).
- **Delivery:** **at-least-once + client dedup (message_id) = effectively-once** (m08); **receipts** =
  ACKs (sent/delivered/read).
- **Group chat = fan-out** (m13): push for small groups, **hybrid (pull) for huge channels**.
- **Presence** = heartbeat-based, **eventually consistent**, **on-demand** to avoid a fan-out storm.
- **Offline = store-and-forward** (persist before ACK) + **push notifications** (APNs/FCM, m15). Media →
  **blob + CDN** (m18/m06). **Message store** = write-optimized (LSM/wide-column), sharded by
  conversation_id (m04/m05).

---

## Go deeper (reading)
- **WhatsApp engineering talks / "The WhatsApp Architecture Facebook Bought For $19B"** — Erlang, 2M
  connections/server, store-and-forward.
- **Discord engineering: "How Discord stores billions of messages"** (Cassandra→ScyllaDB) + their Elixir
  real-time fan-out posts.
- **System Design Interview (Alex Xu): "Design a chat system"** — the standard treatment (connection
  registry + pub/sub).
- Revisit **m03** (WebSockets/SSE/polling), **m07** (stateful connections + LB), **m08** (delivery/
  idempotency + Kafka), **m09** (ordering/slow-vs-dead), **m13** (fan-out for groups), **m15** (push
  notifications).

---

## Practice
➡️ Drill [`CROSS_QUESTIONS.md`](CROSS_QUESTIONS.md), then do [`EXERCISES.md`](EXERCISES.md) (group chat,
presence, offline, scale) before checking
[`solutions/m14_chat_system/`](../../solutions/m14_chat_system/README.md).

When you've done the exercises, say **"Module 15"** to design a *Notification System* (multi-channel
push/SMS/email, fan-out, dedup, priority, retries) — which is exactly the "wake the offline user" piece
of *this* design, generalized.
