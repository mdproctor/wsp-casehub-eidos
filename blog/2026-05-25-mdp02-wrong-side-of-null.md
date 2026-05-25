---
layout: post
title: "Wrong Side of Null"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [tdd, null-safety, code-review, mutiny]
---

The issue description for eidos#2 said `query.slot().equals(d.slot())` would NPE when `d.slot()` is null. It was wrong. Java's `String.equals(null)` returns false — the argument can be null without trouble. NPE fires only when `.equals()` is called *on* a null reference. The real bug was one filter down:

```java
d.capabilities().stream().anyMatch(c -> c.name().equals(query.capabilityName()))
```

When `c.name()` returns null — which the JPA entity prevents but in-memory registration does not — that throws. The fix in both places was `Objects.equals()`, symmetric and null-safe on both sides. But the issue had identified the wrong filter line, and a naive fix would have corrected the wrong thing.

The second catch came from the code review. We wrote two null-slot regression tests. The first verified that a null-slot descriptor is excluded from a slot-only query. Correct. What we hadn't written was the complementary case: that a null-slot descriptor is *included* in a capability-only query. Excluded-when-you-should-include is a different bug to NPE, and Claude flagged the missing assertion before the commit. We added the test.

The rest of the session was cleanup from Phase 1 and Phase 3 reviews. The Javadoc on both JPA entity classes now explains why the class is public but the fields are package-private: Hibernate Reactive's bytecode enhancement needs a public class to instrument it, but accesses fields directly and doesn't need getters. Without the comment, the combination looks like an oversight. `InMemoryReactiveAgentRegistry.register()` changed from calling the delegate synchronously before returning a void Uni to the Mutiny pipeline form — `invoke(delegate::register).replaceWithVoid()` — which is the right template for any implementer who later adds I/O. And `ClaudeMarkdownRendererTest` had four tests sharing ~14 lines of boilerplate each to capture what the renderer sends to the LLM. We extracted `renderAndCaptureYaml()` and each test became a single assertion. `captured[0]` was also initialised to `""` instead of `null` — a small thing, but a null there produces an NPE instead of a useful AssertJ failure message.

`ClaudeMarkdownRenderer.sha256()` truncates to 16 hex chars. The comment was missing, and "why 16?" is exactly the question a reader would ask. 64 bits is adequate for cache-invalidation fingerprinting — you're comparing equality across a small set of cached prompts, not providing cryptographic guarantees. It now says that.
