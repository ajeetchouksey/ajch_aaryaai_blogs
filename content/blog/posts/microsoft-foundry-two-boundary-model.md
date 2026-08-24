---
title: Microsoft Foundry Has Two Boundaries. Most Teams Only Use One.
date: 2026-08-24
tags: [microsoft-foundry, azure-ai-foundry, ai-architecture, named-framework, resource-governance]
category: AI Architecture
---

![Microsoft Foundry's Two-Boundary Model — the account as governance boundary, the project as isolation boundary](/content/blog/images/foundry_two_boundary_model.png)

---

## Introduction

A rose by any other name still needs the right soil, and Microsoft's AI platform has now had three names in three years without changing what actually grows in it. If you provisioned it in 2023 you called it Azure AI Studio. In 2024 and most of 2025 you called it Azure AI Foundry. As of this year, Microsoft's own documentation calls it **Microsoft Foundry** — but the portal is still at `ai.azure.com`, your issue tracker probably still says "Azure AI Foundry," and nobody's Terraform state file got the memo. Both names are correct right now. This post uses them interchangeably and tells you why up front, so the naming churn doesn't distract from the part that actually matters: what you're building underneath it.

The failure mode isn't confusion about the name. It's a resource design mistake baked in at creation time, one that most teams don't discover until the second team wants access to the platform and finds there's no clean way to give it to them. Foundry's architecture has exactly two boundaries that matter — a governance boundary and an isolation boundary — and one of the decisions that separates them is locked the moment you provision the resource. Get it wrong on day one, and you don't retrofit it. You rebuild.

---

## What Foundry Actually Is

Strip away the branding and Foundry is a unified platform-as-a-service for building and deploying generative AI applications and agents — one Azure resource that replaces what used to be a hub, a separate Azure OpenAI resource, and a scattering of Azure AI Services resources stitched together by hand. Inside it you get a model catalog with over 10,000 models from Microsoft, Azure OpenAI, Anthropic, Meta, Mistral, Cohere, Hugging Face, and others — with roughly 50 new models added every month; **Foundry Agent Service** for building declarative prompt agents or deploying hosted agents that run your own code; fine-tuning; evaluation tooling that scores agent and model output against grading criteria; and responsible AI controls — content filtering, guardrails, and risk detection wired directly into the inference pipeline rather than bolted on as a separate review step.

That consolidation is the actual point of the rebrand, not the name change itself. Previously you'd provision a hub, then a separate Azure OpenAI resource, then Azure AI Search, then wire RBAC and networking across all three by hand — different SDKs, different endpoints, different API version schemes. Now it's one Azure resource provider namespace (`Microsoft.CognitiveServices`), one set of RBAC actions, one place to configure network isolation. **The unification is real. The question is whether your resource layout takes advantage of it.**

---

## The Two-Boundary Model

Here's the framework worth citing in your next design review: Foundry's architecture is a **Two-Boundary Model** — one boundary for governance, one for isolation, and a set of connected services that sit outside both and answer to nobody but themselves.

| Boundary | What it is | What it controls |
|---|---|---|
| **Governance boundary** | The top-level Foundry resource (`Microsoft.CognitiveServices/accounts`, kind `AIServices`) | Networking, security, model deployments, control-plane RBAC actions |
| **Isolation boundary** | The Foundry project — a subresource (`Microsoft.CognitiveServices/accounts/projects`) | Agents, evaluations, files, data-plane RBAC actions |
| **Connected resources** | Azure Storage, Key Vault, Azure AI Search — referenced, not owned | Their own governance, independent of Foundry entirely |

The account is where IT and security teams live: model deployment configuration, private networking, encryption keys, and control-plane RBAC actions like "who can create a new project." The project is where developers live: building agents, running evaluations, uploading files, and the data-plane RBAC actions scoped to just that work. This is a structural [Domain Boundary](https://learn.microsoft.com/azure/foundry/concepts/architecture) in the exact sense this platform uses that term elsewhere — an agent built inside one project has no default visibility into another project's connections or model deployments, even though both sit under the same governed account.

The third row is the one that trips up compliance reviews. Storage accounts, Key Vault, and Azure AI Search are Azure resources Foundry *references* through connections — they are not part of the Foundry resource's governance boundary. You manage their networking, access policies, and compliance posture entirely separately. A security review that audits "the Foundry resource" and stops there has audited one boundary and skipped two.

**The account decides what's possible. The project decides who gets to do it. Confusing the two is how you end up with either no isolation or no governance — never both at once.**

---

## The Anti-Pattern: One Resource, One Project, Forever

The mistake isn't choosing Foundry over a standalone Azure OpenAI resource — Microsoft explicitly ships a supported upgrade path that preserves your endpoint, API keys, and existing state, so that decision isn't the irreversible one people assume it is. The mistake most teams actually make is smaller and easier to miss: they provision one Foundry resource, get one default project, ship their first agent, and never revisit the topology. Every subsequent team that wants access either gets thrown into the same project — no isolation, shared everything — or gets a brand-new Foundry resource of its own, duplicating network configuration, RBAC setup, and model deployments that could have been shared.

Both outcomes throw away the thing Foundry's architecture was built to give you: **one governed resource, many isolated projects**, sharing deployments and connections while keeping each team's agents, evaluations, and files in their own lane. The docs are explicit that this is the intended shape for multi-team access — and just as explicit that a single resource with a single project is the right call *only* for solo exploration, not as a default you back into by accident.

Here's the part that makes this a real architecture decision and not just a preference: whether your Foundry resource can even host more than one project is set at creation time, and it cannot be changed afterward.

```bash
# Create the governance boundary — this flag is permanent.
# Omit it and this resource can never host more than its
# single default project. There is no "upgrade later" path
# for this specific setting.
az cognitiveservices account create \
    --name my-foundry-resource \
    --resource-group my-foundry-rg \
    --kind AIServices \
    --sku S0 \
    --location eastus \
    --custom-domain my-foundry-resource \
    --allow-project-management

# Create an isolation boundary inside it — cheap, reversible,
# and the thing you actually want to be creating per team.
az cognitiveservices account project create \
    --name my-foundry-resource \
    --resource-group my-foundry-rg \
    --project-name team-fraud-detection \
    --location eastus
```

Projects are cheap and disposable — create one per team, delete it when the team's done. The `--allow-project-management` flag is not. Skip it because you're "just testing," and the resource you're testing on can never grow into the multi-team platform you'll want in month three — you'll be standing up a second resource, migrating connections, and re-doing RBAC, which is exactly the retrofit tax the Two-Boundary Model was supposed to let you avoid.

---

## Where This Fits a Real Deployment

The baseline end-to-end chat reference architecture in the Azure Architecture Center makes this hierarchy concrete rather than abstract: a Foundry resource sits inside its own subnet, with an *account* and a *project* as distinct nodes in the diagram, connected by a managed identity to Foundry Agent Service, which in turn reaches Azure OpenAI model deployments. Private endpoints, an application gateway with a web application firewall, Microsoft Entra ID for authentication, and Azure Monitor for observability all sit around it — the Two-Boundary Model isn't a simplification for this article, it's how Microsoft's own production reference architecture is drawn.

It's also worth noting where Foundry converges with a principle this platform has already named: the newer unified project client authenticates once against a single project endpoint and talks to any of the 10,000+ catalog models behind it — the same shape as [The Boring Interface](/blog/the-boring-interface), where the caller's contract stays stable regardless of which vendor answers behind it. And the multi-agent, privacy-first patterns in [The AI Architecture Blueprint](/blog/ai-architecture-blueprint-multi-agent-privacy-first) — the Tool Gateway as a governed action surface, the Agent Contract's explicit data-domain scoping — map directly onto what a Foundry project boundary is structurally trying to enforce. If you're building the Blueprint's stack, Foundry's account/project split is a natural place to host it, not a competing architecture.

---

## Key Takeaways

- **Microsoft Foundry and Azure AI Foundry are the same platform mid-rename.** The portal is still `ai.azure.com`; docs now live under `learn.microsoft.com/azure/foundry/`. Use either name — just don't let the churn distract from the resource model underneath.
- **The Two-Boundary Model:** the account is the governance boundary (networking, security, model deployments, control-plane RBAC); the project is the isolation boundary (agents, evaluations, files, data-plane RBAC).
- **Connected resources — Storage, Key Vault, Azure AI Search — are governed independently.** Auditing the Foundry resource alone misses them entirely.
- **The Azure OpenAI → Foundry migration path is real and low-risk** — endpoint, API keys, and RBAC carry over. The irreversible decision is one level deeper: `--allow-project-management`, set once, at creation.
- **Provision for multiple projects even if you only need one today.** Projects are cheap to create and delete; a resource that can't host more than one is not.

> A resource you can reshape costs a migration. A flag you can't reshape costs a rebuild. Know which one you're setting before you hit enter.
