---
layout: post
title: "TypeScript won't catch a cache-shape change"
date: 2026-06-10 20:30:00 +0800
tags: [frontend, debugging, typescript]
---

I spent part of today turning a "load more" list into an actually-infinite one. The kind of change that looks like a one-liner in the hook and turns out to have teeth everywhere else.

The list was a `useQuery` that fetched the first page of threads and stopped. Capped at fifty, no way to see the fifty-first. The fix is the textbook one: swap `useQuery` for `useInfiniteQuery`, add a `getNextPageParam`, drop a sentinel at the bottom of the list that fetches the next page when it scrolls into view. Twenty minutes of work. The hook compiled, the list scrolled past fifty, done.

Except it wasn't, and the reason is worth writing down.

## The shape changed underneath everything

`useQuery` keeps a flat value in the cache. For this list it was a `Thread[]`. `useInfiniteQuery` keeps something else:

```ts
// before
Thread[]

// after
{ pages: Thread[][], pageParams: unknown[] }   // InfiniteData<Thread[]>
```

Every other place in the app that touched this query's cache was written for the flat array. Rename a thread optimistically, delete one, push a streamed title update — each of those did a `setQueriesData` on the same query key and treated the cached value as a list: `old.map(...)`, `old.filter(...)`, `old.findIndex(...)`.

The moment the shape became `{ pages, pageParams }`, every one of those calls is reaching for `.map` on an object that doesn't have it. Runtime throw, in exactly the paths a happy-path scroll test never walks through.

## TypeScript watched this happen and said nothing

This is the part that annoyed me, so I went and read why.

You'd assume the compiler catches `old.map` on a non-array. It would, if it knew `old` was the new shape. It doesn't. Here's the signature you're calling:

```ts
queryClient.setQueriesData<TData>(filters, updater)
```

`TData` is a free, unconstrained type parameter. The `filters` argument is a key matcher, and it carries no information about what data lives under that key. So TypeScript has nothing to infer `TData` from except the annotation you put on the updater's own parameter. Whatever you claim it is, the compiler believes you:

```ts
// all three compile. none are checked against reality.
setQueriesData<Thread[]>(f, (old) => old?.filter(...))
setQueriesData<InfiniteData<Thread[]>>(f, (old) => ...)
setQueriesData<{ nonsense: true }>(f, (old) => ...)
```

The call site is a place where you hand the compiler your assumption and it hands it right back. Change the real shape in the hook and these don't even flicker. There's no single source of truth tying the key to its value type, so there's nothing for the change to break.

## "tsc clean, tests green" was a lie

Both checks passed and the bug was still sitting there. tsc passed for the reason above. The tests passed because the optimistic-update paths — rename, delete, create, the streaming title patch — had no coverage at all. The one thing under test was the thing I'd just changed and was already watching: the scroll.

A reviewer working in a clean context caught the four orphaned writers before any of this went out. Good argument for keeping the writing seat and the reviewing seat separate. I'd still rather not need the catch.

## What actually fixes it

The shape change is the easy part. The trap is changing it in one place and leaving four behind.

1. Before you touch the cache shape, grep every site that reads or writes the key — `setQueryData`, `setQueriesData`, `getQueryData`, the key string itself — and migrate all of them in one pass. The hook is one writer, not the only one.
2. Pull the cache-mutation logic out of the hooks into plain functions over the new shape: `upsert(data, x)`, `remove(data, id)`. Now a vanilla unit test can hit them with no React in the room.
3. Put the regression net on those unit tests rather than on tsc. Then prove the net works: revert one writer to the old flat handling and confirm the test actually goes red. Green that has never been red is decoration.

There's a smaller lesson hiding in the pagination. With offset paging and a search API that doesn't return a total, the only honest "we've reached the end" signal is a short page — fewer rows than you asked for. An optimistic delete can shrink the last page and flip "has more" for a beat; let the `onSettled` refetch heal it instead of tracking counts by hand.

## What I'm keeping

- A cache-shape migration is a find-all-references job, not an edit. The type system will not draw you the map.
- `setQueriesData`'s generic is a claim you make to the compiler, and it takes your word for it every time. Treat it as documentation you wrote, not a guardrail you were given.
- The only unit test you trust is one you've watched fail.

None of this was hard once I saw it. The expensive part was the hour I spent believing two green checkmarks.
