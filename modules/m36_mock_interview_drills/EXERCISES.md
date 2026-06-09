# m36 — Mock Interview Prompt Bank + Self-Scoring

> Self-administer these **timed, out loud, whiteboard/paper, recording yourself.** Set a **45-minute
> timer**, run the §2 skeleton (m36 README), then **score with the rubric** (solutions). Do **2–3 per
> week** in your prep window. The goal isn't a "right answer" — it's **driving the framework fluently**.

---

## How to run a self-mock
1. Pick a prompt. Start the **45-min timer**.
2. **Talk the whole time** (record it) — narrate as if an interviewer is there.
3. Hit all 7 phases (reqs → estimate → API/data → high-level → deep-dive → bottlenecks → wrap).
4. Stop at 45 min. **Score yourself** with the rubric in `solutions/`. Note the 1–2 weakest phases.
5. Re-read the relevant module's CROSS_QUESTIONS for what you fumbled. Repeat next session.

---

## Prompt Bank A — Classic distributed systems (Part 1–2)
1. Design a **URL shortener** (m11). 
2. Design a **distributed rate limiter** for an API gateway (m12).
3. Design **Twitter's news feed / timeline** (m13).
4. Design **WhatsApp / a chat system** (m14).
5. Design a **notification system** (push/email/SMS) (m15).
6. Design **Google-style typeahead / autocomplete** (m16).
7. Design a **web crawler** (m17).
8. Design **S3 / an object store** (or **DynamoDB / a KV store**) (m18).
9. Design **YouTube / Netflix video streaming** (m19).
10. Design **Uber / "find nearby drivers"** (m20).
11. Design a **payment system / ledger** (m21).
12. Design a **distributed job scheduler** / **a leaderboard** / **Google Docs collab** (synthesis).

## Prompt Bank B — ML system design (Part 3)
13. Design the **ML system for a news feed ranking** (m22/m26).
14. Design a **feature store / feature pipeline** for an org (m23).
15. Design **model serving** for 10K req/s with a latency SLO (m24).
16. Design a **recommendation system** (e.g. video/products) (m25).
17. Design **ad CTR prediction + the auction** (m26).
18. Design a **real-time fraud detection** system (m27) — *your DCE world*.
19. Design an **end-to-end ML platform / MLOps** for a company (m28).

## Prompt Bank C — LLM system design (Part 4)
20. Design **LLM inference serving** at scale (cost + latency) (m29).
21. Design **production RAG** over a company's documents (m30) — *your centerpiece*.
22. Design a **customer-support LLM chatbot** (m31).
23. Design an **AI agent** that resolves support tickets / takes actions (m32).
24. Design an **LLM gateway** for an org with many LLM apps (m33).
25. Design **eval + safety** for a production LLM product (m34).
26. Design **"ChatGPT"** end-to-end (synthesis: serving + chat + RAG + gateway + eval).

## Prompt Bank D — Your systems (Part 5)
27. "Tell me about a system you designed." → **DCE** full walkthrough (m35).
28. "Tell me about a system you designed." → **Questimate** full walkthrough (m35).
29. "Walk me through the hardest technical problem you've solved." (m35)
30. "Design a healthcare claims auto-adjudication system." → *you've built it* — design it, then say so.

---

## Self-scoring (quick version — full rubric in solutions)
After each mock, rate yourself 1–5 on:
- **Requirements/NFRs** — did you clarify + name the dominant NFR before designing?
- **Estimation** — did you compute numbers AND use them to find the constraint?
- **High-level** — a working end-to-end system, one request traced?
- **Deep-dive** — 3 levels on a component, with options + trade-off + pick?
- **Bottlenecks/failure** — SPOFs, scaling the constraint, failure modes (unprompted)?
- **Communication** — drove it, checked in, quoted numbers, named patterns?

**Target: 4+ on each, cold, in 45 min.** Below 3 on any → drill that phase.

---

## Stretch drills
- **The 5-minute version:** give a *complete* answer to any prompt in **5 minutes** (forces prioritization).
- **The curveball:** mid-design, have a friend (or ChatGPT) inject "now it needs to be globally consistent"
  / "10× the traffic" / "the budget just got cut" — **adapt live.**
- **The defense:** present a design, then defend every choice against "why not [alternative]?" (m35 Q19
  posture — curiosity + both sides).

---

When you can run any Bank-A/B/C prompt at **4+ across the rubric in 45 min**, and deliver your Bank-D
(m35) walkthroughs fluently, **you're interview-ready.** Check the rubric + final checklist in
`solutions/m36_mock_interview_drills/README.md`.

🏁 **This is the final module. The curriculum is complete — go get the role.**
