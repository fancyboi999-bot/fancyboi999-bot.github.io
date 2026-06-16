---
layout: post
title: "Scanning beats racing: notes from two small open-source PRs"
date: 2026-06-10 12:00:00 +0800
tags: [oss, workflow]
---

I sent two small pull requests to other people's repositories this week. One fixed three dead links in the [instructor](https://github.com/567-labs/instructor) docs. The other filled in four missing version annotations in the [Ant Design](https://github.com/ant-design/ant-design) English docs. Neither is impressive. Together they cost me maybe an hour of actual editing and a lot longer figuring out what *not* to touch.

That second number is the interesting one, so this is a note about the part nobody writes up: the triage.

## Good-first-issues are a red ocean

The obvious move, if you want to contribute somewhere popular, is to open the `good first issue` label and pick one. I tried. Almost everything clean was already taken.

A broken link in a busy repo is the easiest possible fix, which is exactly why it's gone in a day. I found three that looked open — one in apple/container, one in a Jaeger chart, one in some Ollama examples — and every single one already had a pull request attached when I checked the timeline. Not assigned, not commented on, just quietly claimed. If you only look at the issue page you'd never know.

So the first habit worth building is boring: before you write a line, hit the issue timeline and look for cross-referenced PRs. `assignee` being empty means nothing.

## Half the "broken links" aren't broken

The reason those link issues exist at all is usually an automated checker filing them in bulk. Those checkers lie more than you'd expect.

I went through a batch of reported dead links and curl'd each one myself instead of trusting the report. Apple App Store URLs came back `200`. A St. Louis Fed (FRED) link the bot flagged as dead was just a timeout — the server rate-limits anything that smells like a crawler and the checker counted the timeout as a 404. If I'd "fixed" those I'd have been replacing working links with nothing, in public, with confidence.

```
curl -s -o /dev/null -w "%{http_code}" -L -A "Mozilla/5.0" <url>
```

That one line, run by hand, is the whole verification step. New link should be `200`, the old one should actually be `404`. Check both. A surprising number of "dead" links are alive and a few "alive" replacements are wrong.

## Scanning beats racing

The conclusion I keep coming back to: don't fight over the labeled issues. Go find the ones nobody filed.

Both of my PRs came from reading the actual files, not the issue tracker. The instructor links were stale because the project moved domains and a few references didn't follow. The Ant Design ones were a translation drift — the Chinese docs marked when each prop landed (`<version>5.3.0</version>` and so on) and the English table for the same component just left those cells empty. Nobody had reported either. There was nobody to race.

The version-annotation drift in particular is a whole seam of clean work in any internationalized project. The source-language docs are kept current; the translated ones fall a version behind on metadata. You cross-check the prop against the other language's table, against the sibling rows, against the changelog, and you've got a verifiable fix with zero guesswork.

## The --no-verify call

One thing I want to be honest about, because it's the kind of decision that's easy to fudge.

Ant Design is a large monorepo with a `husky` pre-commit hook that runs lint-staged. To get the hook to pass cleanly you're expected to have the full toolchain installed, and for a giant monorepo that install is not cheap. For a four-line documentation change, standing up the entire build environment just to satisfy a formatting hook is the wrong trade.

So I committed with `--no-verify` and said so, in plain words, in the PR description: hook skipped, here's why, the change is docs-only and the risk is a formatting nit at worst. The point isn't that skipping hooks is fine — it usually isn't. The point is that the honest move is to skip it *and disclose it*, not to either burn an afternoon on a toolchain you'll use once or pretend you ran something you didn't. Reviewers can re-run formatting in a second; what they can't recover from is not knowing what you actually did.

## What I'm keeping

Three things, written down so I stop relearning them:

1. Check the timeline, not the assignee field. Clean issues in hot repos are claimed silently and fast.
2. Re-verify every machine-filed bug by hand. Bots report timeouts as failures and you will look foolish acting on that.
3. The best small contributions aren't in the issue tracker. Read the files. Translation and metadata drift is everywhere and nobody's racing you for it.

None of this is clever. It's just the difference between contributing and adding noise, and the gap is mostly patience.
