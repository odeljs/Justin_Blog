---
title: "Taming Client Lambda Deployments: First Light — Building the Pipeline and What It Broke"
date: 2026-11-12
author: Justin Odel
tags: [cloud, aws, lambda, terraform, devops, cicd, serverless, codepipeline, codebuild, s3, kms]
description: "Part 3: the S3 and KMS storage, a Lambda Terraform module, and a seven-stage CodePipeline built on CodeBuild — plus four real bugs our first deploys of a test function found, including a stale code hash and a refresh-only exit code that cried wolf."
series: "Building a CI/CD Practice for Client Lambdas"
part: 3
---

# First Light: Building the Pipeline and What It Broke

*Part 3 of the series: Building a CI/CD Practice for Client Lambdas*

## Introduction

* **The Hook:** The pipeline went green on its first deploy. The second deploy, a one-line code change, failed at Apply with "CodeSha256 differs from the uploaded zip". Lambda was serving exactly the right code. Our check was reading a number Terraform hadn't bothered to update.
* **The Problem:** Part 2 was the design. This part is the build: storage, a reusable Lambda module, and a seven-stage pipeline. Then the first real runs, against a test function in our CI account, to find out where the design and reality disagree.
* **The Takeaway:**
  * the concrete AWS resources and the stage-by-stage buildspec logic
  * how build-once and "approve exactly what you deploy" are enforced
  * four bugs the first runs found, each of which would have hurt badly on a real client function

---

## Architecture Overview

Everything below runs in the CI account, in one region. So does the test function. Cross-account deploys come later, once every use case passes here.

```text
 Gitea tag  cicd-test-a-dev-v1.0.1
      │  webhook → StartPipelineExecution (TAG_NAME, VERSION)
      ▼
 CodePipeline V2 (QUEUED) ── CodeBuild ARM, one project per stage
  1 Build         zip functions/<source>/src once → S3 create-only
  2 Snapshot      publish-version on the current function (restore point)
  3 Plan          download the exact zip by key+version → terraform plan
  3b LambdaChecks drift gate + side-by-side code diff report
  4 Approval      manual, links to the plan and the report
  5 Apply         same zip again → terraform apply (new version, move live)
  6 Record        read Lambda, write a create-only deployment record
      │
      ▼
 S3  artifact bucket (SSE-KMS, versioned, deletes denied, TLS only)
     ├── cicd-test/<commit>.zip
     └── _records/<deployment>/<timestamp>-<execution>.json
 S3  state bucket (SSE-KMS, versioned, TLS only) one key per deployment
```

---

## The Root Cause (why each stage looks the way it does)

The whole build serves one invariant: **the zip that was built, planned and approved is the zip that gets deployed and recorded.** Each stage passes a small JSON file forward through the run's folder in the pipeline artifact bucket, and each stage checks it:

```json
{
  "bucket": "<artifact-bucket>",
  "key": "cicd-test/0f89609b….zip",
  "version_id": "ueBL__Rh…",
  "sha256": "otbmmVEX…",
  "commit": "0f89609b…",
  "source_folder": "cicd-test",
  "tag": "cicd-test-a-dev-v1.0.1"
}
```

Plan and Apply download that exact **S3 object version** and recompute its SHA-256 before Terraform sees it. Plan also checks that the commit matches the checked-out tag.

---

## The Solution

### Step 1: Storage: two buckets, two keys

Both buckets use our registry's S3 module, which covers versioning, SSE-KMS, bucket-owner-enforced ownership and the public-access block. They're applied with a targeted plan and apply against the shared root they live in, so nothing else in that root is touched:

* **Artifact bucket:** its own customer-managed KMS key with rotation on, plus a bucket policy that allows TLS only and **denies `DeleteObject` and `DeleteObjectVersion` for everyone**. Every zip a rollback might need, and every deployment record, is permanent unless the policy itself is changed through review.
* **State bucket:** its own KMS key and TLS only. State keys follow `client-lambdas/<client>/<env>/<function>/terraform.tfstate`, with `use_lockfile`.

No cross-account statements anywhere, because the zip is uploaded directly to Lambda.

I nearly shipped an explicit key policy granting the account root `kms:*`. It turned out to be exactly the AWS default, so it added nothing and looked like more control than it was. It's gone. A tightened policy comes later, once the pipeline has roles worth naming.

### Step 2: A Lambda module that deploys what it's given

The new registry module creates:
- the function, with `publish = true`
- the `live` alias, following `aws_lambda_function.this.version`
- the execution role and log group
- invoke permissions qualified to the alias
- alarms

The code input is just `filename`. The pipeline puts the zip there:

```hcl
filename         = var.filename
source_code_hash = coalesce(var.source_code_hash, filebase64sha256(var.filename))
```

One detail worth sharing: our existing Lambda alarm module filters the `Errors` metric on both `FunctionName` and `Resource = <function name>`. The `Resource` dimension only matches unqualified invocations, so it **never sees calls made through an alias**. The new module alarms on `FunctionName` alone, which aggregates every version and alias.

The deployment's `main.tf` reads runtime, handler and architecture from the source folder's `function.yaml`. Every deployment of a folder then runs the same code the same way:

```hcl
locals {
  function_yaml = yamldecode(file("${path.module}/../../../../functions/${var.source_folder}/function.yaml"))
}
```

### Step 3: Build: sorted, bare, create-only

```bash
( cd "$SRC_DIR" && find . -type f ! -path '*/__pycache__/*' ! -name '*.pyc' \
    | LC_ALL=C sort | zip -q -X -@ /tmp/function.zip )
[ "$(stat -c %s /tmp/function.zip)" -le 52428800 ] || exit 1   # direct-upload limit

aws s3api put-object --bucket "$BUCKET" --key "$SOURCE/$COMMIT.zip" \
  --body /tmp/function.zip --if-none-match '*'
```

* The sorted file list and `zip -X` (no extra attributes) keep the archive to exactly the committed files.
* If the object already exists, the build reuses it. If another pipeline wins the race, the conditional put returns `PreconditionFailed` and we use the winner's object.
* The build image's AWS CLI is checked for `--if-none-match` and upgraded if it's missing.

### Step 4: Credentials: explicit, never a silent fallback

Our older shared buildspecs try to assume the target role and, if that fails, carry on with the pipeline's own credentials. That's how a typo in an account ID becomes a deploy into the wrong account. The new buildspecs make the choice explicit:

```bash
if [ -z "$TARGET_ACCOUNT_ID" ] || [ "$TARGET_ACCOUNT_ID" = "$CURRENT_ACCOUNT" ]; then
  echo "Target is this account: using the pipeline's own credentials."
else
  CREDS=$(aws sts assume-role ...) || { echo "ERROR: could not assume role"; exit 1; }
  # credentials go to a named profile for the AWS provider only;
  # the S3 state backend keeps the pipeline role
fi
```

### Step 5: Snapshot and Record

* **Snapshot** runs `publish-version` before anything changes. If `$LATEST` is unchanged since the last published version, Lambda returns that existing version, which is the restore point anyway. On a first deploy there's no function, so it records `exists: false`.
* **Record** reads `$LATEST`, the `live` version and that version's hash from Lambda. It writes them with the commit, tag and artifact as a create-only JSON object under `_records/<deployment>/`.

### Step 6: The first runs, and the four bugs

**Bug 1: the pipeline couldn't receive a tag.** The plan for the new pipeline showed stages receiving only an execution ID. Our generic pipeline module declares only the variables listed in the config, and the tag webhook starts pipelines with `TAG_NAME` and `VERSION`. The start would have been rejected, and the stages would have cloned nothing. We caught it in the plan review, before the first run, and declared both, with `SET_ME` defaults so a careless manual start fails loudly.

**Bug 2: a stale hash failed a correct deploy.** The second deploy changed one line. Apply ended by comparing Terraform's `code_sha256` output with the uploaded zip, and they differed. Lambda was serving the right code. With the AWS provider version we use, an in-place code update left `code_sha256` in state at the *previous* value. A later plan showed it as an output-only change, "without changing any real infrastructure".

The real damage was the knock-on: Apply failed, so **Record never ran**, and the function had changed with nothing recorded. The next drift check would have flagged our own deploy as a hand edit. The fix:
- Apply now fails only if Terraform fails, and syncs state with a refresh-only apply afterwards.
- Record **always** writes what Lambda actually reports, with a `matches_artifact` flag, and only *then* fails if the flag is false. The baseline always describes what's live, even when something went wrong.

**Bug 3: the drift gate cried wolf.** The first LambdaChecks run failed on an untouched function. The code hashes matched the record, the alias matched, and the refresh-only plan said "No changes". But `terraform plan -refresh-only -detailed-exitcode` returned **2**. Reproduced locally on Terraform 1.11.2 against the same state: exit 2, `resource_drift` empty, every output a no-op. The gate now ignores the exit code. It saves the plan, reads `resource_drift` from `terraform show -json`, and names the drifted resources in the report.

**Bug 4: the test function lied about its layer.** `cicd-test` reports which version of a layer-provided package it can import, so a layer-upgrade test is visible. It reported a version with no layer attached. The package was `six`, which the Lambda Python runtime already provides. We switched to a package the runtime doesn't include, and the deployed function now reports `layer: null`, as it should.

---

## Results & Benchmarks

What's proven on the test function so far:

* **First deploy:** function, `live` alias and the first deployment record created through the pipeline
* **Code change:** v1 ➔ v2, zip built once, the plan's `source_code_hash` equal to the build's hash, `live` moved to the new version
* **Lambda `CodeSha256` = hash of the uploaded zip:** confirmed. This is the property the drift gate rests on.
* **Re-running the same commit:** zip reused, snapshot returned the existing version, **no new version published**, record rewritten as a no-op deploy
* **Existing pipeline platform files modified:** **0**; every pipeline, buildspec and resource is new

Next to prove: no-drift passes the gate, a console edit is stopped before approval, discard, adopt, and the rollback pipeline.

---

## Lessons Learned

* **What went well:**
  * Passing one small JSON file through the run and re-verifying it at every stage made each bug cheap to diagnose. The artifact's key, version and hash were always in the logs.
  * Testing on a function nobody depends on meant every one of these failures cost a re-run, not an outage.
* **Gotchas to watch out for:**
  * Don't trust Terraform's view of a computed value right after an in-place update. Ask the service.
  * Don't trust `-detailed-exitcode` on refresh-only plans in our Terraform version. Read the plan JSON.
  * If a stage writes your baseline, make sure a failure *before* it can't leave the system changed and unrecorded.
  * Packages "provided by your layer" may already be in the runtime. Test on a package that can only come from the layer.
  * Shared platform code with an unpinned dependency tree can redeploy itself on every unrelated plan. Ours did, and it showed up in the middle of reviewing our change.

---

## Conclusion

The storage, the module and a seven-stage pipeline are live in the CI account. A build-once zip is verified by key, version and hash at every stage, credentials are explicit with no fallback, and deployment records always describe what's live. The first deploys found four real bugs, and each one is fixed and written down before any client code goes near the pipeline.

**Call-to-Action:** Has a Terraform computed value ever lied to you after an apply? I'd love to hear which attributes others have learned to re-read from the API instead of from state.

*Next up — Part 4: Catching the Hand Edit — testing the drift gate, discard, adopt, and the rollback pipeline.*
