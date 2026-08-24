---
title: "'Mentor Unavailable' Is Not a Fallback Strategy"
excerpt: "You've written this code: one try, one catch, one error response, done. Here's the Degradation Ladder — the four rungs between a clean success and a wall, and why rung 4 is the one you're probably shipping by default."
author: "Ajeet Chouksey"
authorGitHub: "ajeetchouksey"
date: "2026-08-24"
updated: null
tags: ["degradation-ladder", "named-framework", "reliability", "fallback-design", "cca-f", "production-ai"]
category: "Engineering"
readingTime: 8
featured: true
draft: false
---

You've written this code. Maybe not this exact code, but the shape of it: a `try`, a `catch`, one error response, done. It passed review because the happy path worked in the demo, and the ticket never asked "what happens when it doesn't." You shipped it and moved on. A month later, something upstream hiccups — a timeout, a rate limit, a quota you didn't know existed — and you find out in production what your fallback strategy actually was. It was nothing. It was a wall.

Here's what that wall looks like in this platform's own Worker code, powering an AI study-plan feature:

```ts
try {
  rawText = await callAnthropic(MENTOR_PLAN_SYSTEM, userContent, env.ANTHROPIC_API_KEY);
} catch (err) {
  console.error('anthropic-plan-failed:', (err as Error).message);
  return json({ error: 'mentor_unavailable' }, 503, origin);
}
```

The model call fails for any reason — a timeout, an overloaded response, a bad API key — and the user gets one JSON object: `mentor_unavailable`. No cached plan from their last session. No simplified response. No "try again in a minute" with an actual retry.

I wrote that code. It works exactly as designed — which is the problem. A system that can only succeed is a system waiting to surprise you, and if you've shipped an AI feature, you almost certainly have your own version of this sitting in production right now.

---

## The One-Bit Failure Mode

Most AI features are built with exactly one success path and exactly one failure path, and the failure path is always the same shape: catch the error, log it, return a generic 5xx. Call it the **one-bit failure mode** — the system can tell you *that* it failed, but it has no vocabulary for *how badly*, and no plan for what to do about it.

You already know the tell, because you've seen it in your own code review comments — or you've written it yourself and approved the PR anyway. One `catch` block, one error response, done. Nobody asked "what's the second-best answer we could give here?" because the ticket only scoped the happy path.

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

## The Quota You Didn't Know You Had

There's a second failure class every practitioner meets sooner or later, and it's sneakier than a model timeout: the dependency that works perfectly in every test you ran, then stops working under real usage because it had a limit nobody told you about. A rate limit. A quota. A free tier with a ceiling that only shows up once actual traffic hits it. You never budgeted for it because it never showed up in staging.

This platform hit exactly that, this week — a realtime dashboard, left open and quietly polling in the background (the same "nobody budgets for this" traffic pattern), ran a Cloudflare Worker past its daily KV-read quota. What came back wasn't a clean error. It was an uncaught exception with no CORS headers attached, which meant the browser couldn't even read the response — every failed request just showed up as a bare, diagnosis-free `Failed to fetch`. That's rung 4 by accident, which is the worst version of rung 4: nobody chose it, it just happened to whatever code path didn't have a `catch`.

The fix was rung 1, and it generalizes past this one Worker: **cache the thing you're checking on every request, so most requests never touch the thing that has a limit.** Concretely, a small in-memory cache in front of every read to the rate-limited store, checked first and only falling through on a miss:

```ts
async function cachedKvGet(kv: KVNamespace, key: string, ttlSeconds: number): Promise<string | null> {
  const mem = memGet(key);
  if (mem !== null) return mem;
  const fromKv = await kv.get(key);
  if (fromKv !== null) memSet(key, fromKv, ttlSeconds);
  return fromKv;
}
```

Nothing exotic — the cache TTLs match what the underlying store already guaranteed, so nothing gets staler than it already was. It just means the second request stops paying a cost the first one already paid. And the exception that started all this got a name instead of being left to propagate:

```ts
if (msg.toLowerCase().includes('limit exceeded')) {
  return json({ error: 'kv_quota_exceeded' }, 503, origin);
}
```

Same shape as `mentor_unavailable` — a labeled 503, not a mystery — except this one rarely fires at all now, and when it does, it tells whoever's looking exactly what happened instead of handing them a dead end.

If you've ever debugged a "random" failure that turned out to be a quota you didn't know existed, this is the same story with your dependency's name substituted in.

## Retrofitting Rung 1 into the Mentor Endpoint

The honest version of this post admits the mentor endpoint still isn't fixed. But the Degradation Ladder turns "we should add fallbacks" from a good intention into three specific questions you can ask about any feature you own — I asked them about this one:

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
