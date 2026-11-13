---
title: "Taming Client Lambda Deployments: Designing the Target Architecture"
date: 2026-11-10
author: Justin Odel
tags: [cloud, aws, lambda, terraform, devops, cicd, serverless, codepipeline, s3]
description: "One repo, one pipeline definition, one state file per deployment, and a live alias only Terraform can move. Part 2 walks through the target design for our client Lambdas and the decisions — and rejected options — behind it."
series: "Building a CI/CD Practice for Client Lambdas"
part: 2
---

# Designing the Target Architecture

*Part 2 of the series: Building a CI/CD Practice for Client Lambdas*

## Introduction

* **The Hook:** "One repo, one pipeline" fits on a slide. Then you start asking real questions. Where does the zip live when the function is in another account and another region? What is "the previous version" when a rollback has to restore IAM and environment variables as well as code? How do you stop a deploy when someone changed the function by hand last Tuesday?
* **The Problem:** Part 1 described where we are: committed zips, per-account Terraform, no rollback, invisible drift. Before writing any pipeline code, we needed a design that answers those questions without breaking the deployment tooling every other team already relies on.
* **The Takeaway:** This post is the target architecture and the decision log behind it. It covers a mono-repo with shared source folders, one Terraform project and state file per deployment, a central artifact bucket with direct zip upload, a `live` alias that only Terraform moves, rollback by re-applying the previous deployment, and a drift gate that runs right before approval. I've included what we rejected and why, because that's the part I'd want to read.

---

## Architecture Overview

### What we build on

We already run an in-house pipeline platform. It's a Terraform project that turns one JSON file per pipeline into an AWS CodePipeline V2 pipeline, with CodeBuild stages, a manual approval and SNS notifications. A tag pushed to our self-hosted Git server (Gitea) hits a webhook, and the webhook starts the matching pipeline. There's also an existing PR check that runs `terraform plan` on pull requests and posts the result as a comment.

We decided early not to modify any of that. Everything for client Lambdas is added as new files alongside it.

### The target

```text
  Gitea: client-lambdas mono-repo
  ├── functions/<source>/src/        function code, one folder per distinct version
  ├── functions/<source>/function.yaml   runtime, handler, layers
  ├── layers/<layer>/build.yaml      pinned dependency layer
  └── deployments/<client>/<env>/<function>/
        main.tf  backend.tf  terraform.tfvars.json
                    │
        tag:  <function>-<env>-v1.2.3
                    ▼
  ┌───────────────────────── CI account ─────────────────────────┐
  │ CodePipeline (one per deployment, one shared definition)     │
  │  Build ─► Snapshot ─► Plan ─► Lambda checks ─► Approval      │
  │        ─► Apply ─► Verify ─► Record                          │
  │                                                              │
  │  S3: central artifact bucket   <source>/<commit>.zip         │
  │      deployment records        _records/<deployment>/...     │
  │  S3: state bucket              one state file per deployment │
  └──────────────────────────┬───────────────────────────────────┘
                             │ assume deploy role (or fail)
                             ▼
  ┌──────────── target account / region ────────────┐
  │ Lambda function  ──  versions 1..n  ──  "live"  │
  │ IAM role, log group, alias-scoped permissions,  │
  │ error and timeout alarms                         │
  └──────────────────────────────────────────────────┘
```

---

## The Root Cause (what the design has to fix)

Part 1's problems all come back to three missing things:

1. **No link between source and what's running.** A committed zip isn't tied to any commit.
2. **No history.** Nothing records what was deployed, when, or from where. That means no baseline to compare against and no reliable target for a rollback.
3. **No owner for drift.** Terraform can't see a console edit to code it doesn't track, and nothing else looks.

Every decision below exists to close one of those gaps.

---

## The Solution

### Step 1: One repo, shared source folders

All client Lambda code moves into one repo. The unit of sharing is the **source folder**. By default there's one folder per function, and every client running identical code points at it. Each deployment says which folder it builds from:

```json
{
  "function_name": "cognito-pre-auth",
  "source_folder": "cognito-pre-auth",
  "memory_size": 128,
  "timeout": 10,
  "environment_variables": { "DEPLOY_ENV": "uat" },
  "log_retention_days": 14
}
```

When one client's deployed code genuinely differs, it gets a **variant** folder, imported exactly as deployed. Its `function.yaml` records `variant_of` and a `reason`. When a variant matches its parent again, it's merged back. We deliberately don't extract a `common/` library. That would mean rewriting function code, and that isn't our call to make. Third-party dependencies live in pinned layers, so nothing installed hides inside a function zip.

Settings live in `terraform.tfvars.json` rather than HCL. That's a small choice with a real reason behind it: a script can edit JSON reliably, which matters once the pipeline has to write a hand edit back into the repo.

### Step 2: One Terraform project and one state file per deployment

Each `deployments/<client>/<env>/<function>/` folder is its own Terraform project. It has its own state file in a central state bucket, keyed by deployment and using S3 native locking (`use_lockfile`). Shared code doesn't mean shared state. Two clients can build from the same source folder, but a deploy or rollback for one can never touch the other.

### Step 3: Build once, store centrally, upload directly

The zip is built **once per commit** and stored as `<source>/<commit>.zip`. The first pipeline to need it builds it. Every other deployment of that folder at that commit reuses the same object. The upload is create-only, using the S3 `If-None-Match` conditional write, so two pipelines racing can't replace each other's artifact.

The first draft had **one artifact bucket per region**, because Lambda can only read code from a bucket in its own region. That meant a bucket, a KMS key, and cross-account bucket and key policies in every region we deploy to. We dropped it for a simpler design. The deploy stage downloads the zip from one central bucket, and Terraform uploads it **directly in the Lambda API call** (`filename`). The client account never reads our bucket at all. The cost is the 50 MB zipped limit for direct upload, which is comfortable because dependencies live in layers.

### Step 4: Terraform never builds code

This one took a few iterations. Our CI account's own Lambdas zip their source at plan time with Terraform's `archive_file`. That's convenient, but it means every plan rebuilds the code. Build-once, the drift baseline and rollback would then all depend on every rebuild being byte-identical.

So the rule became: **the pipeline builds, Terraform deploys exactly what it's handed.** That needs our own Lambda plan and apply buildspecs, because the shared ones can't supply a pre-built zip. Client-lambdas pipelines opt out of the existing PR plan check through a flag it already supports. A new Lambda PR check runs the plan itself.

### Step 5: Versions and a `live` alias that only Terraform moves

Every apply publishes a new immutable version (`publish = true`). In the same apply, Terraform moves a `live` alias to that version. Callers, such as a Cognito trigger, invoke the alias, never `$LATEST`. There's no `ignore_changes` on the alias and no script that flips it. If anything else moves it, that's drift, and drift gets caught (Step 7).

### Step 6: Rollback re-applies the previous deployment

The obvious rollback is "repoint the alias at the previous version". It restores code and version-level settings. It doesn't restore IAM policies, invoke permissions, alarms, or anything else the newer deploy changed.

So a rollback **re-applies the previous recorded deployment**: that commit's Terraform, with that commit's existing zip. Everything goes back, and a new version is published with the old code and config. Repointing the alias by hand at a pre-deploy snapshot version is kept as the emergency option, for when the old commit no longer applies cleanly.

### Step 7: Deployment records and a drift gate

After every deploy, the pipeline writes a **deployment record**: the commit, the artifact key and version, the `$LATEST` hash, the version `live` points at, and that version's hash. It records what Lambda itself reports, so comparisons are like for like. Records are create-only JSON objects in a bucket that denies deletes, so the history can't be rewritten. We rejected two other places for them:
- **Function tags** sit in the client account, where an admin could change the very baseline we trust.
- **SSM parameters** have size limits and a capped history.

The latest record is the drift gate's baseline. The one before it is the default rollback target.

The **drift gate** runs as its own stage, right before approval, because a PR can be approved days before the deploy happens:

* It compares Lambda's `$LATEST` hash, the `live` version and the `live` hash with the latest record. That catches console code edits and a hand-moved alias.
* It runs `terraform plan -refresh-only`. That catches edits the hashes can't see: memory, environment variables, IAM, permissions, alarms.
* It builds a side-by-side HTML diff (Python's `difflib.HtmlDiff`) of the code `live` serves against what this deploy will ship. On drift, it adds the hand edit against the last recorded zip. The approval links to it.

Any drift stops the pipeline. Then a person decides:
- **Discard:** let this deploy replace the hand edit. The decision is tied to the exact drifted hash, so a second edit fails again.
- **Adopt:** bring the hand edit into the repo first and deploy it, so it becomes a recorded deployment that a later rollback can restore.

### Step 8: Keep the platform untouched

* Pipeline buildspecs live in the pipeline platform's repo, not in the client-lambdas repo. A rollback checks out an old commit, and we don't want it running an old commit's *scripts*.
* There's no PR template. The change record is what the Git server already holds: the commits and diff, the plan, the Lambda check report, the review and the approval.
* Every client-lambdas pipeline is a new JSON file in the platform. No shared module or shared buildspec changes.

### Step 9: Prove it on something nobody depends on

The first real function to move in was going to be a client's pre-auth Lambda in a UAT account. We changed that. Every use case gets proven first on **`cicd-test`**, a tiny function that isn't client code:
- first deploy
- code change and config change
- layer upgrade
- console edit with discard, and with adopt
- variants
- every rollback path
- failed assume-role

It reports its code marker, version and layer on every call, and fails on demand.

---

## Results & Benchmarks

Still design-stage numbers, but they're specific now:

* **Artifact buckets:** one per region with cross-account policies ➔ **one** central bucket, no cross-account access
* **Things Terraform rebuilds per plan:** the function zip ➔ **nothing**; it deploys the pipeline's build
* **What a rollback restores:** code only (alias repoint) ➔ **code, settings, IAM, permissions and alarms**
* **When drift is checked:** never ➔ on the PR **and** again right before approval
* **Changes to the existing pipeline platform:** **zero** modified files; everything is additive

---

## Lessons Learned

* **What went well:** Writing a decision log, including the rejected options, paid for itself fast. Several decisions got revised, among them the artifact buckets, rollback, and who builds the zip. Each revision was a short conversation instead of a rediscovery.
* **Gotchas to watch out for:**
  * "Roll back by repointing the alias" sounds complete but only covers half the state. Decide what a rollback must restore before you design it.
  * Shared source folders have a sharp edge we haven't fully solved. A shared-code change merged for one client can ride along with an unrelated deploy for another. That's on our known-issues list, to be settled during testing.

---

## Conclusion

The target is one repo with shared source folders and one Terraform project and state per deployment. The zip is built once and uploaded directly, versions sit behind a `live` alias only Terraform moves, rollbacks re-apply the previous recorded deployment, and a drift gate runs right before approval. All of it is added alongside the existing pipeline platform without changing it.

**Call-to-Action:** If you've built rollback for Lambdas, what did "previous version" mean for you: just the code, or the whole deployment? I went back and forth on this more than anything else.

*Next up — Part 3: First Light — building the storage, the module and the pipeline, and the bugs our first real deploys found.*
