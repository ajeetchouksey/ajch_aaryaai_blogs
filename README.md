# ajch_aaryaai_blogs

This repository is the canonical content source for the Aarya blog. Posts are published through a controlled pipeline: draft, issue gate, content validation, approval, and final merge. The platform consumes this repository via a pinned Git SHA through the main app manifest, so the blog content is versioned and reviewable before it goes live.

## What this repo contains

- `content/blog/posts/` — blog post markdown files
- `content/blog/index.json` — canonical blog index manifest
- `.claude/agents/` — blog content agents (canonical Claude Code subagent format)
- `.claude/skills/platform-vocabulary/SKILL.md` — canonical architectural terms Content Lead and Tech Writer require every post to use consistently
- `.github/workflows/validate-content.yml` — automated validation on PR/push
- `scripts/validate-content.mjs` — local and CI schema validation

## Publishing model

This repository is not a free-form publishing sandbox. Blog content follows a review-first workflow.

1. An idea or article request starts as a GitHub Issue.
2. A product/content owner confirms the issue is in scope, has value, and should be scheduled.
3. A draft is created in a branch.
4. The content is validated against schema, markdown, and policy checks.
5. Approval is required before merge.
6. After merge to `main`, the main platform repo can promote the new SHA to the live content manifest.

This keeps blog publishing reproducible and prevents accidental drift between the repo and the live site.

## Required issue gate before publishing

Before writing or publishing any new blog post, the work must have an issue associated with it.

### Required issue checklist

- Clear title and short description of the article
- Business or content goal
- Audience and value proposition
- Target publication date or priority
- Labels such as `type:content`, `blog`, or other relevant taxonomy
- Ownership/approver identified

### Approval gate

The issue is the contract. Publishing is blocked until:

- the issue exists,
- the scope is approved,
- the content is reviewed,
- the validation workflow passes,
- and the PR is approved by the responsible maintainer.

No direct push to `main` for new blog content. No untracked "just publish this" commits.

## How the blog agents work

The repo includes specialized agents under `.claude/agents/`.

### Content Lead

Owns the workflow and orchestrates the full publishing pipeline.

Responsibilities:

- translate the user request into a content brief
- delegate drafting to the Tech Writer
- run the review gate
- coordinate with release and validation steps

### Tech Writer

Produces the blog article draft as markdown only. This is a prose-first step and does not write files to the repo directly.

Responsibilities:

- write the article in the platform voice
- suggest a slug, tags, and reading time
- produce a complete markdown draft
- keep content grounded in real engineering examples

### Release Engineer

Handles the actual repository-side publish action.

Responsibilities:

- validate the slug format
- create/update the markdown file in `content/blog/posts/`
- update `content/blog/index.json`
- keep posts ordered by newest date first

### Security/AppSec gate

Handled by **AppSec Engineer** (`.claude/agents/appsec-engineer.md`) — a hard gate Content Lead delegates to before Release Engineer writes anything. Before publication, content must pass a security and quality gate.

This usually includes:

- policy/compliance review
- no unsafe or misleading claims
- no insecure or private material
- no accidental content leaks or sensitive references
- blog/article validation against publishing rules

### QA/validation

The repo has a CI workflow at `.github/workflows/validate-content.yml` that runs the validator on content changes.

Local validation:

```bash
node scripts/validate-content.mjs content/blog/index.json content/blog/posts/*.md
```

This checks the blog manifest and content structure. It also catches invalid JSON, missing required fields, malformed slugs, and broken references.

## Publishing process

### 1) Create or find the issue

Start with a GitHub issue before writing. This is the approval gate.

### 2) Draft in a working branch

Create a branch for the article, such as:

```bash
git checkout -b blog/<slug>
```

### 3) Write the post

Create the article markdown under `content/blog/posts/` with a slug-style filename.

Example:

```text
content/blog/posts/agentic-rag-in-production.md
```

### 4) Update the manifest

Add the post entry to `content/blog/index.json` with:

- slug
- title
- excerpt
- author
- date
- tags
- category
- readingTime
- draft / featured flags

### 5) Validate locally

Run:

```bash
node scripts/validate-content.mjs content/blog/index.json content/blog/posts/*.md
```

Fix any reported issues before opening a PR.

### 6) Open a PR

Open a PR with the content change. The PR must include:

- article summary
- issue link
- target audience
- review notes or rationale

### 7) Approval and merge

The PR must be reviewed and approved before merge. Approval is intentionally not a one-click action. The content owner or designated reviewer confirms the article is aligned with the issue and publication standards.

### 8) Post-merge promotion

Once merged to `main`, the main app repo can promote the blog repo SHA into the platform manifest. That makes the live site consume the published blog content from the pinned repo version.

## Content standards

All articles should be:

- practitioner-first
- concrete and experience-based
- technically accurate
- opinionated but defensible
- easy to scan and share

### Strong blog headline standard

Use a strong position or claim instead of a generic topic label.

Good examples:

- "Agents That Can't Be Debugged Shouldn't Be Deployed"
- "You're Wasting 60% of Your Context Window"
- "The ADLC: A Software Engineering Discipline for AI Agents"

Avoid weak titles like:

- "AI agents overview"
- "Prompt engineering tips"
- "Context window article"

## Recommended author workflow

Use this flow in practice:

```text
Issue created and approved
        ↓
Draft content in branch
        ↓
Local validate
        ↓
PR opened
        ↓
Review + approval
        ↓
Merge to main
        ↓
Platform app promotes repo SHA
        ↓
Post goes live
```

## File ownership and scope

- Blog content lives in this repo only.
- The platform repo tracks the published version through a manifest pin.
- Do not bypass the issue gate.
- Do not merge unreviewed content to `main`.
- Do not update live platform content directly without the normal review and promotion process.

## Quick summary

The operating rule is simple:

> No issue, no publication. No review, no merge. No merge, no live promotion.

If you want to publish a post, start with the issue, then follow the agent + validation + approval pipeline in this order.

## Related references

- `.claude/agents/content-lead.md`
- `.claude/agents/tech-writer.md`
- `.claude/agents/release-engineer.md`
- `.claude/agents/appsec-engineer.md`
- `.claude/skills/platform-vocabulary/SKILL.md`
- `.github/workflows/validate-content.yml`
- `scripts/validate-content.mjs`
