---
title: "AI Infrastructure Agents: How the Agent Opens a Pull Request"
date: 2026-11-19
author: Justin Odel
tags: [cloud, aws, ai, agents, agentcore, terraform, gitops, security, credentials, change-control]
description: "A step back to the change that Part 6 assumed: how the agent turns a finding into a pull request with a git token that lives for five seconds, why the PR's policy decision is read from the platform rather than the agent, and the refusal path for repositories it may not touch."
series: "AI Infrastructure Agents"
part: 6.5
---

# How the Agent Opens a Pull Request

*Part 6.5 of the series: AI Infrastructure Agents*

## Introduction

* **The Hook:** Part 6 began with "the agent opens a pull request" and moved straight on to judging it. That sentence hides the most sensitive moment in the whole system: the first time a model gets write access to a repository.
* **The Problem:** The agent has to push a branch and open a PR. That requires a git credential, and anything the agent writes into the PR, including the description reviewers trust, could be wrong or manipulated.
* **The Takeaway:** The agent never holds the credential or writes the facts reviewers rely on. A single tool does the git work with a token that exists for about five seconds, and the policy decision shown on the PR is read from the platform, not taken from the agent.

---

## Architecture Overview

```text
 execution worker (Part 5)
   └─ AgentCore Runtime: the agent reasons about the finding
        ├─ terraform-plan tool   read-only, check current state
        └─ git-operations tool   opens the PR            (Cedar: Draft tier and above)
              0. read the task's policy decision and trace ID from the task table
              1. ask the credential broker for a token: one repository, short TTL   (Part 3)
              2. clone, branch agent/<finding-id>, write the change, commit, push
              3. open the PR with a templated body
              4. revoke the token, in a finally block
              └─► { branch, prId, prUrl, commit, traceId, taskId }   → validation (Part 6)
```

The golden rule stays intact: **agent writes code → agent opens a PR → CI validates → a human reviews → CI applies.** No apply tool exists, and the agent has no merge power.

---

## Decision 1: Git Work Belongs in a Tool, Not in the Worker

We could have done the git work inline in the execution worker. We made it a gateway tool instead, the same way the read-only `terraform-plan` tool works:

| | Git work in the worker | Git work as a gateway tool |
|---|---|---|
| Write power in the component that runs every task | Yes | **No** — the worker stays a thin dispatcher |
| Policy check at the moment of use | Only the up-front decision | **Cedar checks every call** at the gateway |
| Where git credentials are handled | Mixed into task dispatch | **One small, auditable function** |
| Fit with the existing tool model | New pattern | **Same as the plan tool** |

The tool also checks the tier itself before doing anything, so there are three independent gates: the policy decision lists the tool only at the Draft tier, Cedar permits it at the gateway, and the tool refuses unless the recorded decision allows a PR.

---

## Decision 2: The Facts on the PR Come from the Platform

The PR body tells a reviewer why the change exists: the finding, the originating signal, the policy decision that allowed it, and a trace ID. Our first live run printed "(none provided)" where the policy decision should have been, because the tool expected the agent to pass it in.

The obvious fix was to make the agent pass it. We did the opposite. The tool now reads the decision and trace ID from the platform's own task record, using the task ID. The agent supplies only the repository, the file changes and the finding. A confused or prompt-injected agent can write a misleading summary, but it cannot forge the decision a reviewer sees.

---

## Decision 3: A Token That Lives for Five Seconds

The broker from Part 3 already issued short-lived, single-repository tokens with a capped TTL, and a scheduled reaper deleted them after expiry. That bounded the risk to about fifteen minutes. We then added a revoke operation, and the tool calls it in a `finally` block as soon as the PR call returns.

From the audit trail of a live run:

```text
issued    18:46:44   TTL would have expired 19:01:43
revoked   18:46:49   "revoked at task end on caller request"
```

Five seconds instead of fifteen minutes. Revocation is best effort: a failed revoke never hides the PR result, and the reaper remains the backstop. A direct check confirmed the behaviour: a fresh token authenticated (HTTP 200) before the revoke and was rejected (HTTP 401) straight after, a second revoke reported `already_revoked`, and an unknown request ID was refused.

---

## The First Real Change

For the first live demonstration we chose a finding that is safe, common and easy to review: **a resource is missing a required tag.** The agent read the finding, authored the one-line Terraform fix and called the tool. The PR arrived with the summary, the originating signal, the governing decision and the trace ID in its body, on a branch named after the finding.

We also tested the refusal path. For a repository that was not enrolled, the policy plane clamped the task to the Observe tier and stripped the git tool. The agent still proposed the fix in its output, and then correctly declined to open a PR.

---

## Results

* **End to end:** a queued finding became a real PR through the worker, the policy decision, the agent and the tool.
* **Token lifetime:** about **5 seconds** in practice, against a 900-second cap.
* **Gates:** three independent checks (decision, Cedar, the tool's own check) before any git write.
* **Refusal:** an unenrolled repository got a proposal, not a PR.

---

## Lessons Learned

* **What went well:** Reusing the tool pattern from the read-only plan tool meant no new concepts for policy or review. Reading the decision server-side turned a cosmetic bug into a real security property.
* **Gotchas to watch out for:**
  * Live testing found that the agent cannot read repository files, so the demo passed the current file in the task. Real findings will need a read-only file tool.
  * The tool does not yet check that the requested repository matches the task's repository.
  * The broker's audit records its own identity instead of the caller's, so it cannot yet say who asked.
  * Policy files are baked into the worker image, so a policy change only takes effect once that image is rebuilt from the right branch.

---

## Conclusion

The agent gets write access to a repository for about five seconds, through one tool, behind three gates, and the facts on the PR come from the platform rather than from the model. That is what lets the validation pipeline in Part 6 treat the PR as just another change to judge.

**Call-to-Action:** Where does your agent's PR description come from, and could the agent make it say something untrue? I would like to hear how others handle provenance for agent-written changes.

*Next up — Part 7: deploying the validator and trying to break it with deliberately bad PRs.*
