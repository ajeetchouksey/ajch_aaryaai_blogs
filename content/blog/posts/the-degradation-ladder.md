---
title: "Your Retry Loop Is Not a Fallback Strategy"
excerpt: "Retrying a failed model call three times and then throwing a 500 isn't resilience — it's denial with extra steps. Here's the five-rung ladder that separates systems that degrade gracefully from ones that just go dark."
author: "Ajeet Chouksey"
authorGitHub: "ajeetchouksey"
date: "2026-08-24"
updated: null
tags: ["degradation-ladder", "named-framework", "ai-architecture", "resilience", "fallback-strategy", "cca-f"]
category: "AI Architecture"
readingTime: 7
featured: true
draft: false
---

At 11:42 UTC, a model provider starts returning `529 Overloaded` for roughly one in eight requests. Nothing dramatic — no status page banner yet, no incident number. Your Worker's retry logic does exactly what it was told: back off, try again. By 11:47, the error rate has climbed to 61%, your retry storm has tripled outbound request volume against a provider that is already struggling, and every user waiting on a response is staring at a spinner that will never resolve. Nobody wrote a fallback. They wrote a retry loop and called it done.

That's the failure mode this post is about. Retrying is not a strategy — it's a delay tactic that works right up until the outage outlasts your patience budget, and then it fails exactly the same way a system with no retries at all would fail: completely, silently, and all at once.

---

## Retrying Is Not Falling Back

A retry loop answers one question: *is this failure transient?* It does not answer the question that actually determines whether your users notice an outage: *what do we do if it isn't?* Most production AI systems have a good answer to the first question and no answer at all to the second — three attempts, exponential backoff, jitter, and then a bare `throw`. That's not a fallback strategy. That's hoping the provider recovers before your timeout does.

The tell is what happens after the last retry exhausts. If the answer is "the request fails and the caller sees an error," you have retry logic, not resilience. **A retry loop that has no plan for permanent failure isn't buying you time — it's just moving the failure three requests later.**

## The Degradation Ladder

The fix is a named structure worth citing in a design review: the **Degradation Ladder** — an ordered sequence of response strategies a system falls through, from full capability to honest failure, each rung tried only after the one above it fails.

The ordering matters as much as the rungs themselves. Each rung is strictly worse for the user than the one before it, and the system should never skip down two rungs at once — jumping straight from "healthy" to "human handoff" wastes a support queue's capacity on requests a cache or a second model could have handled cleanly.

| Rung | Strategy | User experience |
|------|----------|-----------------|
| 1 | Second provider | Full capability, different vendor, imperceptible |
| 2 | Cached answer | Slightly stale, clearly labeled |
| 3 | Degraded mode | Reduced capability, honest about the reduction |
| 4 | Human handoff | Slower, but a person, not a wall |
| 5 | Fail closed | Explicit outage message, zero hallucination |

## Rung 1 — Try a Second Provider

This is the rung most teams reach for first and the one fewest teams can actually execute, because it has a prerequisite most codebases don't meet: a call site that doesn't already know which vendor it's talking to. If your model call is named `callAnthropic()` and its return type has vendor-specific fields bolted on, there is no clean place to insert "try OpenAI instead" — you'd be threading a second vendor's request and response shapes through every caller that currently assumes the first one. I covered this exact prerequisite in [If Your Function Is Named callAnthropic(), You Don't Have an Abstraction](/blog/the-boring-interface): rung 1 of a real fallback strategy is "try a second provider," and that rung doesn't exist until the abstraction does.

With a boring interface in place, rung 1 is a routing decision, not a rewrite:

```ts
async function generateCompletion(call: ModelCall, env: Env): Promise<string> {
  try {
    return await callAnthropic(call, env.ANTHROPIC_API_KEY);
  } catch (err) {
    if (isTransientProviderError(err)) {
      return await callOpenAI(call, env.OPENAI_API_KEY); // rung 1
    }
    throw err;
  }
}
```

**Rung 1 only works if the abstraction was boring before the outage started — building it during an incident is not a fallback plan, it's a rewrite under pressure.**

## Rung 2 — Serve the Cached Answer

Most conversational AI traffic is more repetitive than teams assume — the same study-plan prompt, the same FAQ phrasing, the same order-status question, arriving from different users within minutes of each other. Cache the response, key it on a normalized hash of the input, and when both providers are down, serve the cached answer with an honest staleness label instead of nothing.

The config that makes this defensible, not deceptive: a KV cache with a 15-minute TTL for idempotent generations (study plans, policy answers, FAQ responses), a hard exclusion for anything containing account-specific or time-sensitive data (order status, balances — never cache those), and a visible "this answer may be a few minutes old" flag on every cache hit served during degraded operation. Cached-and-labeled beats fresh-and-wrong. Cached-and-hidden is worse than either.

## Rung 3 — Degraded Mode

Degraded mode means the system stays up but does less: swap to a smaller, faster model that's less likely to be capacity-constrained during the same incident (Haiku instead of Sonnet, for instance), disable tool-calling so the agent can't attempt actions it can't verify, and cap output length so a half-working model produces a short, checkable answer instead of a long, confident, ungrounded one. The Domain Boundary still applies here — degraded mode narrows what the system *attempts*, it doesn't loosen what the system is *allowed* to attempt.

The rule that keeps this rung honest: **degraded mode reduces capability, it never simulates capability.** A system that pretends nothing changed while quietly running a worse model is lying to its users with better production values than usual.

## Rung 4 — Human Handoff

When rungs 1 through 3 have all failed or aren't applicable — the request needs live account data, both providers are down, and there's no safe cached or degraded answer — hand off to a person. This only works if it's designed in advance: a ticketing queue (Zendesk, Freshdesk, an internal support tool) wired to receive the conversation transcript and the reason for escalation, not a dead end that tells the user to "try again later" with no path forward.

**A human handoff that arrives without context is not an escalation — it's a second failure wearing a person's name badge.**

## Rung 5 — Fail Closed, Honestly

The bottom rung exists because sometimes every rung above it is unavailable — no second provider, no valid cache entry, no degraded mode that fits the request, no support queue with capacity. At that point the only correct move is an explicit, honest failure: "We can't process this right now — please try again in a few minutes," with zero attempt to generate an answer anyway. This is the rung most incidents actually violate, not because engineers want to hallucinate under load, but because a bare exception with no rung below it produces exactly that: a raw error, a broken UI state, or — worse — a model call that succeeds against garbage cached context and returns something confidently wrong.

## The Same Incident, Two Architectures

Same 11:42 UTC incident, same 61% error rate by 11:47, two different outcomes:

**No ladder:** retries triple the request volume against an already-struggling provider, every request either times out or returns a raw 500, users see spinners followed by broken UI, and the incident channel fills with "is the site down?" messages that could have been prevented with a two-line status banner.

**With the ladder:** the circuit breaker opens after 5 consecutive failures in a 60-second window, rung 1 routes the next 20 minutes of traffic to the second provider at slightly higher per-token cost, cache hits absorb another chunk of repeat traffic at rung 2, and the incident resolves without a single user-facing error. The provider's outage becomes an internal metrics blip instead of a support ticket spike.

Same failure. The difference is entirely architectural, and it was decided weeks before the incident, not during it.

## Checklist — Build the Ladder Before You Need It

- Do you have a boring interface behind your model call, or does rung 1 require a rewrite mid-incident?
- Is your circuit breaker threshold defined in code (e.g., open after 5 consecutive failures or >50% error rate over 60s), not left to unbounded retries?
- Do you cache idempotent responses with a TTL and a visible staleness label — and explicitly exclude account-specific data?
- Does your degraded mode reduce capability (smaller model, no tools, shorter output) rather than silently swap it?
- Does your human handoff carry the transcript and escalation reason, or does it dump the user into a queue with no context?
- Does your bottom rung fail with an honest message — never a hallucinated answer generated against a broken context?

---

> **CCA-F Domain 1 — Agentic Architecture & Orchestration**
>
> Designing what an orchestration layer does when the model itself is unavailable — routing, circuit breaking, and escalation — is core Domain 1 material. The Degradation Ladder is the concrete shape of that requirement: an ordered, testable fallback sequence, not a single retry block.
>
> [Study Domain 1 → CCA-F Exam Prep](/skillup/ccaf)

A ladder only works if every rung is load-bearing before the fall. Build yours on a quiet Tuesday — the outage will not wait for you to design it live.