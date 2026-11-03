---
title: "AI Infrastructure Agents: The Execution Plane"
date: 2026-10-29
author: Justin Odel
tags: [cloud, aws, ai, agents, architecture, sqs, lambda, eventbridge, dynamodb, agentcore, reliability, security]
description: "Agent workers run model-generated reasoning against untrusted input. Here's why ours are stateless Lambdas behind SQS, structurally unable to write task state — with EventBridge and a single writer persisting to DynamoDB — and the bug a live test caught that unit tests didn't."
series: "AI Infrastructure Agents"
part: 5
---

# The Execution Plane

*Part 5 of the series: AI Infrastructure Agents*

## Introduction

* **The Hook:** The component that runs your agent is the riskiest place in the whole system. It's executing a model's decisions against input you don't control — a compliance finding, a ticket, an alert payload. Now imagine that same component also holds write access to the database that tracks every task's status. A single prompt injection could rewrite history.
* **The Problem:** We needed workers that pick up tasks, run the agent, and report results — reliably, with retries and dead-lettering — without handing the riskiest component any more authority than it strictly needs.
* **The Takeaway:** Our workers are stateless **Lambda** functions triggered by **SQS**, and they **cannot write task state at all**. They publish events to **EventBridge**; a single writer Lambda is the only thing with **DynamoDB** write access. This post covers why, how, and the bug our first live run caught.

---

## Architecture Overview

```text
  SQS task queue ──► worker Lambda (stateless, concurrency-capped)
  (+ DLQ, redrive)        │  1. validate the AgentTask envelope
                          │  2. evaluate the PolicyDecision (Part 4, in-process)
                          │     └─ denied? emit "denied", stop
                          │  3. emit "running"
                          │  4. bedrock-agentcore:InvokeAgentRuntime
                          │  5. emit "succeeded" / "failed"
                          ▼   events:PutEvents
                 EventBridge custom bus
                          │   rule: detail-type = "execution.result"
                          ▼
            writer Lambda (the ONLY state writer)
                          │  validate transition, conditional + idempotent write
                          ▼
   DynamoDB task table (current state)  +  DynamoDB history table (append-only, TTL)
```

Every task carries a common envelope — type, trigger, agent, repository, trace id, payload — so producers and consumers agree on a single format. It's all Terraform: both queues and the redrive policy, the event source mapping, the custom bus and rule, both tables, both Lambdas and their ECR repos, and the IAM.

---

## The Root Cause (the decision that shaped everything)

The real design question was small and easy to get wrong: **who writes task state?**

The simple answer is "the worker updates its own DynamoDB item as it goes." It's strongly consistent and has fewer moving parts. The faithful answer is "workers emit events, and one writer applies them." We weighed them honestly:

| | Worker writes its own state | Workers emit, one writer persists |
|---|---|---|
| DynamoDB write access in the risky component | Yes | **None** |
| Where state-transition rules live | Every worker | **One chokepoint** |
| Event history for observability | Must be added later | **Free — it's the mechanism** |
| Consistency | Strong | Eventual (sub-second) |
| Moving parts | Fewer | One more Lambda + an EventBridge bus |

We chose the single writer. The whole platform is about bounding what the untrusted edge can do — the same instinct that gave us the credential broker in Part 3. Giving the worker DynamoDB write access would have reintroduced "powerful credential in the riskiest place," right after we worked to remove it elsewhere.

And it's enforced by IAM, not convention:

* **Worker role:** `sqs:ReceiveMessage`/`DeleteMessage`/`GetQueueAttributes` on its queue, `events:PutEvents` on the result bus, `bedrock-agentcore:InvokeAgentRuntime` on the runtime. **No** `dynamodb:*`, **no** `secretsmanager:*`.
* **Writer role:** `dynamodb:GetItem`/`PutItem`/`UpdateItem` on the two tables. Nothing else.

---

## The Solution

### Step 1: SQS with a dead-letter queue

Producers send tasks to an SQS queue. An event source mapping triggers the worker with `batch_size = 1`, `ReportBatchItemFailures` (so only failed messages are redriven), and a `maximum_concurrency` cap — that's our backpressure. After `maxReceiveCount` failed receives, a message moves to the DLQ for inspection. Nothing is silently lost.

One constraint matters more than it looks: **the queue's visibility timeout must be at least as long as the worker Lambda's timeout.** Ours is 960 seconds against a 900-second function timeout. Otherwise a task that's still running becomes visible again and gets picked up a second time.

### Step 2: Policy first, then the agent

The worker evaluates the task's policy decision *before* doing anything — the engine and the policy files are baked into the worker's container image. A denial is a correct terminal outcome: emit `denied`, return, no retry. Only an allowed task gets `running` and a call to `InvokeAgentRuntime`, with the decision's timeout enforced against the Lambda's remaining time and its tool posture plus digest passed in the payload.

### Step 3: One writer, one state machine

```text
        ┌──► denied                 (terminal)
  (new) ┤
        └─ queued ─► running ─► succeeded   (terminal)
                        └─────► failed      (terminal)
```

The writer validates every transition and uses a DynamoDB conditional update keyed on the state it just read. A late or duplicate event can't move a finished task backwards, and at-least-once delivery is harmless because repeating a state is a no-op. Every event also lands in an append-only history table (partition `task_id`, sort `event_ts`, TTL-aged) — the stream observability will build on.

### Step 4: Crash recovery comes for free

If a worker dies mid-task, its message reappears after the visibility timeout and another invocation picks it up. Because the dead worker never wrote state, there's nothing to clean up or reconcile.

---

## The bug the live test caught

Unit tests were green. Then we sent real tasks through the deployed plane.

A task against an enrolled repository ran at Tier 2 with git tooling, as expected. A task against an *unenrolled* repository was correctly clamped to Tier 0 — but its decision still listed the git write tool in `allowedTools`. The tier said "read-only"; the tool list said "you can write." Nothing bad happened (the agent at that tier had no git tool registered, and Cedar at the gateway would have refused the call anyway), but the decision object was lying.

The cause was simple: when the engine clamped the *tier* down, it didn't prune the *tools* to match. The fix intersects the tool list with a read-only set whenever the tier lands below Draft, and records the drop in the decision's reason. We rebuilt the worker image, pushed it to ECR, rolled the function with `update-function-code`, and re-ran both tasks live: enrolled kept its git tool at Tier 2, unenrolled dropped it at Tier 0.

```text
Before:  tier 0 (Observe)  allowedTools: [git-operations, terraform-plan]   ← incoherent
After:   tier 0 (Observe)  allowedTools: [terraform-plan]
         reason: "... clamped to Observe; tier < Draft: dropped write tool(s)"
```

---

## Results & Benchmarks

Verified on a live, non-production AWS deployment:

* **End-to-end time:** SQS → policy → AgentCore Runtime → EventBridge → DynamoDB in about **15 seconds**.
* **Policy gating:** enrolled repo ran at **Tier 2**; unenrolled repo was clamped to **Tier 0** with the reason recorded.
* **Worker DynamoDB writes:** **zero** — every state change went through the single writer.
* **History:** each task produced separate, immutable `running` and `succeeded` records.

---

## Lessons Learned

* **What went well:** Choosing the single-writer model up front gave us transition validation, idempotency, and an event history in one place. Making the boundary an IAM fact rather than a coding convention means a future change can't erode it by accident.
* **Gotchas to watch out for:** Keep the SQS visibility timeout above the Lambda timeout, always. Remember that Lambda pins an image *digest*, so pushing a new `:latest` does nothing until you run `update-function-code`. And deploy-and-test early: the clamped-tier bug passed every unit test because each test checked tier *or* tools, never their consistency.

---

## Conclusion

The execution plane runs agents from SQS in stateless Lambda workers that can publish events but never write state, with a single writer enforcing the task state machine in DynamoDB. A live run proved the policy and execution planes work together on AWS — and caught a decision-coherence bug that unit tests had missed.

**Call-to-Action:** When the component running untrusted logic needs to report results, do you let it write directly, or route through something it can't bypass? I'd like to hear how others draw that line.

*Next up — Part 6: PR-First Change Control — the agent finally opens a pull request.*
