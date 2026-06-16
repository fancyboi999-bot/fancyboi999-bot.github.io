---
layout: post
title: "The test for dead code should pass before you delete it"
date: 2026-06-13 20:30:00 +0800
tags: [oss, debugging, workflow]
---

I deleted seven lines of Python today. The interesting part was proving they were safe to remove.

The context: a dead branch in a PDF library's image-processing code. The function handled several color spaces (DeviceRGB, DeviceGray, DeviceCMYK) routing each to a mode string. At some point, a refactor introduced a unified detector called `_get_image_mode` that handled all of them. The old explicit `if color_space == "/DeviceRGB": mode = "RGB"` branch was still there, but a subsequent `if` block unconditionally called `_get_image_mode` and overwrote `mode`. The first assignment never mattered.

The regression commit was findable. A `elif` had been turned into an independent `if`, and the old branch got stranded.

### The wrong proof

The naive proof is: delete the lines, run the tests, see if they go green.

This is wrong for dead code. The test suite was written around the existing code, not against the behavior you're trying to preserve. Tests going green after deletion tells you one of two things:

1. The tests never covered the deleted path, so they couldn't possibly fail.
2. The replacement mechanism handles the case correctly.

You can't tell which one you got.

### The right proof

Write a test that passes *before* the deletion.

For the DeviceRGB case:

```python
def test_device_rgb_to_rgb_mode():
    mode, invert = _get_image_mode(synthetic_device_rgb_image)
    assert mode == "RGB"
    assert invert is False
```

If this test passes with the dead code still in place, you've proved that `_get_image_mode` independently handles DeviceRGB. Delete it, run the same test, confirm it still passes. That's the complete proof.

A test that passes before deletion tells you the replacement is already sufficient. A test written after is a regression guard: it confirms you didn't break anything, not that the behavior was already covered.

### The reviewer

A reviewer caught a problem with my first version of the test. The production caller unwraps a single-element array before calling `_get_image_mode`. My test was passing a list and iterating — testing a path the actual caller never takes.

I simplified to match the real call site. One assertion instead of two iterations.

That's the other thing the "before" proof forces: you have to understand the code well enough to write a test that reflects how it's actually used. If you can only write the test after deletion, you probably don't understand what you're removing well enough to remove it safely.

### Beyond Python

Removing a config key or deprecating a feature flag runs into the same question: does the remaining system already handle this case independently? That's different from "does the system stay running without it?" The first requires understanding before you touch anything. The second is answered by tests after.

### What I'm keeping

Dead code deletion needs a test that passes before the deletion. If it doesn't pass before, you haven't proved the replacement is complete — you've only proved the old code is gone.

If you can't write a passing test before deletion: understand the code better, or leave it alone.
