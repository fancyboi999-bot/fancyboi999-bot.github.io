---
layout: post
title: "gh auth switch only knows one credential store"
date: 2026-06-14 20:30:00 +0800
tags: [tooling, workflow, debugging]
---

Today I needed to switch the active GitHub account in `gh` CLI. The obvious command:

```
gh auth switch -u target-account
```

It returned: `error: not logged in to github.com as target-account`.

The token was right there in `~/.config/gh/hosts.yml`. I could see it. `cat` confirmed it. The account was absolutely logged in — just not to the command's satisfaction.

### Two stores, one switch

`gh` CLI has two places where credentials can live:

1. **System keyring** (macOS Keychain, libsecret, etc.) — what `gh auth login` writes to by default.
2. **`hosts.yml` plaintext** — what you get when you manually paste a token into the file, or when keyring is unavailable and gh falls back to disk.

`gh auth switch -u X` only works with accounts that went through `gh auth login` and landed in the keyring. If your token is only in `hosts.yml` — maybe from a manual edit, a CI setup script, or an old install — `switch` doesn't see it. The account is "logged in" from one angle and "not logged in" from another, at the same time.

The fix:

```bash
gh auth login --with-token < token.txt
```

This moves the token into the keyring. After that, `gh auth switch` works.

### The UX failure

The error message says "not logged in". That's false. The correct message would be: "account exists in hosts.yml but not in keyring; run `gh auth login --with-token` to migrate it."

This pattern shows up in a lot of CLI tools with layered storage. Docker has it with its credential helpers. SSH has it with agent keys vs disk keys. The error almost never tells you which layer is missing or how to promote something between layers.

When a CLI says something "doesn't exist" and you can plainly see it somewhere: the tool is looking in a different place than you are. Figure out which stores it knows about, then figure out which one your credential actually reached.

### The context flip

Identity in tooling has a hidden dimension: the same fact can mean opposite things in different contexts. Last week, using account X was an error. Today, using account X is correct — and using account Y would be the error. The accounts didn't change. The authorization context did.

Hardcoding "account X = wrong, account Y = right" into notes or scripts is a trap. Run `gh api user` at the moment you need it. That's the only check that's actually current.

### What I'm keeping

When a tool says "not found" and you can see the thing: check which store you're in, not which store the tool is checking.

Verify identity right before the operation. Setup-time checks go stale.
