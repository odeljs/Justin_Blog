---
layout: post
title: Taming-Client-Lambda-Deployments-Part-1
subtitle: Taming Client Lambda Deployments - Why We're Rethinking How We Ship Serverless Code
#cover-img: /assets/img/path.jpg
#thumbnail-img: /assets/img/thumb.png
#share-img: /assets/img/path.jpg
tags: [cloud, aws, lambda, terraform, devops, cicd, serverless]
author: Justin Odel
---


# Taming Client Lambda Deployments

*Part 1 of the series: Building a CI/CD Practice for Client Lambdas*

## Introduction

* **The Hook:** It's the middle of an incident. A client's login is failing, someone patched the auth Lambda directly in the console to stop the bleeding, and the fix held overnight. The next morning a routine `terraform apply` on that account quietly redeploys the old committed zip straight over the hotfix — and the outage is back, except now nobody's expecting it.
* **The Problem:** Our client Lambdas are deployed as hand-built `.zip` files committed into each client's Terraform repo. There's no rollback, no reproducible build, no runtime-upgrade path, no drift detection, and the same code is copy-pasted across a dozen accounts.
* **The Takeaway:** This series walks through turning that into a mature CI/CD practice — one repo, one pipeline, maximum re-use. This first post is an honest look at where we are and why it has to change.

---

## Architecture Overview

* **Current State:** We run infrastructure for a number of clients, each in their own AWS account, each described by its own Terraform. Most accounts contain a small set of Lambda functions. The common case is a pair of Amazon Cognito hooks — pre-authentication and post-authentication — written in Python, talking to Postgres, wired into the client's Cognito user pool. They're small, they're in the critical auth path, and they're duplicated everywhere.

```text
  client-a/                  client-b/                client-n/
     └── <env>/                 └── <env>/               └── ...
         ├── lambda.tf              ├── lambda.tf             (same shape,
         │   ├─ pre_auth  ◄── zip   │   ├─ pre_auth  ◄── zip   copied again)
         │   ├─ post_auth ◄── zip   │   ├─ post_auth ◄── zip
         │   └─ shared layer ◄─zip  │   └─ shared layer ◄─zip
         └── (committed .zip artifacts live in the repo)

   N client accounts  ×  the same hand-built functions  =  N copies to maintain
```

Each function is just another Terraform resource pointing at a committed zip:

```hcl
resource "aws_lambda_function" "cognito_pre_auth" {
  filename      = "${path.module}/lambda/preAuth.zip"
  function_name = "${var.environment_name}-cognito-pre-auth"
  role          = aws_iam_role.cognito_lambda_role.arn
  handler       = "preAuth.lambda_handler"
  runtime       = "python3.12"
  timeout       = 15
  architectures = ["x86_64"]

  layers = [aws_lambda_layer_version.db_driver.arn]
  # ... vpc_config, environment, etc.
}
```

The shared database-driver dependency is a hand-built layer, also committed as a zip:

```hcl
resource "aws_lambda_layer_version" "db_driver" {
  filename                 = "${path.module}/lambda/db_driver_layer_3.12.zip"
  layer_name               = "db-driver"
  compatible_runtimes      = ["python3.12"]
  compatible_architectures = ["x86_64"]
}
```

It works. `terraform apply` creates the function, Cognito gets wired up, auth flows. The problem isn't that it doesn't work — it's what happens over months, across a dozen accounts.

---

## The Root Cause

The underlying issue is that we're treating **build output as source**, and we're letting Terraform own something it can't actually track. A committed `.zip` has no reliable link back to a commit of the source, and without a `source_code_hash` tied to the artifact, Terraform can't tell that a function changed underneath it. So a live, out-of-band fix looks like nothing at all to the next apply:

```bash
# Someone hotfixes the function directly in the console during an incident.
# Later, a routine apply on that account:
$ terraform apply
# ...no diff shown for the function code...
# the old committed zip is redeployed over the live hotfix, silently.
```

Stack that on top of per-account duplication and every weakness gets multiplied by N.

Concretely, here's where it hurts:

1. **There's no rollback.** The previous good version lives wherever the old zip happened to live — an old commit, a laptop, or nowhere. Recovery means rebuilding and re-applying under pressure, in the auth path, which is the worst place to improvise.
2. **Runtime upgrades have no owner.** Everything is pinned to a single Python runtime version. When AWS deprecates it, the upgrade touches the function *and* the hand-built layer, in every account. That tends to happen late, against a deprecation deadline, rather than on our terms.
3. **The code isn't really version-controlled — the artifact is.** You can't cleanly diff two versions or trust that a zip matches any commit. "Deployment" is "commit a binary and apply."
4. **Terraform can silently overwrite a hotfix.** The scenario above — drift you can't see is drift that bites you.
5. **Everything is duplicated.** The pre-auth handler, post-auth handler, and shared driver layer are substantially identical across clients. A single security patch becomes the same change applied by hand N times, with N chances to miss one.

---

## The Solution

This series is the solution, built up piece by piece. The framing is deliberately opinionated:

> **One repo. One pipeline. Maximum re-use.**

### Step 1: One repo for all client Lambdas

Bring the function source out of the scattered per-account Terraform repos into a single mono-repo: shared handlers and libraries, shared layers, and thin per-account configuration. A fix is made once, in one place.

### Step 2: One pipeline, many accounts

A single CI/CD pipeline builds the artifacts once and deploys the right functions into each account, parameterised per account and environment, using cross-account deployment roles.

### Step 3: Promote re-use and draw the Terraform boundary

The pre/post-auth logic and the shared driver layer become reusable building blocks; per-account differences shrink to configuration. And we draw a clear line between what Terraform owns (IAM, triggers, networking, log groups — the *infrastructure*) and what the pipeline owns (the *function code and its versions*). Getting that boundary right is what finally kills the silent-overwrite problem.

---

## Results & Benchmarks

We're at the start, so these are the targets this series is working toward rather than measured numbers yet:

* **Rollback time:** rebuild-and-pray ➔ repoint an alias in **seconds**
* **Places to patch a shared fix:** **N repos** ➔ **1 repo**
* **Runtime upgrade:** per-account manual scramble ➔ **one** deliberate, repeatable step rolled out everywhere
* **Silent drift / overwrites:** possible and invisible ➔ **detected and surfaced**

I'll revisit these with real before/after numbers as the pattern lands.

---

## Lessons Learned

* **What went well:** Writing the current state down made the shape of the problem obvious — the duplication is the multiplier on every other issue, so consolidation is the highest-leverage move.
* **Gotchas to watch out for:** The silent-overwrite trap is the dangerous one because it's invisible until an incident — drift you can't see is drift that bites you.

---

## Conclusion

Our client Lambdas work today, but they're shipped as committed zips with no rollback, no reproducible build, and no drift detection — duplicated across every client account. The fix is a mature CI/CD practice built on one repo, one pipeline, and shared, reusable building blocks, with a clear line between what Terraform owns and what the pipeline owns.

*Next up — Part 2: One Repo, One Pipeline — Designing the Target Architecture.*