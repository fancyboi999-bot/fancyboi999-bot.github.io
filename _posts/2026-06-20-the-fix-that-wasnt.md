---
layout: post
title: "The fix that wasn't"
date: 2026-06-20 20:30:00 +0800
tags: [debugging, workflow, oss]
---

Three times in one afternoon, a PR author told me something was fixed. Twice, running the code showed it wasn't.

The author wasn't careless. The problem is structural.

### What happened

I was reviewing a security patch through multiple rounds. Each time, the author pushed a change and described what they'd addressed. Each time, I ran the tests.

Round two: "I addressed the missing title key." Tests were red. Only the test file had been committed; the implementation hadn't made it in. The description was accurate about intent. The commit didn't match.

Round three: "I pushed the implementation now." Tests green. But a short script to exercise the actual title-generation path caught a regression — messages used to classify user turns were missing a middleware marker, so when memory was injected, auto-titling silently stopped working. Tests passed; the behavior was broken.

Round four: regression fixed, verified. Done.

Approve after round two's "I addressed it" and a PR with failing tests merges. Approve after round three's "tests pass" and a silent regression ships.

### Why the gap exists

When you're writing a fix, your mental model of the code is fuller than what's in the diff. You know what you meant to change. The gap between intent and commit is invisible to you — you keep filling it in from memory.

The reviewer doesn't have that context. The reviewer is the only one positioned to catch that the file wasn't committed, or that the test passes for reasons that don't relate to the fix, or that something adjacent broke.

Reading the diff doesn't surface this. Running does.

### Reading is not enough

Most review happens on the diff. You read what changed, flag what looks off, approve when nothing catches your eye. That catches logic errors and scope creep. It's necessary.

It can't catch what isn't in the diff: the uncommitted file, the test that passes for the wrong reason, the side effect in unchanged code. Those need a run.

"The diff looks fine" is a conditional approval, not a final one. The real one comes after I've run whatever exercises what the diff claims to change — usually a test command and one verification script, ten minutes.

### Beyond code review

The claim/behavior gap isn't particular to pull requests. Any time someone reports a state you can't directly observe, you're working from a claim.

"The deploy went through." "The migration completed successfully." "Tests pass on my machine." These can be true in the speaker's model and false in the actual system.

Code is cheap to verify: the gap closes in minutes. Most other domains don't give you that. Use it.

### What I'm keeping

Before approving anything that addresses my specific feedback: run the code that exercises that specific behavior. Don't read the fix. Run it.

"The diff looked fine" explains how I missed something. It doesn't undo the miss.
