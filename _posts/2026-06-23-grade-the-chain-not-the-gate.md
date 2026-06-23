---
layout: post
title: "Grade the chain, not the gate"
date: 2026-06-23 20:30:00 +0800
tags: [llm, tooling, debugging]
---

The default agent benchmark is a gate: did the agent capture the flag? Did it complete the task? Binary.

The problem is that binary scoring hides the failure. An agent that gets 90% of the way through a task and an agent that misreads the prompt in step one both score zero. You can't improve what you can't distinguish.

Cybersecurity eval researchers hit this first, because their tasks are long and the failure modes are genuinely different. The fix they landed on: decompose the task into an attack chain, and score each level.

```
Level 1: Find the vulnerability in the code.
Level 2: Reproduce it with a PoC (trigger a crash).
Level 3: Execute arbitrary code through it.
Level 4: Achieve the attacker's goal.
```

A failed run now tells you something. An agent stuck at Level 1 needs better code exploration. One that reaches Level 3 but stalls at Level 4 needs different reasoning. The fix changes because the diagnosis is different.

This is not a cybersecurity insight. It's a general principle: any long-chain task should be graded by where the chain broke, not whether the chain completed.

You already know this at the infrastructure level. CI pipelines don't just say "build failed"—they tell you whether it was compilation, lint, unit tests, integration, or deploy. But when evaluating an AI agent on a multi-step task, the reflex is to collapse it back to a binary gate. Easy to instrument. Feels objective.

It is not more objective. It is less informative.

Pass/fail agent evals are intellectual laziness dressed as rigor. They feel decisive—fail is fail—but the verdict tells you nothing about what to change. Partial credit forces you to decompose the task precisely enough to define what "half done" means. That decomposition is the actual work, and skipping it is where most agent evaluations go wrong.

One data point worth sitting with: in current security benchmarks, even the strongest models succeed on only 10–22% of real CVE exploitation tasks. The failure mode breakdown shows that 67–80% of zero-day failures happen at step one, exploration. Not at the hard parts. At "find the thing to look at." Binary scoring would just log all of those as "failed," and you'd spend effort improving the wrong layer.

When a subagent fails on something complex, the first question isn't "why did it fail?" It's "how far did it get?" Different question. Only one leads somewhere useful.
