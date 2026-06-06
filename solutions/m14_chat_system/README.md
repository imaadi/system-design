# m14 — Solutions (Chat System)

> Check the *reasoning*. Recurring lessons: server-push + write-heavy → stateful WebSockets → session
> registry + pub/sub backplane; scale by connection count; per-conversation seq_no; at-least-once + dedup;
> store-and-forward for offline.

---

## Exercise 1 — Estimation
800M DAU, 30 msgs, 12% online, 200 B.
1. **Messages:** 800M × 30 = 24B/day ÷ 10⁵ = **~240,000/sec** avg (~1M peak). Write-heavy.
2. **Concurrent connections:** 12% × 800M = **~96M concurrent**. At 80k/server → **~1,200 connection
   servers** (plus HA headroom).
3. **Storage:** 24B × 200 B = **~4.8 TB/day** → ~1.75 PB/year (sharded, with retention).
4. **Concurrent connections (~96M)** most shapes it — it's the scaling dimension unique to chat (you size
   the stateful connection fleet by it, plus the registry + backplane to route across those servers). QPS
   shapes the message store, but connections drive the hard part.

---

## Exercise 2 — Why WebSockets
1. HTTP is **client-initiated** — the server can't push a newly-arrived message to B without B asking, and
   chat needs **server→client push**.
2. **WebSockets** are **full-duplex** (both sides send frequently); **SSE** is one-way (server→client
   only, fine for an LLM token stream but not for B *also* sending), and **long-polling** is a wasteful
   hack with a connection held per request. Chat's two-way real-time traffic fits WebSockets.
3. **WebSockets are stateful/persistent** (a connection pinned to one server) — which breaks stateless
   load balancing and forces the session-registry + pub/sub-backplane design.

---

## Exercise 3 — The routing problem
1. A → its WebSocket to **conn-server-1** → **chat service** persists (seq_no) + ACKs "sent" → **look up B
   in the session registry (Redis)** → B is on **conn-server-7** → route via **pub/sub backplane (Redis/
   Kafka)** → conn-server-7 **pushes** down **B's socket** → B ACKs "delivered." (Offline B → inbox + push.)
2. A **session registry** (`user → connection server`) and a **pub/sub backplane** (to carry the message
   between the two servers).
3. **Redis pub/sub** = low-latency, fire-and-forget (no durability); **Kafka** = durable + replayable, a
   bit more latency (better for guaranteed/offline delivery).

---

## Exercise 4 — Connection management
1. Because each connection is a **long-lived, stateful socket pinned to one server** — you can't route an
   arbitrary request for that user to any server; their frames must go to the server holding their socket.
2. The **session registry** maps **`user_id → connection_server_id`** ("where is this user connected"),
   updated on connect/disconnect, so the chat service knows where to route a message.
3. **Heartbeats (ping/pong)** — missed heartbeats → treat as dead, clean up registry, mark offline. It
   echoes the **slow-vs-dead** problem (m09): a silent socket is ambiguous, so the heartbeat timeout is a
   guess (too short → flapping; too long → stale).
4. The server's connections **drop** → clients **auto-reconnect with backoff + jitter** → LB routes them
   to a **healthy** server → it **re-registers** them and they **sync missed messages**. Persistence means
   nothing is lost in the gap.

---

## Exercise 5 — Ordering & delivery
1. Assign a **monotonic `seq_no` per conversation** server-side (by the shard owning the conversation);
   clients render by `seq_no`. **Not timestamps** because client/server **clock skew** (m09) would
   misorder messages.
2. **At-least-once + client dedup = effectively-once.** Each message has a unique **message_id**; the
   client dedups on it so a re-delivered message isn't shown twice (exactly-once delivery is impossible,
   m09).
3. **Persist before ACK** so a crash after "received" but before storing doesn't **lose** the message —
   acking first would make it at-most-once. Persisting first means a crash → redelivery (at-least-once),
   which dedup handles.
4. **Sent** = server persisted it (ACK to sender, one tick); **delivered** = recipient's client received
   it (two ticks); **read** = recipient viewed it (blue ticks). Each is a small ACK routed back via the
   registry + backplane.

---

## Exercise 6 — Offline delivery
1. It's **persisted in the recipient's inbox/undelivered queue** (store-and-forward) — it was persisted
   before ACK anyway.
2. A **push notification** (APNs/FCM) wakes their app; on **reconnect** the client **syncs** missed
   messages (by last-seen seq_no/cursor) and they're delivered + marked delivered.
3. Store-and-forward works **only because you persist the message before ACKing the sender** (Ex 5.3) — so
   the message survives the recipient being absent and can be delivered later, rather than being lost.

---

## Exercise 7 — Group chat
1. **Fan-out on write** — deliver/store the message to each of the 20 members (route to online members,
   store for offline).
2. For **200k members**, fan-out-on-write is a **storm** (one message = 200k deliveries). Instead use the
   **hybrid** (like celebrities, m13): fan out to **online** members and have others **pull/sync** the
   channel on read, rather than pushing to all 200k.
3. It's the **same celebrity/hot-key problem** (m13/m05/m06): one message/entity causing disproportionate
   fan-out → special-case the huge case with pull instead of push.

---

## Exercise 8 — Presence
1. Broadcasting status to **all contacts** on every change = a **fan-out storm** (millions of users
   flipping online/offline → a flood of update messages), wasteful and hard to scale.
2. **Mitigations:** eventual consistency (slightly stale is fine), **on-demand/lazy** presence (fetch a
   contact's status when you open the chat, don't push to everyone), **batch** updates, only notify users
   currently viewing you.
3. **Eventually consistent** — acceptable because presence is **non-critical** (a "last seen" off by a few
   seconds, or a brief "online" lag, harms nothing), so the coordination cost of strong consistency isn't
   worth it.

---

## Exercise 9 — Extensions
1. **Media:** upload the 20 MB video to **object/blob storage (S3/MinIO)**, send only the **URL +
   metadata** in the message; recipients **download via a CDN** (m18/m06). Keeps the message pipeline
   light.
2. **E2E encryption (Signal protocol):** the **server relays only ciphertext** it can't read. Complicates
   **server-side search** (can't index content) and **multi-device sync** / **push content** (can't show
   message text in the notification), plus group key management.
3. **Slack vs WhatsApp:** Slack is **channel-centric** with **searchable history** (so **not** E2E —
   server indexes content, m16) and large channels (more hybrid fan-out); WhatsApp is **contact-centric**,
   **E2E-encrypted**, 1:1 + smaller groups, mobile store-and-forward. Different fan-out/encryption/search
   trade-offs on the same connection+routing core.

---

## Exercise 10 — Bottlenecks + your-systems tie-in 🏭
1. **Connection servers** (stateful → size by connection count, handle reconnect storms with jitter);
   **pub/sub backplane** (HA Redis/Kafka — all cross-server delivery flows through it); **session
   registry** (Redis — replicate/shard); **message store** (write-heavy → shard by conversation_id +
   replicate; hot conversations); **presence fan-out** (lazy + eventual). Each: redundancy + the listed
   mitigation.
2. **Pub/sub backplane = Redis pub/sub (fast) or Kafka (durable/replayable)** — your exact toolset (m08);
   **effectively-once delivery = at-least-once + dedup**, the same **idempotent-processing** pattern as
   DCE claim reprocessing (a redelivered claim/message mustn't double-act).
3. **Questimate real-time trading:** the server needed to push live order/holding/position updates — a
   server-push problem. **WebSocket** would give true real-time bidirectional updates (worth it if updates
   are frequent and latency-critical) at the cost of **statefulness** (connection servers, registry);
   **polling** (what you likely did against broker APIs) is simpler and stateless but wastes requests and
   adds latency. The trade: real-time push vs operational simplicity — choose by how real-time the UX
   truly needs to be.
