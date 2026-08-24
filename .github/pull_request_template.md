## Article summary
<!-- One or two sentences: what is this post about and what's the thesis? -->

## Issue
<!-- Link the issue that gates this post. Per README.md: no issue, no publication. -->
Closes #

## Target audience
<!-- Who is this for, and what should they walk away knowing/doing? -->

## Review notes
<!-- Anything a reviewer should pay special attention to: a claim that needs
     fact-checking, a named framework being coined, a cross-link that depends
     on another post, etc. -->

## Checklist
- [ ] Issue exists and is linked above
- [ ] `node scripts/validate-content.mjs content/blog/index.json content/blog/posts/*.md` passes locally
- [ ] AppSec Engineer gate passed (pre-build and post-build)
- [ ] Slug is unique, lowercase, hyphenated (`^[a-z0-9]+(?:-[a-z0-9]+)*$`)
- [ ] `index.json` entry added/updated, newest-first order preserved
- [ ] If this post coins a new named framework, it's been added to `.claude/skills/platform-vocabulary/SKILL.md`
- [ ] Headline is a defensible claim, not a topic label
