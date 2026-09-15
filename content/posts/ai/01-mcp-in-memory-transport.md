+++
title = 'No HTTP, No Hacks: In-Process MCP with the Node SDK'
date = '2026-09-15'
draft = false
tags = ['mcp', 'node', 'architecture', 'gaas']
categories = ['AI']
+++

When building out my GaaS system, I integrated the Model Context Protocol (MCP) to expose my internal tools. The protocol itself is brilliant for standardising tool execution and context gathering, but the spec ships exactly two standard transports: `stdio` and Streamable HTTP (SSE before it). Everything else is explicitly a "custom transport, implemented in a pluggable fashion."

I'm prototyping an MVP here, not shipping a production topology. My Express server and my MCP server run in the same process. For a tool call that never leaves that process, standing up HTTP is pure overhead — latency, a serialisation tax, and a network boundary to manage for something that shouldn't leave the execution environment in the first place.

So I reached for the Node SDK's `InMemoryTransport` and got to shore.

### First, Credit Where It's Due

`InMemoryTransport` is not part of MCP. It's a TypeScript/Node SDK class — `InMemoryTransport.createLinkedPair()` links a client and a server inside a single process, no network, no child process. The SDK docs present it as the third transport and build their testing guide on top of it. The protocol doesn't care that I'm using it; the protocol only cares that JSON-RPC messages arrive.

### The Context Problem (And My Previous Mistake)

My GaaS architecture relies heavily on context — specifically directory (`dir/`) ownership and authentication context — to ensure processes only touch what they're explicitly allowed to. Over HTTP, you'd extract this from headers in middleware and resolve it into `AuthInfo`.

I'd assumed the in-memory transport couldn't carry this kind of arbitrary execution context, which led me to a terrible architectural decision: monkey-patching `AuthInfo` directly into the transport lifecycle.

Monkey-patching is almost always a code smell, and in this case it was solving a problem that didn't exist.

### Where AuthInfo Actually Lives

The SDK already has a slot for it, and that slot sits exactly where it belongs: at the transport boundary.

`Transport.onmessage` receives `(message, extra)`, where `extra` is a `MessageExtraInfo` — `{ requestInfo?, authInfo?, ... }`. The protocol layer copies that straight through, so a request handler sees `extra.authInfo`. And the in-memory transport's `send` accepts it directly:

```ts
await transport.send(message, { authInfo });
```

The transport supplies the context, the SDK carries it, and the tool handler receives it — no interception, no hacking the lifecycle. The SDK doesn't care *how* `AuthInfo` arrives, only that it does; that's the transport's problem. Mine just happens to solve it in memory.

### Be Honest About What This Is (And Isn't)

In-process `authInfo` is **not authentication**. Nothing verified anything — it's trust-by-construction inside the same process. It's context passing, and it's only as trustworthy as the code that hands it over. Calling it "securely authenticated" would be overstating it badly.

### Why This Was the Right MVP Call

1. **No HTTP overhead:** No parsing headers, no TCP handshakes, no network stack. Just object passing.
2. **Context intact:** By using the transport's native `AuthInfo` support, I keep the strict `dir/` ownership boundaries the system requires without standing up an auth middleware layer.
3. **Clean architecture:** The internal services don't need to know they aren't on HTTP, and the codebase stays free of fragile monkey-patches.
4. **No cleanup debt:** I'm on Node/TS anyway, so wiring this up took less effort than hacking around the gap — and left nothing behind to unpick later.
5. **Cheap to port later:** A well-configured agent — my Codex1 OpenClaw agent, or the HermesX agents (more on those in a later post) — can knock the port out in a Saturday-afternoon hackathon. Prototyping on the in-memory transport doesn't lock me in.

### The Exit Plan

This is rapid prototyping only, to get an MVP out the door. MCP servers are almost never deployed alongside the app in production — that's a scaling nightmare. Production will be Streamable HTTP, with the MCP server deployed separately.

Because the tool wiring is already proven through `InMemoryTransport`, porting over is roughly an hour's work with AI assistance: swap the transport, keep the handlers, keep the context shape.

Sometimes the cleanest architectural choice is just reading the documentation properly.
