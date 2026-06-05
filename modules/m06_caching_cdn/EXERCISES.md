# m06 — Exercises (Caching & CDNs)

> Do these on paper / out loud before checking `solutions/m06_caching_cdn/`. Goal: pick the right tier
> + pattern, reason about invalidation/consistency, and diagnose the failure modes.

---

## Exercise 1 — Hit-ratio math
A read endpoint does **50,000 reads/sec**. Each cache miss costs one DB query.
1. At a **95%** hit ratio, how many DB queries/sec? At **99%**? At **99.9%**?
2. The cache cluster goes fully down (hit ratio → 0). How many DB queries/sec now, and by what factor
   did DB load jump vs the 99% case?
3. What does this tell you about how to treat the cache operationally?

---

## Exercise 2 — Pick the tier
For each, which cache tier (browser / CDN / app-local in-process / distributed Redis / DB buffer pool)
and why:
1. A site's logo and JS bundle, served to users worldwide.
2. A feature-flag config read on every request by every app server, changes rarely.
3. Hot eligibility/reference data read during claim scoring, shared across all workers.
4. A single user's "recently viewed" list, only they see it.
5. The 10 hottest product keys that are read tens of thousands of times/sec.

---

## Exercise 3 — Choose the pattern
For each, choose a read pattern (cache-aside / read-through) AND a write pattern (write-through /
write-around / write-back), and justify:
1. A user profile read constantly, updated occasionally; staleness of a few seconds is fine.
2. A view-counter incremented millions of times/sec; an occasional lost increment is acceptable.
3. Bulk-imported reference data written once, read heavily; must be fresh immediately after import.
4. An account balance read often but must never be stale.

---

## Exercise 4 — Invalidate or update?
A product's price changes in the DB.
1. Should you **update** the cached price in place or **delete** the key? Give the race-condition reason.
2. You choose to delete-on-write. Describe the failure where the DB and cache still diverge, and name
   the underlying problem.
3. What's the simplest mechanism that bounds staleness even if your invalidation logic has a bug?

---

## Exercise 5 — Name the failure mode
For each scenario, name the failure (stampede/thundering herd, penetration, avalanche, hot key) and
give one fix:
1. A celebrity posts; one Redis node hosting their profile key saturates.
2. Attackers request `user:<random>` for millions of non-existent ids; the DB is hammered.
3. At midnight, 1M keys that were all set with TTL=3600 at 11pm expire together; the DB spikes.
4. A popular homepage query's cached result expires; 5,000 concurrent requests all hit the DB at once.

---

## Exercise 6 — Kill the stampede
You have one very expensive query whose result is cached with a TTL. When it expires, you get a
thundering herd.
1. Describe **single-flight (request coalescing)** and how it limits DB hits to one.
2. Describe **stale-while-revalidate** and why users never wait.
3. Describe **jittered TTL + probabilistic early refresh** and what problem each addresses.

---

## Exercise 7 — CDN design
You serve a global product: a JS/CSS/image bundle, user avatars, and large product videos.
1. Which of these go on a CDN, and why?
2. You ship a new CSS file every deploy. How do you make sure users get the new one **without** doing a
   global CDN purge each time?
3. Pull vs push CDN for the large product videos at launch — which and why?
4. Name two benefits of the CDN beyond latency.

---

## Exercise 8 — HTTP caching headers
1. You serve `app.[contenthash].js`. What `Cache-Control` would you set and why?
2. You serve a user's private account page. What `Cache-Control` directive(s) and why?
3. A client has a cached resource and wants to check if it changed cheaply. What header does it send,
   and what status does the server return if unchanged?
4. What's the difference between `no-cache` and `no-store`?

---

## Exercise 9 — When NOT to cache
For each, say whether caching helps and why/why not:
1. A write-heavy audit log that's almost never re-read.
2. A search where nearly every query string is unique.
3. A bank balance shown on a transfer-confirmation screen.
4. A product catalog read 100× more than it's written.

---

## Exercise 10 — Apply to your own systems 🏭
1. DCE caches hot reference/eligibility data in **Redis** and reads it during scoring. Which pattern is
   this, what failure mode could a Redis flush trigger, and how would you protect the DB?
2. Questimate **precomputes risk into snapshot tables**. Frame this as caching: what's being cached,
   what's the "TTL"/refresh, and what's the staleness/consistency risk?
3. Questimate's **Redis-backed JWT sessions** — how is this caching, and how does invalidation become a
   feature here (instant revocation)?
4. If Questimate's read load grew 10×, order these levers and justify: add Redis cache, add read
   replicas, add a CDN, shard Postgres.

---

When done, check `solutions/m06_caching_cdn/README.md`, then say **"Module 7"**.
