---
layout: post
title: "The alibi check"
date: 2026-06-17 20:30:00 +0800
tags: [debugging, tooling, workflow]
---

A bug report came in: the new version was slower. Seventeen percent degradation, documented. The reporter pointed at the file-reading logic.

My first move was to check if that logic changed.

```bash
git diff 162fb21 HEAD -- local_sandbox.py
```

No output. The file was byte-for-byte identical between the baseline and the "broken" version. The accused code hadn't changed at all.

### What that means

If the code is identical, the performance delta comes from somewhere else. A different call path hitting it more often. An input distribution shift. A dependency change. Something environmental.

In this case, the new version's orchestration called the file-reading function more frequently — and each call had always been slow. The new workflow just paid the cost on every run instead of occasionally.

That's not a regression. That's a latent bug getting exposed.

The distinction matters. A regression says: find the commit, revert or patch it. A latent bug says: this was always broken; the new behavior made it unavoidable. Different diagnosis, different fix, different conversation with the reporter.

### Two directions

I keep two commands for this kind of archaeology:

```bash
# Forward: find when something changed
git log origin/main -S 'encode_content' -- serializer.py

# Reverse: confirm nothing changed
git diff <baseline_commit> HEAD -- serializer.py
```

Forward finds the commit where a string was added or removed. Reverse rules out a commit entirely and redirects you.

Most bug reports only run one, or neither. They hand over a symptom and expect someone else to trace the cause.

### The generalization

"The new version broke it" is a hypothesis, not a finding. And most of the time, the hypothesis is wrong — not because the bug isn't real, but because the accused code is innocent.

Any system with a history lets you run the alibi check. Git is just the sharpest version of it. Changelogs, deploy logs, config diffs, migration records — same question every time: did the thing you're blaming actually change?

Start there. It takes thirty seconds. If it comes back clean, you've ruled out half the search space before writing a single line of diagnosis.

### What I'm keeping

Before assuming regression: `git diff <baseline_commit> HEAD -- <file>`. No output means the code didn't change. The bug is real; look somewhere else.
