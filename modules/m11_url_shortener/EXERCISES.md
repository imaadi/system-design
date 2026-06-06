# m11 — Exercises (URL Shortener)

> Do these on paper / out loud before checking `solutions/m11_url_shortener/`. The first ones build the
> core; the later ones are **extensions/variations** (the way interviewers escalate a problem).

---

## Exercise 1 — Run the estimation yourself
Assume **500M new URLs/day**, read:write **50:1**, avg stored record **600 bytes**, keep **10 years**,
**3× replication**.
1. Writes/sec and reads/sec (avg + peak at 5×)?
2. New storage/day and total over 10 years with replication?
3. State one architectural implication per number.
4. How many base62 characters do you need so you never run out in 10 years? Show the capacity check.

---

## Exercise 2 — Base62 math
1. How many unique codes do 6, 7, and 8 base62 characters give (order of magnitude)?
2. Encode the decimal number **125** in base62 (use `0-9a-zA-Z` ordering). Show the steps.
3. Why is base62 preferred over base64 for URL codes?

---

## Exercise 3 — Pick a code-generation scheme
For each scenario, choose hash+retry / counter / KGS-ranges / Snowflake, and justify:
1. You need the **shortest possible** codes and zero collisions, at high create throughput.
2. You want **no central coordinator at all** and can accept ~11-char codes.
3. You want the **same URL to always map to the same code** (natural dedup).
4. A small internal tool, low traffic, simplicity matters most.

---

## Exercise 4 — The redirect path
1. Draw the full redirect flow including DNS, LB, cache, DB, CDN, and a cache miss.
2. Why is caching *easy* in this system compared to most (what property of the data helps)?
3. The cache cluster dies entirely. Does the system survive? At what cost, and how do you protect the
   DB during the miss storm?

---

## Exercise 5 — 301 vs 302
1. Your product team wants click analytics on every visit. Which redirect, and why?
2. Your product team wants maximum speed and minimum server load and doesn't care about analytics.
   Which redirect?
3. With 301, why can't you reliably count clicks or change the destination later?

---

## Exercise 6 — Custom aliases
1. A user requests alias `launch2027`. Describe the steps to accept or reject it.
2. How do you make the "is this alias taken?" check cheap at scale before hitting the DB?
3. How do you ensure a randomly-generated code never collides with an existing custom alias?

---

## Exercise 7 — Add analytics (without slowing redirects)
Design click analytics that capture count, timestamp, referrer, and rough geo.
1. Which redirect type is required, and why?
2. Sketch the pipeline from "click happens" to "dashboard shows counts" — name the components.
3. Why must none of this be synchronous on the redirect path?

---

## Exercise 8 — Extension: scale 10× and go global
Traffic grows 10× and users are now worldwide.
1. Which tier saturates first, and how do you scale it?
2. How do you make a redirect fast for a user in São Paulo when your origin is in Virginia?
3. A link goes viral (millions of clicks/sec to one code). What handles this and why is it the *easy*
   hot-key case here?

---

## Exercise 9 — Variation: design Pastebin
Reuse as much of this design as possible.
1. What's the same (ID generation, lookup)?
2. What changes because the payload is large text/files instead of a tiny URL? (Storage tier, read
   path/bandwidth, limits.)
3. Where does the content live vs the metadata, and why?

---

## Exercise 10 — Security & abuse + your-systems tie-in 🏭
1. Name three ways a public URL shortener can be abused, and a defense for each.
2. What consistency model does this system need, and which "-ility" must you never compromise?
3. Map three pieces of this design to patterns you've used in **DCE/Questimate** (caching, async
   analytics, idempotent create, range/precompute to avoid hot-path coordination).

---

When done, check `solutions/m11_url_shortener/README.md`, then say **"Module 12"**.
