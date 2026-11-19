---
title: "AI Infrastructure Agents: Validating the Pull Request"
date: 2026-11-17
author: Justin Odel
tags: [cloud, aws, ai, agents, terraform, security, codebuild, opa, conftest, change-control]
description: "The agent now opens pull requests, so something has to judge them. We built a separate validation pipeline with a bot-review merge gate and an OPA tagging policy, then found that terraform plan itself runs the PR's code, and closed that hole before it shipped."
series: "AI Infrastructure Agents"
part: 6
---

# Validating the Pull Request

*Part 6 of the series: AI Infrastructure Agents*

## Introduction

* **The Hook:** The agent opens a pull request. It looks fine. The title is sensible, the diff is small, and a status check says green. Who set that status, and could the agent have set it too?
* **The Problem:** Change control only works if the check that approves a change is out of reach of whatever made the change. We needed fmt, validate, lint and a tagging policy on every agent PR, a merge gate the agent cannot forge, and a way to run Terraform on untrusted code without handing that code our credentials.
* **The Takeaway:** We built a separate validation pipeline. Along the way we found that `terraform plan` runs code from the PR, which breaks the obvious isolation story. This post covers the design, the policy engine comparison, and the guard that closed the gap.

---

## Architecture Overview

```text
  Gitea webhook (PR opened / updated, base = main)
        │
        ▼
  API Gateway ──► webhook Lambda   verify HMAC signature, filter, start build
                        │
                        ▼
                  CodeBuild runner   clone (read-only token), then per changed directory:
                        │              fmt · validate · tflint · plan (read-only role) · Conftest
                        │  result JSON
                        ▼
                       S3
                        │  build state-change event
                        ▼
                  reporter Lambda    write token (shared only with the webhook Lambda)
                        │              comment · statuses · bot review (approve / request changes)
                        ▼
                  Gitea PR  ◄── branch protection requires the validator bot's approval
```

Two rules shaped the layout. The build, which runs the PR's code, holds only a read-only clone token. The write token lives only in the two Lambdas, which never run PR code: the webhook Lambda uses it to mark the commit as pending, and the reporter uses it to post results. The existing PR plan flow is untouched; this is a separate pipeline with its own bot, secrets and webhook.

---

## Decisions

### The gate is a review, not a status

Commit statuses are convenient, but the agent's own token can post them. We tested it: the agent's broker-issued token could create a commit status (HTTP 201) while being unable to read or write PR comments. A status alone cannot be a gate, because the thing being checked could set it.

So the merge gate is a **review from a dedicated validator bot**: approved or changes requested. Statuses and the comment are shown for people, but branch protection requires the bot's approval, and approvals are dismissed when new commits arrive.

### Judge the plan, not the source

For the tagging policy we did not assume a tool. We wrote eight test cases and ran three approaches:

| Approach | Correct | Notes |
|---|---|---|
| Conftest (OPA) on the plan JSON | 8 / 8 | A 15-line rule, about 0.05 s |
| Checkov on source | 7 / 8 | Misses tags that come from provider `default_tags`; custom check is more code and slower |
| Conftest on source | 5 / 8 | The HCL parser does not resolve variables, `merge()` or tfvars |

Tags are only known after Terraform resolves variables, tfvars and provider defaults, so the plan is the right thing to judge. The rule reads the required tag list as data, so when the organisation settles its tagging policy it is a YAML change, not a code change.

The plan runs against an empty local backend, so state is never read, with tfvars fetched from SSM and a read-only role per environment. Only the directories a PR changes are validated, which works for both single-client repositories and monorepos.

### Fail closed

If any check cannot be judged, the result is `error`, and `error` blocks. A PR that touches Terraform outside every configured environment is not approved. A build that crashes produces a synthesized result with every check in error, because an empty result would otherwise approve. A docs-only PR is approved, since there is nothing to judge.

---

## The Root Cause (of the gap we nearly shipped)

The runner was already careful. The clone token is passed to git in a scoped header and never reaches the tools that run PR code. Terraform gets short-lived credentials for a role with almost no permissions. We were happy with it.

Then we asked what a hostile PR could do during `plan`. We tested instead of guessing:

```text
data "external" "x" {
  program = ["touch", "/tmp/marker"]
}
```

After `terraform plan`, the marker file existed. Three PR-controlled things execute during validate or plan:

* `data "external"` runs any program.
* A provider outside the HashiCorp namespace is a binary the PR picks.
* A provider binary committed under `terraform.d/plugins` is used as is.

That code runs in the same container and as the same user as the runner. It could read the build's credentials and write a forged "all checks passed" result to the results bucket. Container isolation does not fix this, because what is being tricked is already inside the container.

---

## The Solution

### Step 1: Refuse before running anything

A pre-flight guard decides whether the code may run at all:

1. A committed `terraform.d/` or `.terraform/` directory is refused.
2. `terraform providers` lists every provider the configuration needs. We run it before `init`, so a disallowed provider is never downloaded, and again after `init`, when remote modules are fetched and the list is complete.
3. Every provider must come from `registry.terraform.io` in an allowed namespace. The default is `hashicorp`, configurable per repository, and `hashicorp/external` is always denied.

We let Terraform's parser decide instead of searching the source for strings. HCL accepts unquoted block labels (`data external x {}`) and a JSON syntax (`.tf.json`), and a regex would miss both. We verified that `terraform providers` reports the provider in all three forms, before `init`.

A refused PR fails `validate` with the reason and sets `policy` to error. No credentials are assumed, and nothing is initialised or planned.

### Step 2: Shrink what is left to forge

* The build role can write to the results bucket only under one prefix.
* The reporter checks the result's shape and size, and that the repository, PR number and commit match the build it came from, before it posts anything.
* The reporter skips results for a commit that is no longer the PR's head.

### Step 3: Prove the guard fails when it is off

The guard test writes PRs whose code would `touch` a marker file, once for each syntax (HCL, unquoted labels and JSON). It asserts that the marker never appears and the gate does not approve. Then we disabled the guard and reran it: the marker appeared and the test failed. A security test you have never seen fail proves little.

---

## Results

* **Policy engine:** 8 of 8 cases correct on the plan JSON, against 7 and 5 for the alternatives.
* **Runner tests:** 173 unit tests pass in about three seconds. Opt-in end-to-end tests run real `terraform`, `conftest` and `tflint` on fixture repositories.
* **Guard:** `data "external"` did not run in any of three syntaxes. With the guard off, it ran.

The webhook, reporter and Terraform module are still to come. Numbers from a live deployment will go in the next post.

---

## Lessons Learned

* **What went well:** Comparing engines on real cases took an afternoon and changed the answer. Having Terraform itself list providers avoided a fragile text scan.
* **Gotchas to watch out for:** Treat `terraform plan` on untrusted changes as code execution, not as a read. Isolating the container does not help when the untrusted code runs inside it. Do not make a gate out of anything the checked party can write, and test that your security tests can fail.

---

## Conclusion

Agent PRs are now judged by a pipeline the agent cannot write to: a read-only build, a separate reporter, and a bot review as the gate. The most useful finding was that planning Terraform is running someone else's program, so the pipeline refuses the constructs that make that dangerous before it runs anything.

**Call-to-Action:** If you run `terraform plan` on pull requests, what can the code in that PR reach from the runner? I would like to hear how others handle it.

*Next up — Part 6.5: a step back to how the agent opens the pull request in the first place.*
