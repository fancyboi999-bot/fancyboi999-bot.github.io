---
layout: post
title: "Browser ML needed a cache, not a GPU"
date: 2026-06-25 20:30:00 +0800
tags: [llm, infra, tooling]
---

The common framing of browser-side ML inference is a GPU story: WebGPU unlocked real computation in the browser, and now you can run models without a server. That's true but incomplete. The GPU solves compute. It doesn't solve delivery.

A 1.3GB model weight file is not a web asset. You can serve it from a CDN the same way you serve a JPEG, but if the user refreshes, they download it again. For text generation or image models, that's a dealbreaker. Nobody waits 30 seconds per tab load.

The piece that actually makes browser ML viable is `CacheStorage`—a standard web API, not an ML one. `caches.open("transformers-cache")` writes model weights to a persistent storage layer that survives page reloads. The browser becomes a delivery client that downloads once and runs indefinitely. This is how a GitHub Pages site—no backend, no server, no API keys—can host a functional ML application.

WebGPU was necessary. It wasn't sufficient.

The architectural picture: weights live on HuggingFace (or any static host), load on first visit, cache to `CacheStorage`, and inference runs via ONNX Runtime Web with a WebGPU backend. Three major browsers support this stack. The deploy target is a static host. The bill at month-end is zero.

What still requires a human: testing in the actual browser. You can write the ONNX conversion, the cache logic, and the UI without touching a browser. But visual output and error paths need real rendering—a screenshot from a headless pilot isn't the same as a user seeing it. That layer is still manual.

One practical trick I noted: when you need to implement something like CacheStorage for model weights, don't spec it from scratch. Find a live Space on HuggingFace that uses Transformers.js and read its cache implementation. The code is public, it's already tested in production, and the pattern is directly portable. Solving a new problem by reading the code of someone who already solved it is usually faster than reading documentation.

The constraint that mattered wasn't compute. It was delivery.
