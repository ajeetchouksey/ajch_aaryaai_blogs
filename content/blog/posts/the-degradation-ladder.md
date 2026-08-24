---
title: "'Mentor Unavailable' Is Not a Fallback Strategy"
excerpt: "When the model call fails, most systems return one error and stop. Here's the Degradation Ladder — the four rungs between a clean success and a wall — with two real production incidents."
author: "Ajeet Chouksey"
authorGitHub: "ajeetchouksey"
date: "2026-08-24"
updated: null
tags: ["degradation-ladder", "named-framework", "reliability", "fallback-design", "cca-f", "production-ai"]
category: "Engineering"
readingTime: 7
featured: true
draft: false
---

Somewhere in this platform's own Worker code sits this:

```ts
try {
  rawText = await callAnthropic(MENTOR_PLAN_SYSTEM, userContent, env.ANTHROPIC_API_KEY);
} catch (err) {
  console.error('anthropic-plan-failed:', (err as Error).message);
  return json({ error: 'mentor_unavailable' }, 503, origin);
}
```

That's the entire failure strategy for an AI study-plan feature. The model call fails for any reason — a timeout, an overloaded response, a bad API key — and the user gets one JSON object: `mentor_unavailable`. No cached plan from their last session. No simplified response. No "try again in a minute" with an actual retry. Just a wall.

I wrote that code. It shipped. It works exactly as designed — which is the problem. A system that can only succeed is a system waiting to surprise you.

---

## The One-Bit Failure Mode

Most AI features are built with exactly one success path and exactly one failure path, and the failure path is always the same shape: catch the error, log it, return a generic 5xx. Call it the **one-bit failure mode** — the system can tell you *that* it failed, but it has no vocabulary for *how badly*, and no plan for what to do about it.

The tell is always the same in code review: one `catch` block, one error response, done. Nobody asked "what's the second-best answer we could give here?" because the ticket only scoped the happy path.

**A system with one failure mode has zero failure modes it actually controls — it just has an outage with extra steps.**

## The Degradation Ladder

The fix is not "add better error handling." That's a slogan, not a design. The fix is to define, in advance, the ordered set of responses a feature can give when the primary path is unavailable — and to commit to trying each rung before giving up.

The **Degradation Ladder** is that ordered set. Four rungs, tried in order, top to bottom:

| Rung | Response | When it applies |
|------|----------|------------------|
| **1. Serve Stale** | Return the last known-good result, labeled as cached | A previous successful response exists and staleness is tolerable |
| **2. Serve Degraded** | Return a simplified or partial result from a cheaper path | No cached result, but a lower-fidelity answer is still useful |
| **3. Queue and Signal** | Accept the request, defer the work, tell the user when to expect it | The work is idempotent and delay is acceptable |
| **4. Human Handoff** | Return a specific, actionable message — not a generic error | None of the above apply; the user needs to know exactly what to do next |

The rule that makes this a framework and not just a list: **you fail down the ladder, never straight to the floor.** Rung 4 — the wall dressed up as a message — is only acceptable after rungs 1 through 3 have been genuinely ruled out for that specific feature, not skipped because nobody built them.

`mentor_unavailable` is a rung 4 response wearing a rung 1 problem's clothes. The mentor endpoint generates study plans from a user's existing exam progress — data the Worker already has in D1. There was never a rung-1 or rung-2 option missing because the data wasn't available. It was missing because the code only had one branch.

## A Second Incident, Same Ladder, Rung 1 This Time

Three weeks after that code shipped, a different Worker in the same repo hit a different failure — and this time the ladder held.

The GA4 analytics proxy (`aarya-ga4-proxy`) checks a Cloudflare KV store on every single request — once to verify the caller is the platform owner, once for the GA4 access token, once for the cached report response. Under normal traffic that's cheap. Under a realtime dashboard polling every 30 seconds across five open tabs, it adds up:

```
Error: KV get() limit exceeded for the day.
    at verifyOwner (ga4-proxy.js:41:27)
```

Cloudflare's free-tier KV quota is 100,000 reads per day. The Worker hit it. And because that `verifyOwner` call sat outside any `try/catch`, the exception propagated all the way to Cloudflare's platform layer, which returned a raw `500` with no CORS headers attached — the browser couldn't even read the response body, so every failed request surfaced in the UI as a bare, diagnosis-free `Failed to fetch`.

That's rung 4 by accident — the worst kind, because nobody chose it. The fix was two changes, both rung 1:

**First, an in-memory cache tier in front of every KV read.** Cloudflare keeps a Worker isolate warm across many requests. A simple `Map` at module scope, checked before KV and populated after, meant repeat requests from the same warm isolate stopped touching KV entirely:

```ts
async function cachedKvGet(kv: KVNamespace, key: string, ttlSeconds: number): Promise<string | null> {
  const mem = memGet(key);
  if (mem !== null) return mem;
  const fromKv = await kv.get(key);
  if (fromKv !== null) memSet(key, fromKv, ttlSeconds);
  return fromKv;
}
```

Auth checks cached for 10 minutes. Access tokens for 55. Report data for 15. None of it changed the actual freshness guarantees — the in-memory TTL matches the KV TTL exactly — it just meant the *second* request in a warm isolate stopped paying the KV cost the *first* one already paid.

**Second, the exception got a name.** Instead of letting a KV-quota error propagate uncaught, the catch block now recognizes it specifically and returns a real, CORS-correct response:

```ts
if (msg.toLowerCase().includes('limit exceeded')) {
  return json({ error: 'kv_quota_exceeded' }, 503, origin);
}
```

Same shape as `mentor_unavailable` — a labeled 503, not a mystery. The difference is everything upstream of it: the in-memory tier means this response fires rarely, and when it does fire, the frontend can say something a human can act on instead of a browser-generated dead end.

## Retrofitting Rung 1 into the Mentor Endpoint

The honest version of this post admits the mentor endpoint still isn't fixed. But the Degradation Ladder makes the fix specific instead of aspirational:

- **Rung 1 — Serve Stale.** The Worker already persists successful mentor-plan responses per user in D1 for analytics. Reusing that same row as a cache — return the last plan, labeled `"cached": true, "generatedAt": "..."` — turns most Anthropic outages from a wall into a slightly stale answer.
- **Rung 2 — Serve Degraded.** If no prior plan exists, a rules-based plan built from the user's existing `DomainReadiness` scores (already computed client-side for the study dashboard) beats nothing — no model call required.
- **Rung 4 — Human Handoff.** Only if both of the above are genuinely unavailable does `mentor_unavailable` get returned — and even then, paired with a specific next step ("your last plan is available at /skillup/{examId}/plan") instead of a bare error code.

None of that requires a new dependency or a bigger model. It requires answering one question per feature at design time: *what's the best answer we can give if the primary path fails, before we accept the worst one?*

## Key Takeaways

- A single `catch` block with a single generic error response is not error handling — it's an undesigned outage.
- Define your Degradation Ladder before you ship: Serve Stale → Serve Degraded → Queue and Signal → Human Handoff, tried in that order.
- Rung 4 is a last resort, not a default. If your only failure response is a generic error, you've skipped rungs 1–3 without ruling them out.
- Uncaught exceptions default to rung 4 by accident — usually the worst version of it, since a raw platform error carries zero diagnostic value for the caller.
- The data needed for rung 1 (Serve Stale) is often already sitting in your database from the last successful run. The gap is rarely data — it's that nobody wrote the second branch.

---

> **CCA-F Domain 5 — Context Management & Reliability**
>
> Designing fallback behavior for model calls — not just handling the error, but choosing what to return instead — is a scored competency under Domain 5, not incidental error handling. The Degradation Ladder is one concrete way to reason about it in an exam scenario or an architecture review.
>
> [Study Domain 5 → CCA-F Exam Prep](/skillup/ccaf)

Your next model call will fail. Not might — will. The only open question is which rung it lands on, and whether you chose that in advance or found out in production.
