---
layout: post
title: "The user who can't ask"
date: 2026-06-19 20:30:00 +0800
tags: [llm, tooling, workflow]
---

HuggingFace ran an experiment: give a coding agent the same task, twice. Once with the raw `transformers` Python API. Once with a purpose-built CLI that handled the boilerplate.

Same final output both times. Token cost: six times higher on the Python path.

The model didn't get confused. It wrote correct code. It just took detours — downloaded docs, ran a test, noticed the result format was different from what it expected, adjusted. Forty lines of scaffolding where one command would have done it.

Six-times overhead on a working agent is not a model problem. It's a design problem.

### Why agents detour

A human developer hitting an unfamiliar API asks a question: "Does this accept a file path or file contents?" They read the error message, they open the source, they ping the channel. The feedback loop is short.

An agent doesn't ask. It guesses, runs, reads the error, guesses again. It pays with tokens where you'd pay with seconds.

The detour tax is exactly the ambiguity budget of your API — the sum of all places where the correct usage isn't obvious from the interface alone. Humans pay it once, then remember. Every agent session starts fresh.

### The wrapper antipattern

The popular response to "agents struggle with my library" is to add an MCP server, or a LangChain wrapper, or a custom tool. Add a second surface; leave the first one alone.

This is backwards. You've now given the agent *two* APIs to navigate. And the second one often inherits the ambiguity of the first in thinner disguise.

The more useful question: why does the agent need a wrapper at all? Usually because the core API has some ambiguity — an overloaded function, a parameter name that doesn't describe its type, a return value where the failure mode looks like the success mode. Fix that, and the agent uses the original API correctly. Don't fix it, and the wrapper becomes a permanent translation layer for a confusion that never gets addressed.

### Designing for the non-interactive case

The constraint is simple: write your docs as if the user will never send you a question.

That constraint is brutal. It forces you to name things for their actual semantics, not their implementation. It forces return values to be self-describing. It forces examples to cover the cases where people *want* to ask a clarifying question — because those are exactly the cases your agent will misuse first.

This isn't a new principle. Any async interface already has this requirement. You write a PR description assuming the reviewer won't be in the room. You write a job posting assuming the candidate won't know what "senior" means to this team. You specify the interface upfront, because you can't answer questions at runtime.

Agents just make the cost of ambiguity visible, because they pay it every time, in tokens, measurably.

### What I'm keeping

If an agent takes multiple attempts to use a function correctly, treat it as a documentation bug, not a model limitation. The function works. The function's interface is teaching the wrong expectation. Fix the interface.

The agent who can't ask is doing you a favor: it's telling you exactly where your API assumes knowledge it doesn't provide.
