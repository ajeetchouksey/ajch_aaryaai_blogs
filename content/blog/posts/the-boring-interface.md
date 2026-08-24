---
title: "If Your Function Is Named callAnthropic(), You Don't Have an Abstraction"
excerpt: "You've named a function after a vendor at least once. Here's why that's not a style nit — it's the reason swapping models later means touching every call site instead of one."
author: "Ajeet Chouksey"
authorGitHub: "ajeetchouksey"
date: "2026-08-24"
updated: null
tags: ["boring-interface", "named-framework", "ai-architecture", "vendor-lock-in", "llm-abstraction", "cca-f"]
category: "AI Architecture"
readingTime: 6
featured: true
draft: false
---

You've named a function after a vendor at least once. `callAnthropic`. `callOpenAI`. `stripeCharge` instead of `chargeCard`. It felt harmless when you wrote it — there was one provider, the name was accurate, and accurate names are supposed to be good. Then the model gets deprecated, or the pricing changes, or you want to route the cheap classification calls somewhere cheaper than the model you use for real reasoning — and you find the vendor's name isn't just in the function. It's in fourteen call sites, three type definitions, and a system prompt that assumes a specific model's quirks.

Here's that exact function, in this platform's own Worker code, doing exactly what it says on the tin:

```ts
const ANTHROPIC_API = 'https://api.anthropic.com/v1/messages';

async function callAnthropic(system: string, userContent: string, apiKey: string): Promise<string> {
  const res = await fetch(ANTHROPIC_API, { /* ... */ });
  // ...
}
```

Two call sites depend on it today — a study-plan generator and a chat mentor. Both would need to change, by hand, the day this platform wants a second model provider for anything: a cheaper model for simple turns, a fallback when Anthropic has an incident, or just a different vendor entirely. The function works. That's not the problem. The problem is that the name told you which vendor before it told you what the function does — and that ordering is a decision you're going to regret exactly once, at the worst possible time.

---

## The Tell Is in the Name

A function name is supposed to describe a capability: *generate text from a prompt*, *charge a payment method*, *fetch analytics for a date range*. When the name describes a vendor instead — `callAnthropic`, not `generateCompletion` — the vendor has leaked out of the implementation and into the contract every caller depends on. You didn't just write a function. You wrote a promise that this specific vendor is involved, everywhere the function is called, forever, until someone does a find-and-replace across the codebase.

That's not a style opinion. It has a concrete cost, and you've paid it before even if you didn't name it: **you cannot swap, test, or fall back to a second provider without touching every call site**, because every call site knows the vendor's name. A [Degradation Ladder](/blog/the-degradation-ladder) needs somewhere to fail *to* — rung 1 of a real fallback strategy is often "try a second provider" — and that rung doesn't exist if the first provider's name is load-bearing throughout your code.

## The Boring Interface

The fix has a name because it's worth being able to cite in a design review: **the Boring Interface**. An interface is boring, in the useful sense, when it satisfies three properties:

1. **The name describes the capability, not the vendor.** `generateCompletion`, not `callAnthropic`.
2. **The input and output shapes carry no vendor-specific fields.** If your return type has an `anthropic_stop_reason` field bolted on, the abstraction already leaked.
3. **Swapping the implementation touches exactly one place** — the adapter behind the interface — never the call sites that use it.

None of that requires a plugin system, a config file, or an abstract factory pattern borrowed from a language that isn't yours. It requires one function whose name and shape don't know what's behind them:

```ts
interface ModelCall {
  system: string;
  userContent: string;
}

// The vendor lives here, and only here.
async function generateCompletion(call: ModelCall, env: Env): Promise<string> {
  return callAnthropic(call.system, call.userContent, env.ANTHROPIC_API_KEY);
}
```

Nothing about this is clever. That's the point — a boring interface is one nobody has to think about except the person maintaining the one function behind it. **The measure of a good abstraction isn't how much it can do. It's how little anyone has to know to use it correctly.**

## The Pattern Already Exists in This Same Repo

This isn't a hypothetical, and it isn't even a new pattern here — it's just not applied to the model call yet. The GA4 analytics Worker in this same platform already does this correctly, for a different kind of provider swap: it can authenticate to Google either via a service account or via OAuth, and every caller is completely indifferent to which:

```ts
async function getAccessToken(env: Env): Promise<string> {
  const cached = await cachedKvGet(env.GA4_CACHE, 'ga4:access_token', 3300);
  if (cached) return cached;
  if (env.GA4_SERVICE_ACCOUNT_B64) return getAccessTokenSA(env);
  return getAccessTokenOAuth(env);
}
```

`runRealtimeReport` and `runDataReport` call `getAccessToken(env)` and get a string back. Neither one knows or cares whether that string came from a signed JWT exchange or a stored refresh token. The choice happens in exactly one place, and it happens by looking at what's configured — not by every caller carrying an `if (usingServiceAccount)` branch of its own. That's a Boring Interface, already shipped, already tested, sitting one file away from the function that isn't one yet.

## What Retrofitting `callAnthropic` Actually Looks Like

The honest scope here: this post isn't announcing that the mentor endpoint got multi-provider support today. It's naming the gap precisely enough that closing it is a small, specific change instead of a someday-maybe:

- Rename `callAnthropic` to `generateCompletion`, and move the Anthropic-specific request shape (the endpoint URL, the header format, the response parsing) inside it — the function becomes the adapter, not the contract.
- The two existing call sites (`handleMentorPlan`, `handleMentorChat`) change their import, not their logic. They were never using anything Anthropic-specific except the name.
- Only then does adding a second provider — a cheaper model for the chat path, a fallback for outages — become "write a second adapter and pick between them in one place," instead of "update every place that ever mentioned the first vendor."

The GA4 Worker's `env.GA4_SERVICE_ACCOUNT_B64 ? ... : ...` branch is the template. It already exists in this repo. It just hasn't been pointed at the model call yet.

## Key Takeaways

- If a function name contains a vendor's name, that vendor is part of your contract — not just your implementation.
- The cost shows up exactly once, at the worst time: when you need to swap, fall back, or A/B test a provider and discover the change touches every call site instead of one.
- A Boring Interface has three properties: capability-named, vendor-agnostic shapes, one place to swap. All three, or it's not one yet.
- You don't need a new pattern to fix this — check whether your own codebase already has one done right for a different provider, and copy it.
- Fallback strategies (a [Degradation Ladder](/blog/the-degradation-ladder), a second model for cost or reliability) are blocked entirely until the first abstraction exists. This is the prerequisite, not an optional refinement.

---

> **CCA-F Domain 1 — Agentic Architecture & Orchestration**
>
> Designing the boundary between "what the system does" and "which vendor currently does it" is a core architecture decision under Domain 1 — not an implementation detail to defer. Provider abstraction is what makes multi-model routing, fallback, and cost optimization possible later without a rewrite.
>
> [Study Domain 1 → CCA-F Exam Prep](/skillup/ccaf)

Name the function after what it does. The vendor is a detail — the day you have to change it, you'll be glad it was never in the title.
