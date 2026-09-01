# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is a **content-only repository** — the canonical source for the "Aarya — My AI Learning Hub" blog. There is no application code, no `package.json`, no build step. It exists purely so blog content can be versioned and reviewed independently of the main platform app (`ajch_platform`), which consumes this repo via a pinned Git SHA in its content manifest.

Do not add application/build tooling here unless the task explicitly calls for it — this repo's job is markdown + a JSON manifest + a validator script.

## Repository layout

- `content/blog/posts/*.md` — blog post markdown files (frontmatter + body)
- `content/blog/index.json` — canonical blog index manifest (one entry per post)
- `content/blog/images/` — post images, referenced from `index.json`/posts as `/content/blog/images/...`
- `scripts/validate-content.mjs` — the only script in the repo; validates content schema
- `.claude/agents/` — subagents implementing the publishing pipeline (see below)
- `.claude/skills/platform-vocabulary/SKILL.md` — canonical named frameworks/terms every post must reuse verbatim
- `.github/workflows/validate-content.yml` — CI validation on PR/push
- `.github/CODEOWNERS` — `.github/`, `scripts/`, `.claude/agents/` require owner review; `content/` intentionally has none (contributor-friendly)

## Commands

There is no build, lint, or test suite — the only command in this repo is the content validator:

```bash
node scripts/validate-content.mjs content/blog/index.json content/blog/posts/*.md
```

Run this before opening any PR that touches blog content. It is also the CI gate (`.github/workflows/validate-content.yml`, Node 22, runs on every PR — no path filter, since it's a required status check on `main` — and on push to `main` when `content/blog/**` or the script itself changes).

The validator (`scripts/validate-content.mjs`) is a copy synced verbatim across `ajch_platform` and each standalone vertical repo, so it auto-detects layout (checks for a `public/` dir) rather than hardcoding paths — don't hand-fork it per repo. In this repo it checks:
- `content/blog/index.json` — every post entry has `slug`, `title`, `author`, `date` (`YYYY-MM-DD`), `tags`
- `*.md` files — non-empty, and frontmatter (`---`) is properly closed if present
- Any JSON file — must at minimum parse

There's no single-file/single-test invocation beyond passing that one file's path as an argument to the same script.

## Publishing model — the hard rule

**No issue, no publication. No review, no merge. No merge, no live promotion.**

Blog content follows a mandatory review-first workflow — this is not a free-form sandbox:

1. A GitHub Issue must exist and be approved before any drafting starts (title, business/content goal, audience, target date, labels like `type:content`/`blog`, owner).
2. Draft in a branch, e.g. `git checkout -b blog/<slug>`.
3. Write the post under `content/blog/posts/{slug}.md`.
4. Add/update the matching entry in `content/blog/index.json`, keeping posts ordered **newest-first by date**.
5. Run the validator locally (see Commands).
6. Open a PR linking the issue, with summary/audience/review notes (see `.github/pull_request_template.md` — it also has the reviewer checklist, including "if this post coins a new named framework, add it to platform-vocabulary/SKILL.md").
7. PR requires human approval before merge — no direct push to `main` for new blog content.
8. After merge, the platform repo (`ajch_platform`) promotes this repo's new SHA into its live content manifest — this repo does not "go live" on its own.

## The agent pipeline (`.claude/agents/`)

Four subagents implement the workflow above as a strict pipeline. Each has a narrow write scope — respect it even when working manually:

```
Content Lead → Tech Writer → AppSec Engineer (hard gate) → Release Engineer
```

- **content-lead** — orchestrator only, never writes files. Turns a request into a Tech Writer brief, runs the security gate, then briefs Release Engineer.
- **tech-writer** — prose-only, no file I/O. Returns markdown + suggested slug/tags/reading time as a string. Model: inherit.
- **appsec-engineer** — hard gate, read-only, runs pre-build and post-build. Returns `PASS ✓` or `BLOCK ✗ <reason>` only — never a partial pass. Checks path traversal, slug format, secrets, schema conformance, content policy, and (for this repo) that `content/blog/index.json` entries match `{ slug, title, excerpt, author, date, tags[], category, readingTime, featured, draft }`.
- **release-engineer** — the only agent that writes to disk, and only inside `content/blog/`. Model: `claude-haiku-4-5-20251001` (deliberately cheap — mechanical file writes, no creative judgment). Validates slug against `^[a-z0-9]+(?:-[a-z0-9]+)*$`, checks for slug collisions, computes reading time as `Math.ceil(wordCount / 200)`, and inserts into `index.json` at the correct date-sorted position (never appends blindly).

When authoring a post manually rather than through the agents, follow the same contract: slug regex, newest-first manifest ordering, and matching frontmatter/manifest fields. Note the manifest schema documented in `release-engineer.md` doesn't include `authorGitHub` or `image`, but real posts (e.g. `microsoft-foundry-two-boundary-model`) use both — treat the actual `index.json` entries as the source of truth over the agent doc when they diverge.

## Content standards

- **Strong-Claim headlines**: titles must be a defensible position, not a topic label (e.g. "You're Wasting 60% of Your Context Window", not "Context Window Management"). Test: if a reader can say "well, not always..." it's too hedged; if they say "that's wrong, and here's why," it's right.
- **Named-Framework Technique**: every post should coin or reinforce one 2–5 word, title-case named concept, defined in one sentence on first use and carried through the post's tags (e.g. "The Degradation Ladder", "The Boring Interface", "Domain Boundary").
- **Platform Vocabulary**: `.claude/skills/platform-vocabulary/SKILL.md` is the canonical glossary (The Context Budget Rule, The 4-Layer Agent Stack, The Retry Pattern, Domain Boundary, Lazy Tool Loading, Recency-Weighted Pruning, plus v3.x platform types like `ContentType`, `ExamPalette`, `RichScenario`). Never invent a synonym for a term defined there — reuse the exact name. Coining a new framework in a post means adding it to this file in the same PR.
- **Human Sense**: hook openings and close sections with a proverb, punch line, or analogy where it fits naturally — but only when it lands; a forced idiom is worse than none, and personality never substitutes for precision or overrides Anthropic's usage-policy compliance.
- Categories in use: `AI Architecture`, `DevOps`, `Cloud`, `Opinions`, `Azure`.
