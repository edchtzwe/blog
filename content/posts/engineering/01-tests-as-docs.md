+++
title = 'Tests Are Your Fastest Map of an Unfamiliar Codebase'
date = '2026-10-03'
draft = false
tags = ['testing', 'tdd', 'documentation', 'onboarding', 'agents']
categories = ["Engineering", "Testing"]
+++

You've just joined a service with a few hundred files, thin docs, and one bug report: readiness is flaky. Where do you start?

Not the README. The test suite. Before you can change anything safely, you need to know what the system does, which boundaries matter, and what must not break. Tests are the cheapest place that knowledge already exists — written down, and executable.

Docs and tests answer different questions. An ADR explains *why* a design was chosen. A test shows *what must still be true*. When you're new to a system, the "what" is what keeps you from breaking it.

That gap widens when an AI agent is doing the reading. A human can skim a design doc for intent; an agent needs an executable target. Tests give both of you the same thing: a contract you can check.

---

### Start With the Test Run

Run the suite with verbose output, and the test names become an inventory of the system. Seven lines can sketch a whole service:

```text
 ✓ src/common/url-safety.spec.ts > validateTargetUrl > rejects URLs with embedded username or password
 ✓ src/config/config.service.spec.ts > ApiConfigService > rejects an out-of-range port
 ✓ src/config/config.service.spec.ts > ApiConfigService prod mode > refuses to start when Vault connection settings are absent
 ✓ src/config/config.service.spec.ts > ApiConfigService prod mode > treats an unrecognised mode as prod
 ✓ src/auth/jwt-auth.middleware.spec.ts > JwtAuthMiddleware > rejects with 401 and WWW-Authenticate header when authorization header is missing
 ✓ src/readiness.service.spec.ts > ApiReadinessService > rejects readiness when the job queue is unavailable
 ✓ src/jobs/jobs.service.spec.ts > JobsService > maps BullMQ states to canonical JobStatus enum
```

Read them the way a new engineer would:

- URL validation rejects credentials smuggled into a URL — an SSRF guard.
- Config rejects an out-of-range port.
- In prod, the service refuses to start without Vault connection settings — fail fast, not fail later.
- An unrecognised mode is treated as prod — the safe default.
- Auth returns `401` with a `WWW-Authenticate` header when the header is missing — the exact contract a client sees.
- Readiness depends on the job queue; if it's down, the service reports not ready.
- Job states map to a canonical enum — an internal model with one source of truth.

That is the skeleton of the service: its security boundary, its config rules, its auth contract, its dependency gates, and its domain model. You got it from one command.

Run it once and hand the output to an agent. Instead of opening every file to infer the feature set, it now has a behavioural index with paths to the specs that matter. Fewer speculative reads, fewer tokens, faster context. For a small suite, show the whole output; for a large one, show the relevant slice. The point is signal, not volume.

Test names decide how well this works. `works correctly` tells an agent nothing. `rejects readiness when the job queue is unavailable` names a dependency and a failure condition. Verbose output is worth the noise only when it carries meaning.

And the index is not proof. The Redis test might hit a real backend or a mock; a green line doesn't tell you which. Read the spec before you treat it as evidence that an integration works.

---

### Turn a Bug Report Into a Handoff

"Readiness is broken" leaves an agent guessing at what broken means. A failing test named `rejects readiness when the job queue is unavailable`, with real setup and a meaningful assertion, narrows it immediately.

Picture it: the test makes the queue unavailable and asserts that readiness must fail — but the implementation still reports ready. Now you can point the agent at the failing spec and say: fix this, without weakening the assertion.

The handoff now contains the starting location, the conditions that reproduce the problem, and the expected result. That beats three rounds of chat, and rerunning the test tells you whether that specific behaviour is repaired.

A failing test is not a root-cause diagnosis — the fault could live in a dependency, the config, or the test itself. But it gives the investigation a concrete entry point. If you haven't reproduced the bug yet, point the agent at the area and have it write a regression test that fails for the reported reason before touching the implementation. Then run the surrounding suite as well: one test passing proves a local fix, not that you preserved every neighbour.

---

### Write the Requirement Before the Implementation

This is where TDD stops being ceremony and starts earning its keep — specifically with agents.

If an agent writes an implementation and then writes tests by copying its own assumptions, both can agree on the same mistake. The suite goes green and proves consistency, not correctness. A test written from the requirement first gives the agent a target that is independent of the code it's about to write.

For a bug, that means reproducing the wrong behaviour first. For a feature, it means asserting the externally meaningful outcome instead of prescribing every internal call. And the red step matters: a test that fails because an import is broken hasn't shown it can detect the missing behaviour. Inspect the failure, not just the colour.

Tests written afterwards can still be valuable. The point isn't timing purity; it's whether the expectations come from the contract you intend. TDD theatre optimises for the look of a process. Useful TDD removes ambiguity before the change and gives you a check after it.

---

### Coverage That Actually Protects

The more real behaviours a suite checks, the less you and the agent have to guess. Cover the happy path, rejected inputs, boundary conditions, and failures at important dependencies — the cases that make it hard to silently drop behaviour the team relies on.

Push for high coverage, but make the number earn its place. Line and branch coverage show what ran, not whether the assertions would catch a defect. A test can execute a whole method and verify almost nothing.

Concretely: for a jobs endpoint, checking that the handler ran is weaker than checking the status code, the response payload, and the `Location` header. For queue readiness, a mocked unit test can check how an unavailable backend is handled, but you need an integration test to check the real backend interaction. Pick the level that can actually observe the boundary you're claiming to protect.

When a test fails, fix the implementation or fix a genuinely defective test. Don't delete a valid expectation to get a green run — remove tests only when the behaviour they cover is truly gone.

---

### Make the Contract Easy to Find

Colocation removes one more discovery cost: `user-service.ts` and `user-service.spec.ts` sit beside each other, in separate files with an obvious relationship. An agent opening the source finds the expectations without searching a parallel tree.

Proximity doesn't guarantee maintenance. Expect the agent to read the spec before editing, keep valid assertions, cover changed behaviour, and run the tests afterwards. Updating tests should mean reflecting an intended contract change, not loosening them to fit a regression.

Inside the spec, use names that describe behaviour, setup that makes the state clear, and assertions that expose meaningful outcomes. A reader should be able to see what must hold without reverse-engineering the implementation first.

---

The payoff compounds: verbose output shows you where to look, readable specs explain the contract, and a failing regression test hands off a defect precisely. Broad, meaningful coverage then checks that your fix didn't break something else.

Write tests the machine can execute and the next person — or agent — can learn from. You're not just buying a green run. You're lowering the cost of understanding, changing, and repairing the system.
