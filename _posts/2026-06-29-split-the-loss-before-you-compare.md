---
layout: post
title: "Split the loss before you compare"
date: 2026-06-29 20:30:00 +0800
tags: [llm, debugging, tooling]
---

AllenAI trained two 7B models on the same data with the same schedule — the only difference was architecture. One was a pure transformer (Olmo 3). One was a hybrid: transformer layers alternating with recurrent layers (Olmo Hybrid). Then they looked at the final loss.

Almost identical. The hybrid has a small edge, but not much.

If you stopped there, you'd conclude: recurrent layers don't buy you much at this scale. That conclusion is wrong.

The issue is what "average loss" is averaging over. When you compute perplexity over a text corpus, you're averaging across every token: content words, function words, pronouns, punctuation, and tokens that are verbatim repetitions of something earlier in the same passage. These token categories have completely different prediction difficulty profiles, and architectures have different strengths against them.

When you split the loss by token type, a different picture appears:

On **content words** — nouns, verbs, adjectives, anything semantically dense — the hybrid wins by a noticeable margin. Recurrent layers maintain a rolling compressed state over the sequence. For tokens whose prediction depends on tracking something that's been evolving (who the subject is, what state a variable is in, what the conversation has established), that state is useful. The hybrid gets this right more often.

On **verbatim repetition** — tokens that appear again because the text is literally reusing a phrase, variable name, or snippet — the hybrid's advantage collapses to near zero. Transformer attention can reach back to any position and copy exactly. A recurrent layer's compressed state is lossy; it can't do verbatim recall as cleanly. The transformer catches up here.

The two effects roughly cancel in the aggregate. That's why the headline number is boring. The architecture isn't boring — it's trading capabilities.

This is a version of Simpson's paradox. If you average over a population where your variable has opposite effects in two subgroups, the aggregate can look flat even when both effects are real and large. The aggregate isn't lying, but it's answering a question you didn't ask.

The practical consequence: if you're evaluating architectures and both have similar perplexity, look at where each one loses. The pattern of failures often tells you more than the average. A hybrid that struggles on verbatim recall is a different failure mode than a transformer that struggles on coreference. For your downstream task, one of those failure modes matters more.

The AllenAI work found that filtered token loss — stratified by POS tag and n-gram repetition overlap — shows this divergence early, at 1B parameters, well before the overall loss curves separate. If you were doing architecture search, that's a signal you could use much earlier in a training run rather than waiting until full scale.

The broader point isn't about hybrid architectures specifically. It's that when two systems are competing on a metric, and you know the metric is an average, ask what it's averaging. If the underlying distribution is heterogeneous in a way that's relevant to the comparison, split it. The signal might be there, just cancelled out.
