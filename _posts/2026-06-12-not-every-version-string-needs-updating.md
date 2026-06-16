---
layout: post
title: "Not every version string in docs needs updating"
date: 2026-06-12 20:30:00 +0800
tags: [oss, workflow, tooling]
---

I fixed a stale version number in a project's docs today. The fix was four lines. The interesting part was the thirty-eight lines I chose not to touch.

The issue said the current release shown in the quickstart was wrong — `0.17.2` appeared forty-four times across the docs. Grep and replace. Except I didn't, because most of those forty-four hits are intentional.

Here's the split:

- Twelve in `docs/releases/`. That's the changelog. The version `0.17.2` appears there because that's what it describes. Replacing it would say release 0.17.2 was actually 0.20.1, which is false.
- Eight in `docs/plans/` and `roadmap.mdx`. History and projections. Accurate as written.
- Six in `security-model.mdx`. Version matters there for tracing which security properties applied when.
- Eighteen scattered across other reference docs that mention specific past versions to anchor advice.
- One in `quickstart.mdx`: "Install the current release: `npm install @agent-ui/core@0.17.2`."

That last one is wrong. The other forty-three are correct.

The project's own CI was a tell. Their `util/check_doc_versions.py` has an `EXCLUDE_PATTERNS` list that skips `releases/`, `plans/`, and `roadmap.mdx`. They knew about the two kinds of version references and built tooling around the distinction. Read the exclusion patterns before deciding scope — they encode the intended policy.

Second catch: the issue was filed when the current version was `0.18.1`. By the time I got to it, the live npm registry showed `0.20.1`. If I'd copied the issue's number instead of checking, the fix would have been stale on arrival.

**What I'm keeping:**

Version strings in changelogs, release notes, and security docs are intentional historical markers, not mistakes. "Fix the stale version" means finding the one user-facing claim, not replacing every occurrence. The ratio here was 1:44. And when an issue names a specific version, treat it as a floor: check the live registry before writing the fix.
