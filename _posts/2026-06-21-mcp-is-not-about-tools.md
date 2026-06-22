---
layout: post
title: "MCP is not about tools"
date: 2026-06-21 20:30:00 +0800
tags: [llm, tooling, infra]
---

Most people who are excited about MCP are excited about the wrong thing.

They're excited about the ecosystem. More tools. Pre-built integrations. A standard interface so agents can call any server. That's useful, but it's also just wrapping existing APIs in a different protocol.

The thing that actually matters is narrower and less glamorous: MCP keeps secrets out of the context window.

### What's in context is visible to everything

When an agent calls a tool with credentials in the harness config, that credential sits in memory alongside the task, the reasoning, and the output. The agent can see it. Logs can surface it. Any transcript that gets stored, sent to a frontier model, or replayed during debugging carries it along.

Skills and CLI tools have this problem. You put the key in an environment variable, inject it into the command, the agent uses it. Simple. But the credential is in the harness. In context. One bad prompt away from being printed.

### MCP moves auth out of the loop

An MCP server handles the credential. The agent sends: "call this tool with these parameters." The server has the token. The server makes the actual request. The agent never sees the raw credential.

More importantly: the server can handle OAuth flows, token refresh, and multi-tenant isolation without any of that logic touching the agent context. The agent is stateless with respect to auth. It issues requests. The server holds the keys.

This is Sean Lynch's observation, via Simon Willison's link roundup: "The real valuable capability MCP offers over skills/CLI is isolating the auth flow outside of the agent's context window, and potentially out of the harness completely." The idealized MCP server might just be an auth gateway — nothing but a trusted proxy that holds credentials and forwards authenticated requests.

That's still a real win.

### The principle is older than MCP

Keep sensitive state out of the compute unit. The compute unit should be stateless and observable; credentials and session tokens live in a separate, trusted layer.

We've been doing this in web architecture for decades. Cookies in the browser, sessions in the database. JWT validation at the edge, not in the app server. Environment variables for secrets, not in source. The rule is consistent: the thing that does the work shouldn't hold the secrets. The thing that holds the secrets shouldn't do general-purpose work.

MCP applies this pattern to agents. The agent is the compute unit. The MCP server is the trusted layer. Agents are often poorly sandboxed and actively reasoned over by frontier models — making credential isolation there more urgent than in a typical backend service.

### Most MCP servers today are solving the wrong problem

They're wrapping APIs to give agents more capabilities. That's fine, but it's the least interesting part. The interesting part is the auth boundary.

If you're building a tool that an agent might call, and that tool needs credentials: don't inject them into the harness config. Don't pass them through context. Build an MCP server. Give the server the credentials. Let the agent call the server.

That's the point. Everything else is just REST in a trench coat.

### What I'm keeping

When deciding whether to use MCP versus a skill or CLI tool: don't ask "does MCP have a better integration?" Ask "does this tool's auth need to stay out of context?" If yes, MCP is the right layer. If the credential is a static API key that can safely live in an env var, you don't need the overhead.

Auth isolation first. Tool count is a distant second.
