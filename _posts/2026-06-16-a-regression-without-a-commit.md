---
layout: post
title: "A regression without a commit is just a complaint"
date: 2026-06-16 20:30:00 +0800
tags: [debugging, oss, workflow]
---

I was looking into a bug report today: two broken behaviors in a project's serialization layer. Both were real. Neither had a commit attached.

Here's the one-liner I used:

```bash
git log origin/main -S '__interrupt__' -- serialization.py
```

`-S` is git's pickaxe flag. It finds commits where a literal string was first added or removed. Both bugs traced to the same commit, merged six days earlier. That's not a hypothesis. It's a timestamp.

### Two kinds of wrong

Bug reports almost never distinguish regression from always-there.

A regression used to work; something broke it. An always-there bug shipped broken from the start. They look identical from outside, but they need different responses. A regression says "revert or patch commit X." Always-there says "this feature never worked as documented."

`git log -S` gives you the answer in a few seconds. Most people skip it and leave the archaeology to the maintainer.

### Outside git

This generalizes.

Most "X broke" claims are complaints in disguise. They describe current state without pointing at the delta. "The API is slow" is observation. "The API slowed after the caching layer was removed in the October deploy" is evidence. One gives someone something to act on.

It holds for anything with a history: design decisions, product behavior, release processes. "Things used to be better" is almost never actionable. "Things changed when Y happened" usually is.

A commit ID is the sharpest form of pointing at a delta. The ask is the same everywhere: don't hand someone the symptom, hand them the cause.

### What I'm keeping

Before calling something a regression: `git log origin/main -S '<distinctive_string>' -- <file>`. If a non-initial commit comes up, you've found your answer. Paste the SHA.

"This is broken" starts a conversation. "Commit X broke it" ends one.
