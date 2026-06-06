# m14 — Cross-Questions ("if-and-buts") on the Chat System

> Answer out loud in 2–3 sentences before reading the model answer. Connection management, message
> routing, ordering, and delivery guarantees are where this is won.

---

### Q1. Why can't you build chat with normal HTTP request/response?
**A.** Because chat needs the **server to push** to clients in real time (a message arrives for B without
B asking), and HTTP is **client-initiated** — the server can't speak unless asked. You also need
**bidirectional** flow (both sides send/receive constantly). So you use **WebSockets** — a persistent,
full-duplex connection — which lets the server push down the socket. (Long-polling/SSE are weaker
fallbacks; WebSocket fits because both directions are active.)

---

### Q2. WebSockets are stateful — why is that a problem, and how do you handle it?
**A.** A WebSocket is a **long-lived connection pinned to one server**, so you **can't** freely load-
balance requests across stateless servers (m07): that user must keep hitting the server holding their
socket. You handle it by **L4-balancing the initial connection** across connection servers (then
affinity), keeping a **session registry** (`user → server`) so others know where to route, and a **pub/
sub backplane** so a message from a user on server A reaches a user on server B. Statefulness is the
central cost of real-time push.

---

### Q3. User A (on server 1) sends a message to user B (on server 7). Walk me through it.
**A.** A sends over its WebSocket to **server 1** → server 1 hands it to the **chat service**, which
**persists** it (durable, assigns a per-conversation `seq_no`) and ACKs **"sent"** to A → the service
**looks up B in the session registry** (Redis: B is on **server 7**) → **routes** the message via the
**pub/sub backplane** (Redis/Kafka) to server 7 → **server 7 pushes** it down **B's socket** → B's client
ACKs **"delivered."** If B is offline, store in B's inbox + send a **push notification**.

---

### Q4. What's the scaling dimension for chat — QPS or something else?
**A.** **Concurrent connections.** Unlike most systems (where you scale by request throughput), chat's
constraint is the number of **simultaneous open WebSocket connections** you must hold — e.g. ~50M for
500M DAU at 10% online. Each consumes memory + a file descriptor on a connection server, so you provision
the **connection fleet by connection count** (a server holds ~tens of thousands to, with Erlang like
WhatsApp, ~2M). QPS matters for the message store, but connections drive the connection tier.

---

### Q5. What is the session registry and why do you need it?
**A.** A fast store (Redis) mapping **`user_id → connection_server_id`** — "which server currently holds
this user's socket." You need it because connection servers are **stateful and sharded**: to deliver a
message to B, the chat service must know **where B is connected** so it can route the message to that
specific server (via the backplane). It's updated on every connect/disconnect. (An alternative is
per-user topics on the backplane that each server subscribes to, avoiding an explicit registry.)

---

### Q6. What's the pub/sub backplane for, and Redis pub/sub vs Kafka?
**A.** It carries a message **from the sender's connection server to the recipient's** connection server
(they're different machines). **Redis pub/sub** is **low-latency but fire-and-forget** (no durability —
if the subscriber is momentarily gone, the message is lost). **Kafka** adds **durability + replay** (good
for guaranteed/offline delivery) at slightly higher latency. Many designs use **both**: Redis pub/sub for
the fast online path, plus a durable store/Kafka for offline/guaranteed delivery. You pick by the
delivery guarantee you need.

---

### Q7. How do you guarantee message ordering within a conversation?
**A.** Assign a **monotonic `seq_no` per conversation**, generated **server-side** by the shard that owns
that conversation — the client renders by `seq_no`, so messages always appear in order within a chat.
You **don't** order by client wall-clock (clock skew, m09, would misorder). Global ordering across
conversations isn't needed (m08: scope ordering to the conversation = the partition key), which keeps it
scalable.

---

### Q8. What delivery guarantee do you provide, and how do you avoid duplicate messages?
**A.** **At-least-once + client-side dedup = effectively-once** (m08). Exactly-once *delivery* is
impossible over an unreliable network (m09), so messages may be re-delivered (e.g. an ack was lost). Each
message has a unique **message_id**; the **client dedups** on it, so a re-delivered message isn't shown
twice. You **persist before ACKing the sender** so nothing is lost on a crash (acking before persisting
would make it at-most-once → lost messages).

---

### Q9. How do the "sent / delivered / read" receipts work?
**A.** They're **ACKs at each stage**, each a small message routed back: when the server **persists** the
message it ACKs **"sent"** to the sender (one tick); when the **recipient's client receives** it, it sends
**"delivered"** (two ticks); when the recipient **views** the chat, the client sends **"read"** (blue
ticks). Each receipt travels back through the same routing (registry + backplane). E2E-encrypted apps
still send delivery/read receipts even though the server can't read the message content.

---

### Q10. What happens when the recipient is offline?
**A.** **Store-and-forward:** the message is **persisted** in the recipient's **inbox/undelivered queue**
(it was persisted before ACK anyway), and a **push notification** (APNs/FCM) is sent to wake their app.
When they **reconnect**, the client **syncs** all missed messages from the inbox (by last-seen seq_no/
cursor) and they're delivered + marked delivered. This is why persistence happens before delivery — the
message survives the recipient being gone, and "delivered" simply fires later on reconnect.

---

### Q11. A connection server crashes (or you deploy). What happens to its users?
**A.** Its WebSocket connections **drop**. Clients detect the drop and **auto-reconnect with backoff +
jitter** (m10), the LB routes them to a **healthy** connection server, which **re-registers** them in the
session registry and they **sync missed messages**. You design for graceful reconnection and avoid a
**reconnect stampede** (jitter spreads the reconnects, m06). Because messages are persisted, nothing is
lost during the gap — they're delivered on resync.

---

### Q12. How do you detect that a connection is actually dead vs just idle/slow?
**A.** **Heartbeats** — periodic ping/pong frames. If a client misses N heartbeats, the server treats the
connection as dead, **cleans it from the registry**, and marks the user **offline**. This is the
**slow-vs-dead** problem (m09): a silent socket could be a dead client, a frozen app, or a network blip,
so the heartbeat timeout is a **guess** (too short → false disconnects/flapping; too long → stale
presence + wasted resources). Same tuning tension as failure detection everywhere.

---

### Q13. How do you design group chat, and what breaks for huge groups?
**A.** Group chat = **fan-out** (m13): for **small groups**, **fan-out on write** — deliver/store the
message to **each member** (route to online members, store for offline). For **huge groups/channels**
(10k–100k+ members, Slack/Discord), naive fan-out-on-write is a **storm** (one message = 100k deliveries)
→ use the **hybrid** (like celebrities, m13): don't push to everyone — fan out to **online** members and
have others **pull/sync** the channel on read. Same celebrity/hot-key pattern as the news feed.

---

### Q14. Why is presence ("online"/"last seen") harder than it looks?
**A.** Because naive presence is a **fan-out storm**: if every status change broadcasts "I'm online/
offline" to **all** your contacts, millions of users flipping status = a flood of updates. Mitigations:
make presence **eventually consistent** (slightly stale is fine), compute it **on-demand/lazily** (fetch a
contact's status when you open their chat rather than pushing to everyone), **batch** updates, and only
notify users currently viewing you. "Last seen" is **approximate by design** — exactness isn't worth the
cost.

---

### Q15. Where and how do you store messages, and why?
**A.** In a **write-optimized, append-mostly store sharded by `conversation_id`** — a **wide-column/LSM
DB** (Cassandra/ScyllaDB/HBase) or MongoDB. Reasons: chat is **write-heavy** (~230k/s) so LSM's sequential
writes fit (m04); querying is **by conversation** (fetch a chat's recent messages) so sharding by
conversation_id co-locates and orders them (m05); time-sortable message_ids give ordering. You add
retention/tiering for the PB/year and replicate for durability.

---

### Q16. How do you handle media (images/videos) in messages?
**A.** Don't send media through the message pipeline. **Upload media to object/blob storage (S3/MinIO)**,
get a URL, and send only the **URL + metadata** in the message; recipients **download the media via a
CDN** (m18/m06). This keeps the message store small and the real-time path light, and lets large media be
served from the edge. (Same "split metadata from blobs" pattern as m04/m11-Pastebin.)

---

### Q17. How would you add end-to-end encryption, and what does it complicate?
**A.** Use the **Signal protocol** (X3DH + Double Ratchet): clients exchange keys and the **server only
relays ciphertext** it can't read. It complicates: **server-side search** (can't index content),
**multi-device** sync (keys per device), **push notifications** (can't include message content), and
**group key management**. You still do delivery/read receipts and routing (those are metadata). The trade:
strong privacy vs lost server-side features — a deliberate product choice (WhatsApp/Signal do it,
Slack/Discord generally don't for searchability).

---

### Q18. How is designing Slack different from WhatsApp?
**A.** Slack is **channel/workspace-centric** with **large, persistent channels** and rich history/search,
so: **huge-channel fan-out** needs the hybrid (pull) treatment more often; **searchable history** means
**not** E2E-encrypted (server indexes content into a search store, m16); **threads/reactions/edits** add
data model complexity; and presence/notifications are workspace-scoped. WhatsApp is **contact-centric**,
mostly **1:1 + smaller groups**, **E2E-encrypted**, mobile-first (store-and-forward + push heavy). Same
connection/routing core; different fan-out, encryption, and search trade-offs.

---

### Q19. What are the main bottlenecks and SPOFs?
**A.** **Connection servers** (stateful; sized by connection count; reconnect storms on deploy/crash);
the **pub/sub backplane** (must be HA — Redis/Kafka cluster; it's how all cross-server delivery flows);
the **session registry** (Redis — replicate/shard); the **message store** (write-heavy → shard +
replicate, hot conversations); and **presence fan-out**. Observability (m10): concurrent connections,
delivery latency, **backplane/consumer lag**, reconnect rate, undelivered-queue depth — alert on delivery
latency and connection churn.

---

### Q20. How does this connect to the notification system (m15)?
**A.** The **"wake the offline user"** piece **is** a notification system: when the recipient is offline,
chat hands off to **push notifications** (APNs for iOS, FCM for Android) to alert them. m15 generalizes
this — multi-channel (push/SMS/email), fan-out, deduplication, rate-limiting, priority, and retries.
So chat is a *producer* of notifications, and the offline-delivery path you designed here is a concrete
instance of the general notification pipeline you'll build next.
