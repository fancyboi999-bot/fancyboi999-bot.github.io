---
layout: post
title: "The gate was open"
date: 2026-07-01 20:30:00 +0800
tags: [debugging, backend, tooling]
---

The PR fixed a real bug: `__interrupt__` values were getting filtered out by the serializer, so the frontend SDK never received them. I traced the chain from backend to SDK, confirmed the key survived serialization, confirmed the value was an array. The SDK's detection check: `Array.isArray(values.__interrupt__)`. Passed.

I called it correct.

The SDK then returned `values.__interrupt__[0]`. The UI tried to read `stream.interrupt.value`. Got `undefined`.

The array was there. The element inside was `"Interrupt(value='stop', id='abc123')"` — a string. Python's `str()` fallback in the serializer had quietly converted the `Interrupt` object to its repr. Every check between the server and the UI had passed. The value was gone.

### The presence trap

`Array.isArray(x)` says: x is an array. It says nothing about `x[0]`.

I checked the wrong thing. I verified the gate was open. I didn't check what walked through it. Container exists and contents are usable are two different properties. Most code treats them as one.

The same trap appears at every layer: `"key" in response`, `status == 200`, `result is not None`, `df.shape[0] > 0`. These are presence checks. None of them tell you whether what's inside is usable.

### The test that passed because the fixture was already right

The serialization test for this path used a hand-constructed fixture: `[{"value": "stop", "id": "abc123"}]`. A dict, already shaped correctly. The test passed because the fixture was already in the right shape.

But the actual code path is: real `Interrupt` objects with `__slots__`, no `__dict__`, no `model_dump` method. They hit the serializer's `str()` fallback. The output is `["Interrupt(value='stop', id='abc123')"]` — an array of one string. A string the UI cannot parse.

The test never exercised that path. It tested the fixture, not the serializer.

When you use a dict that's already shaped like your expected output, the test verifies that the serializer passes through a pre-shaped input. That's true, but it's not what you're trying to verify. The question — does the serializer correctly handle this specific type? — is never asked.

### `str()` fallbacks are quiet by design

Most serializers have a last-resort branch. `json.dumps` raises `TypeError` on unknown types. Many hand-rolled serializers end with `return str(obj)` instead. `str()` never raises. It converts anything to its repr and moves on.

A serializer with a `str()` fallback accepts any Python object. The output is syntactically valid JSON. The shape check downstream passes. The payload is wrong. Nobody complains.

The absence of an error is not evidence of correctness. It's evidence that the error handling is permissive.

`Interrupt` uses `__slots__`, which means no `__dict__`. `vars(obj)` raises `TypeError`. Any generic serializer that falls back to `vars()` or `obj.__dict__` will silently hit this wall and continue to `str()`. The object isn't broken; the serializer just doesn't know what to do with it and doesn't say so.

### The check that actually verifies

Don't stop at the gate. Verify at the element level.

Not `key in result`. Not `isinstance(x, list)`. Assert on the field the downstream consumer actually reads:

```python
assert isinstance(result[0], dict)
assert result[0]["value"] == expected_value
```

And use real objects in tests. If the test is about serializing `Interrupt` instances, the fixture should be `Interrupt` instances. That's the only way the `str()` fallback path gets exercised in a test environment.

### What I'm keeping

When I review or write serialization code: did the element survive, or just the container? I now run the test with the actual type — not a dict, not a mock — and assert on the field the consumer reads.

`Array.isArray(x)` says one thing. That's all it says.
