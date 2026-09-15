---
title: "Why I Stopped Monkey-Patching MCP's InMemoryTransport for Internal Tools"
date: 2026-09-15T12:00:00+10:00
draft: false
categories: ["AI"]
tags: ["mcp", "node", "architecture", "gaas"]
---

When building out my GaaS system, I integrated the Model Context Protocol (MCP) to expose my internal tools. The protocol itself is brilliant for standardizing tool execution and context gathering, but out of the box, most of the ecosystem assumes you're going to communicate over HTTP (SSE) or `stdio`.

For purely internal tools where the components are already tightly coupled, HTTP is just unnecessary overhead. I didn't want the latency, the serialization tax, or the hassle of managing local network boundaries for something that shouldn't leave the execution environment in the first place.

The solution seemed obvious: use Node's `InMemoryTransport`.

### The Context Problem (And My Previous Mistake)

My GaaS architecture relies heavily on context—specifically directory (`dir/`) ownership and authentication context—to ensure processes only touch what they're explicitly allowed to. In a standard HTTP transport, you'd extract this from headers via middleware and resolve it into `AuthInfo`.

I previously thought the vanilla `InMemoryTransport` didn't natively carry this kind of arbitrary execution context, which led me to a terrible architectural decision: monkey-patching `AuthInfo` directly into the transport lifecycle. 

Monkey-patching is almost always a code smell, and in this case, it was completely unnecessary.

### The Native Solution

It turns out `InMemoryTransport` was built specifically to handle `AuthInfo` transport natively. 

As documented in the MCP SDK, the transport's `onmessage` callback accepts an `extra` parameter that explicitly includes `authInfo` (and `requestInfo`). Instead of intercepting and hacking the transport, you can pass your custom `AuthInfo` object—including the `dir/` ownership data—directly through the native API. 

When an internal tool gets called via the MCP server, it receives this context just as if it had been securely authenticated via a web request, but it happens instantly, in memory, and without any dirty hacks.

### The Payoff

1. **Zero HTTP Overhead:** No parsing headers, no TCP handshakes, no network stack. Just raw object passing.
2. **Strict Internal Context:** By using the native `AuthInfo` support, I maintain the strict directory ownership boundaries my system requires without needing to stand up a complex auth middleware layer.
3. **Clean Architecture:** The internal services don't need to know they aren't on HTTP, and the codebase remains free of fragile monkey-patches.

Sometimes the cleanest architectural choice is just reading the documentation properly. Using the native `AuthInfo` support gave me the exact performance profile I needed while keeping the GaaS security boundaries perfectly intact.
