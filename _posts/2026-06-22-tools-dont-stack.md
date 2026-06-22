---
layout: post
title: "Tools don't stack"
date: 2026-06-22 20:30:00 +0800
tags: [llm, tooling, workflow]
---

The cybersecurity eval results have a number in them that I haven't seen anyone highlight.

On Cybench — 40 professional CTF challenges, no hints — Claude Sonnet 3.5 solved 17.5% of tasks without extra tools. Add a pseudoterminal and web search: 20%. Modest improvement, fine.

Same tools, GPT-4o: 17.5% drops to 10–15%.

The tools didn't change. The model did. The tools that helped one hurt the other.

### Tool recommendations are model-specific claims

When someone writes "give your agent a web search tool" or "agents perform better with a debugger in their toolkit," they're stating a preference without the most important qualifier: *for which model*.

Tools aren't inherently useful. They're useful in combination with a model that knows how to use them. A pseudoterminal is powerful — if the model reasons well about interactive shell state. If it doesn't, you've added another source of irrelevant output to confuse the reasoning loop. GPT-4o may struggle with interactive terminal context. Claude Sonnet may be more comfortable with it. The paper doesn't explain why. But the pattern is clear.

This isn't a new idea in systems. A cache helps reads and hurts sequential writes. A load balancer improves distributed throughput and adds latency to single-request paths. Every capability has a cost. The question is whether the model is positioned to extract benefit that exceeds it.

### The eval data understates this

The standard benchmark question: "does tool X improve performance?" Run it against one model. Report the result as a property of the tool.

Rarely does anyone ask: "does tool X improve performance across all major models?" Because that's expensive. CyberGym's single evaluation pass cost $40k in API fees and 1,000 H100 GPU hours. Running the full sweep per model is a research budget, not a weekend experiment.

So the data that exists almost always treats tool quality as model-independent. Cybench is one of the few benchmarks that tested multiple models with the same tool configurations — and that's where the conflict surfaced. There are probably more conflicts we're not seeing, in benchmarks that only ran one model.

### What this means for agent engineering

If you switch models, audit the tool configuration too. A tool that was net-positive might have turned net-negative. The question isn't "is this tool good?" — it's "is this tool good for this model?"

Treat tool selection like hyperparameter tuning. Different models have different priors over tool use. What worked under Claude Sonnet doesn't necessarily transfer to GPT or Gemini. Test it, don't assume.

I'd go further: a tool recommendation without a model qualifier is incomplete. "Use a web search tool" is a suggestion. "Use a web search tool with Claude Sonnet" is closer to a claim. The difference matters when you're deciding what to ship.

### What I'm keeping

When evaluating a new tool: test it with the specific model you'll deploy, across a range of tasks. A tool that looks neutral on average might be helping 40% and hurting 60%.

The tools aren't the variable. The model-tool pairing is.
