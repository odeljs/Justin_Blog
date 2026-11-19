---
title: "AI Infrastructure Agents: Deploying the Validator and Trying to Break It"
date: 2026-11-18
author: Justin Odel
tags: [cloud, aws, ai, agents, terraform, codebuild, lambda, eventbridge, security, testing, change-control]
description: "We deployed the PR validation pipeline and fed it deliberately bad pull requests: unformatted code, a data source that runs programs, a missing tag and racing pushes. It caught all of them, and the first live runs found three bugs that 224 unit tests had not."
series: "AI Infrastructure Agents"
part: 7
---

# Deploying the Validator and Trying to Break It

*Part 7 of the series: AI Infrastructure Agents*

## Introduction

* **The Hook:** A validation pipeline that has only ever seen unit tests is a hypothesis. The first time a real webhook arrives, every assumption about tokens, payloads, caches and timing gets tested at once.
* **The Problem:** In Part 6 we built a pipeline that judges agent pull requests, with a bot review as the merge gate. Now it had to run for real, against a real Git server, and survive pull requests designed to fool it.
* **The Takeaway:** We deployed it, opened a series of hostile and awkward PRs, and watched what happened. It caught every bad PR. The first live runs also found three bugs that the unit tests had not, and each took minutes to fix once seen.

---

## Architecture Overview

```text
  push to a PR ──► webhook ──► API Gateway ──► webhook Lambda
                                                 │  verify HMAC, filter, stop older builds
                                                 ▼
                                           CodeBuild runner   (one build per commit)
                                                 │  result JSON
                                                 ▼
                                                S3
                                                 │  build state-change event
                                                 ▼
                                           reporter Lambda ──► comment · statuses · bot review
```

There is no CodePipeline. Each commit needs its own one-off run with different inputs and no deploy stage, so the webhook Lambda calls `StartBuild` on a single CodeBuild project and passes the PR's details as build variables. When the build finishes, CodeBuild emits a state-change event, and an EventBridge rule hands it to the reporter.

Where a shared, versioned module already existed in our internal registry (for the Lambda functions, the image repository and the bucket), we used it. CodeBuild, API Gateway and EventBridge had none, so those are plain resources.

---

## The Test Plan: Pull Requests Built to Fail

We opened every test PR as the agent would: through the credential broker from Part 3, with a short-lived token that was revoked straight after.

| Scenario | What should happen |
|---|---|
| A clean agent PR with a trace ID | Approved, trace ID carried through |
| An unformatted file | `fmt` fails, changes requested |
| A `data "external"` block | Refused before Terraform runs anything |
| Two pushes eight seconds apart | Only the newer commit is judged |
| A change missing a required tag | Plan runs, `policy` fails |
| A change that adds the tag | `policy` passes |

---

## Results

Every scenario behaved as intended:

| Scenario | Result | Bot review |
|---|---|---|
| Clean agent PR | All checks pass, trace ID in the build, comment and event | Approved |
| Unformatted file | `fmt` fails, the other checks pass | Changes requested |
| `data "external"` | Refused before `init`: the provider "runs arbitrary programs and is not allowed" | Changes requested |
| Two quick pushes | First build stopped as superseded; the reporter skipped its result as stale | Approved on the second commit only |
| Missing tag | Plan with the read-only role; `policy` reports the resource and the missing key | Changes requested |
| Tag added | `policy` passes; the comment is edited in place and the same-commit review is not repeated | Approved |

* **Latency:** about **30 seconds** from push to review: roughly 10 s for CodeBuild to provision and 8 to 9 s for the runner.
* **The gate counts:** the Git server marks every bot review as official, so the approval is what branch protection will rely on.
* **The plan-time guard held:** in the `data "external"` case the external provider was never downloaded and its program never ran.

---

## The Three Bugs the Unit Tests Missed

### Bug 1: the bot could not ask who it was

The reporter only edits a comment if the bot wrote it, so it asked the Git server "who am I?" first. The live call failed with a 403: that endpoint needs a `read:user` scope, and we had deliberately given the write token only the scopes it needed for comments, statuses and reviews. The unit tests' fake server answered the call happily.

The fix was to make the bot's username configuration instead of asking, so the token kept its narrow scopes. The fake server now returns the same 403 the real one did.

### Bug 2: a warm Lambda remembered the old config

For the tagging test we turned on a required tag and applied the change. The next PR was still judged with no required tags. The repository config was cached in memory for the life of the Lambda container, and warm containers can live for hours.

The config is now read on every event. Secrets are re-read every five minutes, so a rotation takes effect without a deploy.

### Bug 3: Terraform could not count before apply

The first plan failed with "Invalid count argument". The shared Lambda module creates its inline policy only when the policy input is not null, and our policies referenced ARNs that are only known after apply. Building those ARNs from resource names fixed it. A plan-time check fails the plan if the bucket name the shared module builds ever differs from the one we assume.

---

## Lessons Learned

* **What went well:** Testing with deliberately bad PRs, not just a happy path, showed each defence working in context: the plan-time guard, the superseded-commit handling and the stale check before the review. Every test PR went through the real webhook path, so delivery and signatures were proven by real events.
* **Gotchas to watch out for:**
  * Least-privilege tokens break calls you did not know needed a scope, and fakes will not tell you.
  * Anything cached in a warm Lambda outlives your deploys. Decide on purpose what may be cached and for how long.
  * Shared modules that pick `count` from optional inputs need those inputs known at plan time.

---

## Conclusion

The validator is deployed and catches bad agent PRs in about 30 seconds, with a bot review the agent cannot forge as the gate. The first live hour found three bugs that 224 unit tests had not, which is a good argument for getting to a real deployment early and attacking it on purpose.

**Call-to-Action:** What is the first PR you would write to try to fool a validation pipeline like this? I would like to hear the ones I have not thought of.

*Next up — Part 8: closing the same plan-time hole in the older plan pipeline, then letting the agent fix its own failed PRs.*
