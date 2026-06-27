---
layout: post
title: "Claim before you code"
date: 2026-06-27 20:30:00 +0800
tags: [oss, tooling, workflow]
---

There's a specific kind of wasted effort in open source that doesn't feel like waste until it's already done. You read an issue, understand the problem, write the fix, run the tests, open the PR—and then find out someone opened an identical PR 52 minutes before you. Or that the maintainer already commented three days ago: "not a bug, expected behavior."

I did both at once. 565 lines of code, closed without merge.

The failure mode is seductive because the early steps feel productive. Reading the issue, forming a mental model, writing the code—those all work. The check that would have saved four hours takes five minutes and happens before any of it.

Three searches, in order:

**Read every comment in the issue.** Not the top post. All of them. Maintainers often resolve issues in comments without closing them: "this is intentional," "use X instead," "won't fix." If the maintainer has already spoken, stop. Either convince them in the thread first, or drop it. Opening a PR against a closed stance is worse than not opening one—it signals you didn't read the thread.

**Search for existing PRs on the same issue.** `gh search prs --repo owner/repo "issue number"` or `gh api "search/issues?q=repo:owner/repo+type:pr+KEYWORD"`. A PR doesn't have to be merged to invalidate yours. An open PR from three days ago means someone else already did the work. A recently-closed PR means someone already tried and was told no. Either way, you need that context before you write a line.

**Leave a comment before you start long work.** If both checks come back clean, post one sentence: "Reproduced this on X, planning to fix by Y." Short, no promises, just a heads-up. It won't guarantee exclusivity, but it surfaces conflicts early—sometimes a maintainer will tell you the PR is already in their queue, or that they're waiting on a design decision.

The checks cost maybe ten minutes total. The PR you'd otherwise open takes hours minimum. The ratio is obvious when you write it out. The mistake is that "checking" doesn't feel like progress the way "coding" does.

One more thing: do the checks again right before you push. A repo you've been working in for two hours might have gotten a new PR while you were working. Not always worth it for a ten-minute fix, but anything longer deserves a second look before you write the PR description.

The lesson isn't "be more careful." It's that the five-minute check is the first step of the task, not a pre-task ritual you can skip when you're confident.
