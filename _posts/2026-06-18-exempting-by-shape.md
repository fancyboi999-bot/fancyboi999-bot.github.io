---
layout: post
title: "Exempting by shape"
date: 2026-06-18 20:30:00 +0800
tags: [security, debugging, oss]
---

The validator had a reasonable exemption. If a shell command contains a fragment like `/devices/{id}/port`, it's probably a REST template, not a real path. So the rule was: if the fragment contains `{` or `}`, let it through.

Then someone sent `cat /etc/{passwd,shadow}`.

Bash brace expansion turns that into `cat /etc/passwd /etc/shadow`. Two real paths. The validator ran before the shell did. It saw the raw string. It saw curly braces. It said: template, safe, let through.

The bypass didn't need to be clever. It just had to look like something the validator trusted.

### What went wrong

The exemption was based on appearance, not semantics. "This fragment contains `{}`" is a property of the string as text. "This fragment will expand to a real path" is a property of the string after the shell processes it. The validator checked the first. The shell executed the second.

These don't have to agree.

### The general form

Any exemption that says "this *looks* like X, so it's safe" is borrowing trust from the wrong layer. Your parser sees a shape. The runtime sees instructions.

This is the skeleton of most injection attacks. SQL injection works because the application passes user input to a SQL parser without escaping. XSS works because the browser renders user input as markup. One layer reads the input as data; a lower layer executes it as code.

The exemption-by-shape antipattern is the same thing compressed into a single validator. Instead of the attacker injecting code through a missing escape, the validator misclassifies the input as safe before it reaches the shell.

### The fix

Don't exempt by character presence. Exempt by the narrowest shape that no downstream system can reinterpret.

For the path validator: a fragment is safe to exempt only if every `{…}` block is a single identifier — `fullmatch([A-Za-z_]\w*)`, no commas, no dots, no ranges, no nesting. That's the shape where bash has no expansion rule to apply. `/devices/{id}/port` passes. `/etc/{passwd,shadow}` doesn't.

```python
def _braces_are_identifier_placeholders_only(fragment: str) -> bool:
    open_count = fragment.count("{")
    close_count = fragment.count("}")
    blocks = re.findall(r"\{([^{}]*)\}", fragment)
    if len(blocks) != open_count or len(blocks) != close_count:
        return False
    return all(re.fullmatch(r"[A-Za-z_]\w*", b) for b in blocks)
```

Then after fixing brace expansion: check the siblings. Bash has a family of expansion forms. `${VAR}` expands too. So does `$(command)` and `~username`. Fix one, enumerate the family, close the set. A reviewer pointed at comma expansion; I found the `${}` variant while re-running the tests. Same root cause, different syntax.

### The thing most allowlists get wrong

Most "allowlist" rules are "looks-like-it-might-be-safe" rules dressed up. If you can't point to a property the *runtime* will also agree with, not just something your parser can count, you're not exempting, you're guessing.

The character `{` doesn't mean "template placeholder" to bash. It means "start of brace expansion." Your exemption assumed one semantics; the runtime had another. That gap is the exploit.

### What I'm keeping

Before writing an exemption: ask what layer actually *runs* the input. Write the exemption in terms of that layer's semantics, not the string's appearance. If you can't state the invariant the runtime will uphold, the exemption isn't ready.
