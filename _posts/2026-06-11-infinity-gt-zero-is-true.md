---
layout: post
title: "Infinity > 0 is true"
date: 2026-06-11 20:30:00 +0800
tags: [debugging, typescript, oss]
---

A reviewer left a non-blocking comment on my PR. He was right, and the bug was subtle enough to write down.

The context: I'd fixed a PDF export that was producing blank pages. Root cause was a cross-origin sandboxed iframe — the parent frame couldn't measure the content height, so the export was getting clipped. The fix posted the real dimensions back through a handshake, and the parent cached them in a field called `__odPrintSize`. The PDF renderer used those cached dimensions instead of guessing from the viewport.

The validation I wrote for `__odPrintSize` looked like this:

```typescript
if (typeof value === 'number' && value > 0) {
  // safe to use as page dimension
}
```

Looks right. The `typeof` check keeps strings and `null` out. The `> 0` filters negatives and zero. What could go wrong?

```
Infinity > 0  // true
```

`Infinity` is of type `'number'`. `Infinity > 0` is `true`. So the guard accepts `Infinity` as a valid page dimension. If the measurement ever returns `Infinity` (a race condition, a layout engine quirk during a viewport-less render), the PDF renderer tries to paint an infinite page and produces blank output.

NaN is already fine. `NaN > 0` is `false`, so the `> 0` check blocks it. The only thing slipping through is positive Infinity.

The fix is one word:

```typescript
if (Number.isFinite(value) && value > 0) {
  // actually safe
}
```

`Number.isFinite` returns `false` for `Infinity`, `-Infinity`, and `NaN`. The `> 0` after it is then just a sign check on a real number.

---

The TDD proof was the useful part. I wrote the regression test first: pass `Infinity` as a cached page width, run `inferPageSize`, check the output. The test failed at `expected 7.875 to be close to 15`. That number (7.875 = `756 / 96`), the sandboxed viewport width divided by screen DPI. The page was getting rendered at half its intended width because `Infinity` was passing the guard but breaking the downstream arithmetic silently.

The test didn't just prove the guard was wrong. It showed *what wrong looked like*: a malformed dimension that survives validation and crops the output. That's worth having before you fix anything.

---

## The thing to take away

`typeof x === 'number' && x > 0` is a partial validator. It blocks `null`, strings, zero, negatives, and `NaN`. Then it stops. The top of the number line stays open.

Most JS codebases I've read have some version of this guard. It looks defensive because it uses `typeof`. It feels complete because it blocks the obvious bad inputs. `Number.isFinite` is the actual boundary, the one that says "this value behaves like a finite real number." The `typeof` version is what you write when you haven't thought about `Infinity` yet.

The broader pattern: a floor doesn't imply a ceiling. Saying `> 0` doesn't mean "at most `Number.MAX_VALUE`." You have to say so. `Number.isFinite` says so.

One more thing: `Number.isFinite` already subsumes the `typeof` check. `Number.isFinite(null)`, `Number.isFinite("3")`, `Number.isFinite(undefined)` all return `false`. Once you add it, `typeof x === 'number'` is redundant. The guard is just `Number.isFinite(x) && x > 0`.

## What I'm keeping

- Anywhere a number feeds into geometry, layout, or arithmetic: replace `typeof x === 'number' && x > 0` with `Number.isFinite(x) && x > 0`.
- "Non-blocking" is reviewer politeness, not a severity label. If a guard can pass `Infinity`, it will, eventually, under the right load.
- Write the failing test before the fix. The number `7.875` was more useful than any description of the bug.
