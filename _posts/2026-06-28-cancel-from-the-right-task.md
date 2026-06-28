---
layout: post
title: "Cancel from the right task"
date: 2026-06-28 20:30:00 +0800
tags: [python, debugging, tooling, async]
---

There's a category of bug where the fix is obviously correct and then crashes in a way the hang never did.

I was looking at an MCP tool-call timeout issue: a hung server would block a run indefinitely because `session.call_tool()` had no timeout. The natural fix is `asyncio.wait_for(session.call_tool(...), timeout=N)`. It adds a deadline. It should work.

It doesn't, because of where the session lives.

The session pool gives each `ClientSession` its own dedicated asyncio task. The tool call isn't running in the caller — it's running in that session's task. `asyncio.wait_for` cancels the awaitable from the *caller's* task. anyio's model is strict about this: cancel scopes must be exited from the same task that entered them. Cross-task cancellation hits a `RuntimeError: cancel scope exited in a different task`. The hang becomes a crash, and the crash is harder to debug than the original hang because it looks like an infrastructure failure, not a timeout.

The SDK already has the right hook: `session.call_tool("name", args, read_timeout_seconds=timedelta(seconds=N))`. That timeout is handled inside the session's own task. It fails as `mcp.McpError`, not `asyncio.TimeoutError`, so you need to check your error-rendering path handles that — but at least it fails *cleanly*.

The general pattern: **when a concurrency model owns task boundaries, apply timeouts at the model's layer, not above it.** If the tool that runs the work also exposes a timeout parameter, use that. Wrapping with `wait_for` from the outside is only safe when you're certain there's no task boundary between you and the coroutine.

A second thing from the same investigation: don't name the config field `timeout`. The server config dict gets passed down to langchain's connection objects, where `timeout` is already a field on HTTP/SSE connections (maps to httpx's connect/read timeout) and an unknown key on stdio connections. A name collision that only manifests at runtime, on the wrong transport, is worse than the original bug. Pick something specific — `tool_call_timeout`, `mcp_tool_timeout` — and keep it out of what the downstream library consumes.

The broader lesson isn't about MCP or anyio specifically. It's that **the surface area of a timeout includes how it fails, not just whether it fires**. A timeout that crashes the process is worse than no timeout. When you're adding one, trace what actually gets cancelled and from which execution context.
