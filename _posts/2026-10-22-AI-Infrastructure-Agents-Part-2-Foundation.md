---
title: "AI Infrastructure Agents: Building the Read-Only Foundation"
date: 2026-10-28
author: Justin Odel
tags: [cloud, ai, agents, devops, iac, terraform, security, policy, bedrock]
description: "Before an agent can propose a fix, it has to safely read the world. Here's the Tier 0 foundation — a read-only agent that can run terraform plan and nothing else, gated by a default-deny policy engine."
series: "AI Infrastructure Agents"
part: 2
---

# Building the Read-Only Foundation

*Part 2 of the series: AI Infrastructure Agents*

## Introduction

* **The Hook:** The scariest moment in an agent platform isn't the first time it writes code — it's the first time it runs *anything* against a real account. So we made the first version structurally incapable of changing a single resource. It can look, it can reason, it can run `terraform plan`. That's it.
* **The Problem:** You can't get to "the agent proposes a fix" without first solving "the agent can safely observe the current state, and is provably unable to do more." That means a runtime to host the agent, a controlled way to expose tools to it, and a policy layer that denies by default.
* **The Takeaway:** This post is the Tier 0 (read-only) foundation: a managed agent runtime, a single read-only tool behind a gateway, and a default-deny policy engine. The design choices here are what every later, more-powerful tier inherits.

---

## Architecture Overview

* **Current State:** We started from nothing — no agent, no tools, no policy. The goal for this milestone was a *safe skeleton*: an agent you can invoke, that can discover exactly one read-only tool, run it, and return an analysis, with a policy engine that would block anything else even if a bug tried to expose it.

```text
   You (CLI / SDK, signed request)
          │  invoke
          ▼
   ┌──────────────────────┐
   │  Agent Runtime        │   managed runtime hosts the agent container
   │  (LLM + tool loop)    │
   └──────────┬───────────┘
              │  discovers tools over a gateway (signed)
              ▼
   ┌──────────────────────┐        ┌───────────────────────────┐
   │  Tool Gateway         │──────► │  Policy Engine (default    │
   │  (authz on every call)│        │  deny; permit read-only    │
   └──────────┬───────────┘        │  "terraform-plan" only)    │
              │  permitted         └───────────────────────────┘
              ▼
   ┌──────────────────────┐
   │  terraform-plan tool  │   clones a repo, runs init + plan,
   │  (isolated runner)    │   assumes a READ-ONLY role in the
   └──────────┬───────────┘   target account, returns the plan
              ▼
      Target cloud account  (read-only; plan never applies)
```

Everything runs in one region, in a single non-production account, with state in a remote backend. The point of this milestone was never breadth — it was proving the safety model on the narrowest possible slice.

---

## The Root Cause (what makes "read-only" actually read-only)

"Read-only" is easy to *claim* and hard to *guarantee*. We leaned on three independent layers so that no single mistake opens a hole:

1. **The tool itself only plans.** The one tool clones a repo and runs `terraform init` + `terraform plan`. There is no apply tool in the gateway to call.
2. **The credential is read-only.** When the plan tool needs access to the target account, it assumes a role scoped to describe/list/get — it physically cannot mutate.
3. **The policy engine denies by default.** The gateway checks every tool call against a policy engine whose only `permit` statement covers the read-only plan tool. Anything else — any tool that doesn't exist yet, any action not explicitly allowed — is denied.

```text
# The safety argument in one line:
no mutating tool exists  ×  the token can't write  ×  policy permits only "plan"
= a confused or injected agent can, at worst, run a plan
```

That "× " matters. Each layer would be enough on a good day; stacking them is what makes it safe on a bad one.

---

## The Solution

### Step 1: A managed runtime to host the agent

Rather than stand up and babysit our own long-running service, the agent runs on a managed agent runtime. We hand it a container image; it handles invocation, scaling, and logging. The agent inside is a straightforward model-plus-tool-loop: it takes a prompt (a compliance finding or a question), discovers available tools, and decides whether to call one.

### Step 2: A gateway so tools are discovered, not hard-coded

The agent doesn't get a hard-coded list of functions. It discovers tools through a gateway over a standard protocol, with every call signed and authorized. This is the seam that lets us add tools later (and gate them by policy) without touching the agent. Today it exposes exactly one: `terraform-plan`.

### Step 3: The plan tool as an isolated runner

The tool is a container that clones the requested repo, optionally assumes a read-only role in the target account, runs `terraform init` and `terraform plan`, and returns a truncated, structured result. It's deliberately boring and deliberately scoped.

```text
clone repo ─► (assume read-only role) ─► terraform init ─► terraform plan ─► return summary
```

### Step 4: Default-deny policy

A policy engine sits behind the gateway. Its posture is **deny everything**, with a single explicit allow for the read-only plan tool. We run it first in a log-only mode to watch decisions and confirm the tool identifiers line up with reality, then promote it to actively enforce. Mutating actions aren't "forbidden by a rule we hope holds" — they're simply not permitted and not present.

---

## Results & Benchmarks

This milestone is about capability boundaries, not latency, so the "benchmarks" are the guarantees:

* **Actions the agent can perform:** exactly one — run a plan.
* **Ways to cause a mutation:** none — no apply tool, read-only token, default-deny policy.
* **Blast radius of a hallucination or prompt injection:** at most, a wasted plan run.
* **Path to more power:** additive — new tools go behind the same gateway and must earn an explicit policy allow.

---

## Lessons Learned

* **What went well:** Starting at Tier 0 made the first production touch boring, which is exactly what you want. The gateway-plus-policy seam means growing capability later is an *addition*, not a redesign.
* **Gotchas to watch out for:** Tool identifiers and policy identifiers have to match exactly, or "enforce" either blocks everything or silently allows the wrong thing. Run the policy in log-only first and read the decisions before you flip it to enforce. Also: managed runtimes often have real constraints on image architecture and format — budget time for the packaging, not just the code.

---

## Conclusion

The foundation is a read-only agent that can run `terraform plan` and provably nothing else, protected by three independent layers — a plan-only tool, a read-only credential, and a default-deny policy engine. It's intentionally narrow, because the value of this milestone is the safety model every later tier builds on.

**Call-to-Action:** When you've bounded an automated system's blast radius, did you rely on one strong control or several overlapping weak ones? I've come to prefer the overlap — curious where others land.

*Next up — Part 3: The Credential Broker — giving the agent just-in-time, short-lived access without ever handing it the keys.*
