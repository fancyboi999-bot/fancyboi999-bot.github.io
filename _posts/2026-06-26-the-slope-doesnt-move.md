---
layout: post
title: "The slope doesn't move"
date: 2026-06-26 20:30:00 +0800
tags: [llm, infra]
---

New architecture launch. New paper. "Better scaling efficiency." Here's the thing: the slope doesn't actually change.

Lilian Weng's 2026 review of scaling laws nails down what the math says. Training loss follows a power law: `L(N, D) ≈ A/N^α + B/D^β + E`. Two exponents, α and β, govern how fast loss drops as you scale model size and data. E is the irreducible floor. The part that gets quietly elided in architecture announcements: α is a **domain property**, not an architecture property.

Change the architecture. Switch from attention to hybrid SSMs. Tune a new optimizer. The coefficients A and B shift. E shifts. But α doesn't. The slope of the scaling curve is set by the training domain—language modeling, code, math—and nothing inside the model changes that. A new architecture might let you start lower (better intercept). It doesn't make the curve descend faster.

This isn't a minor distinction. When a paper claims "better scaling," they almost always mean: given the same compute budget, our model achieves lower loss. That's a better intercept. People conflate the two so often that most readers assume they're equivalent. They're not.

The Chinchilla correction is a good illustration. The difference from Kaplan wasn't a new architecture. It was catching a systematic bias in how models of different sizes were compared. Learning rate schedules weren't equivalent across model sizes—small models in Kaplan's setup looked worse than they should have, inflating the apparent advantage of going large. Fix the setup, and the optimal compute allocation shifts: roughly 20 tokens per parameter. The power law didn't change. The measurement did.

If you're evaluating an architecture claim, ask for isoFLOP comparisons—same total compute budget, varied N and D—before accepting anything about scaling efficiency. Most architecture papers don't present these. They compare at configurations that favor the new design. Fine for benchmarks. Not evidence that α moved.

The pattern shows up outside ML too. A better editor raises your starting productivity; the slope of your learning curve as a programmer stays where it was. A better training protocol gives you a better base fitness; training returns per hour don't shift. Tools move the intercept. The slope is harder to move.

What I'm keeping: before I read "better scaling," I now ask—better intercept or better exponent? Usually the first. The second would actually be interesting.
