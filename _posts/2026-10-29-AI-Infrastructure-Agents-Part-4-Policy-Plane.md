---
title: "AI Infrastructure Agents: The Policy Plane"
date: 2026-10-29
author: Justin Odel
tags: [cloud, aws, ai, agents, policy, guardrails, cedar, agentcore, devops, iac, security]
description: "Before an agent runs a task, something has to decide whether it should — at what autonomy level, with which tools. Here's the layered, policy-as-code engine that makes that decision, and why every layer can only turn the dial down."
series: "AI Infrastructure Agents"
part: 4
---

# The Policy Plane

*Part 4 of the series: AI Infrastructure Agents*

## Introduction

* **The Hook:** Letting a remediation agent open pull requests against one test repository is easy to justify. Letting the same agent loose on every repository in the organization is not. The first time someone asks "can the agent touch *this* repo?", you discover that a per-agent on/off switch was never going to be enough.
* **The Problem:** Every task needs a decision made *before* it runs: should this agent act at all here, how much autonomy does it get, which tools may it use, does its output have to become a PR? That decision depends on the organization, the agent, the repository, and whichever compliance frameworks apply — all at once.
* **The Takeaway:** We built a policy plane that merges four layers of policy-as-code into a single decision per task, where **the most restrictive layer always wins**. Policies live in git and change only through reviewed pull requests — the same path any infrastructure change takes.

---

## Three kinds of "policy" (and why we had to separate them)

"Policy" turned out to mean three different things, at three different moments:

| Kind | Question it answers | When | How it's enforced |
|------|--------------------|------|-------------------|
| **Autonomy policy** (this post) | Is this agent *allowed* to act here, at what tier, with what tools? | Before the task runs | The task is denied or clamped |
| **Authoring guidance** | *How* should the agent write the code? | While it generates | Soft — injected as context |
| **Validation policy** | Does the produced code *actually* comply? | After the PR opens, in CI | Hard — a failing check blocks merge |

The first governs the *agent*. The other two govern the *code*. Mixing them up leads to a guardrail that tries to do everything and enforces nothing well. This post is about the first.

We also already had a runtime policy engine: **AgentCore Policy**, the default-deny **Cedar** engine attached to the AgentCore Gateway (from Part 2). It answers "may this specific tool call happen right now?". That stays exactly where it is. The new plane sits **above** it and answers a broader question about the whole task. Two complementary layers, not a replacement:

| | Cedar (AgentCore Policy) | PolicyDecision (this post) |
|---|---|---|
| Scope | One tool call | One whole task |
| When | At the gateway, at call time | Before the task is dispatched |
| Answers | "May this call to `terraform-plan` proceed?" | "Should this task run, at what tier, with which tools?" |

We deliberately didn't try to express the task-level decision in Cedar. Cedar is excellent at authorizing a principal/action/resource triple; it's not where you want to merge four owners' YAML into tiers, timeouts, and approval counts.

---

## Architecture Overview

```text
  policies/autonomy/
  ├── organization.yaml        the ceiling — the loosest anyone can ever get
  ├── agents/<agent>.yaml      what this agent is granted
  ├── repositories/<repo>.yaml whether this repo opted in, and its conventions
  └── compliance/<fw>.yaml     caps that apply across everything
                │
                ▼   merge: most-restrictive-wins
        ┌────────────────────────────────────────────┐
        │ PolicyDecision                             │
        │   allowed, tier (0–4), requiredApprovals   │
        │   constraints: maxIterations, timeoutMs,   │
        │     allowedTools, deniedTools,             │
        │     requiresPR, requiresValidation         │
        │ + digest: human-readable rules for the agent│
        └────────────────────────────────────────────┘
```

The autonomy tiers run from **0 Observe** (read-only) through **1 Recommend**, **2 Draft** (open PRs, never merge or apply), **3 Sandbox Execute**, to **4 Prod with Gates**. Our first production use case lives at Tier 2.

---

## The Root Cause (why one setting doesn't scale)

The tempting design is a single config per agent: `max_tier: 2`. It falls apart the moment two owners care about the same task. The platform team wants an org-wide ceiling. A repository's team wants to keep agents out of a sensitive repo entirely. Security wants a compliance framework to cap anything touching regulated surface area. Each of those people is right, and none of them should have to edit the others' config to get their way.

Layering solves the ownership problem, but only if the merge rule is unambiguous. Ours is one sentence: **no layer can loosen what a stricter layer set.**

---

## The Solution

### Step 1: Policies are files, changed by pull request

Each layer is a small YAML file in the platform repo. Enrolling a repository is one file:

```yaml
layer: repository
repository: "owner/some-repo"
enrolled: true
constraints:
  max_tier: 2            # Draft — the agent may open PRs here
  required_approvals: 1
conventions:
  - "Terraform only; follow the existing module layout."
  - "Branch names: agent/<finding-id>; never push to a protected branch."
```

Because it's git, every guardrail change has an author, a reviewer, and a history. Code ownership rules let you put the org and compliance layers behind platform/security review while delegating repo enrollment to the repo's own team.

### Step 2: An explicit merge table

| Field | Rule |
|-------|------|
| `tier` | Start at the agent's grant, then clamp down by repo cap, compliance cap, and org ceiling |
| `maxIterations`, `timeoutMs` | Clamp to the org ceiling |
| `deniedTools` | Union across layers — a denied tool can never be re-allowed |
| `allowedTools` | The agent's set minus every denied tool |
| `requiresPR`, `requiresValidation` | Sticky: once true at any layer, stays true |
| `requiredApprovals` | The maximum across layers |
| `allowed` | False if any layer says so |

The apply tool is denied at the org layer, so it can never reach an agent's allowed set no matter what an agent file says. (And even if it somehow did, it isn't registered on the gateway and Cedar has no `permit` for it.)

### Where the engine runs

The engine is a small pure-Python module with no AWS SDK dependency — a function from (policy files, task) to a decision. It's packaged into the execution-plane worker's **Lambda** container image together with a copy of the policy files, so every task is evaluated in-process before the agent is invoked. Changing policy means merging a PR and rebuilding that image; there's no separate policy service to run.

### Step 3: Fail safe on anything unknown

A repository with **no** policy file is not enrolled, and any write tier gets clamped back to Observe. Enrollment is always a deliberate act; forgetting to write a file makes the system *more* restrictive, never less. A write tier that requires a PR but has no git tool is incoherent, so the engine denies it rather than guessing.

### Step 4: Tell the agent the same rules that gate it

Alongside the decision, the engine emits a **digest**: the agent's hard rules, the repo's conventions, and the compliance rules, as plain text. That gets injected into the agent's context. The policy that *gates* the task is the same policy that *informs* it.

---

## Results & Benchmarks

These are guarantees rather than performance numbers:

* **Default posture for anything unconfigured:** Observe (read-only).
* **Ways for a lower layer to loosen a higher one:** none.
* **Ways for the apply tool to reach an agent:** none — it's denied org-wide.
* **Test coverage:** the suite loads the *real* shipped policy files and evaluates them, so a policy edit that breaks an expectation fails before it merges.

---

## Lessons Learned

* **What went well:** Separating "may the agent act?" from "how should it write code?" and "is the code compliant?" made each piece small and testable. The engine itself has no cloud dependencies at all — it's a pure function from policy files plus a task to a decision.
* **Gotchas to watch out for:** A policy change *is* a guardrail change. Treat it like code — review it and test it. And be suspicious of any merge rule you can't state in one sentence; if two people can read it two ways, someone eventually will.

---

## Conclusion

The policy plane merges organization, agent, repository, and compliance policies into one decision per task, with most-restrictive-wins and fail-safe defaults. Because the policies are files in git, the guardrails themselves travel the same reviewed path as every infrastructure change.

**Call-to-Action:** How do you decide which repositories an automated system may modify — an allow-list, a deny-list, or something layered? I'm curious where the ownership lines fall for others.

*Next up — Part 5: The Execution Plane — stateless workers, a task queue, and why only one component may write state.*
