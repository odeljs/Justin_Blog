---
title: "AI Infrastructure Agents: The Credential Broker"
date: 2026-10-28
author: Justin Odel
tags: [cloud, ai, agents, security, credentials, secrets, git, lambda, dynamodb]
description: "Agents should never hold long-lived credentials. Here's the broker that mints short-lived, scoped git tokens on demand and audits every issuance — plus the messy real-world lessons from building it against a Git host that fought us every step."
series: "AI Infrastructure Agents"
part: 3
---

# The Credential Broker

*Part 3 of the series: AI Infrastructure Agents*

## Introduction

* **The Hook:** An agent working inside a sandbox needs to push a branch. The lazy answer is to bake a Git token into the worker's environment. Do that, and you've created a long-lived, broadly-scoped credential sitting in the most attackable part of the system — the place running model-generated code against untrusted input. One prompt injection or one leaked log line later, that token is someone else's.
* **The Problem:** The agent genuinely needs to write to Git to open a PR. How do you grant that without ever giving it a credential worth stealing?
* **The Takeaway:** A credential broker. The agent requests access just-in-time; the broker mints a **short-lived, narrowly-scoped** token, hands it over, records the issuance in an immutable audit log, and ensures the token stops working when its time is up. This post is how we built it — and the three places the real Git host refused to behave like the textbook.

---

## Architecture Overview

* **Current State:** Coming out of the Tier 0 foundation, the agent could read but not change anything. To move toward "open a PR," it needs Git write access — but per our principles, it must never hold a long-lived credential. The broker is the first brick of the Integration plane, and it's deliberately self-contained: one integration (Git), one job (issue and audit scoped tokens).

```text
   Sandboxed worker                         ┌─────────────────────────┐
     │  "I need to push to repo X"          │  Secrets store          │
     │  request (signed)                    │  (one long-lived source │
     ▼                                      │   credential lives here)│
   ┌──────────────────────────┐  read       └────────────▲────────────┘
   │  Credential Broker         │───────────────────────────┘
   │  (on-demand function)      │
   │                            │  mint scoped token
   │  1. validate request       │──────────────► Git host API
   │  2. clamp TTL (fail-safe)   │◄────────────── token + host-side id
   │  3. mint scoped token       │
   │  4. record id + expiry      │──────────────► tracking table (mutable)
   │  5. write immutable audit   │──────────────► audit table (append-only)
   │  6. return token only       │
   └────────────┬───────────────┘
                │  { token, expires_at, scope, request_id }
                ▼
      worker uses it for one push, then discards it.

   Separately, on a schedule:
   timer ──► broker (reap mode) ──► delete every token past its expiry
                                    + write a "revoked" audit record
```

The only component that ever touches the long-lived source credential is the broker. Callers only ever see a short-lived token that expires on its own.

---

## The Root Cause (what the broker is really protecting against)

Everything the broker does ladders up to one rule: *the agent never holds a long-lived credential.* The threat isn't an evil agent; it's a **compromised** one — a prompt injection in a finding, a bug that logs a token, a dependency CVE that exfiltrates worker memory. Short-lived, narrowly-scoped tokens turn all of those from "standing breach" into "a time-boxed incident with an audit trail."

So the broker's contract is strict by design:

* **Scoped:** a request names exactly one repository. Multi-repo and wildcard requests are rejected.
* **Short-lived:** the broker *clamps* the requested TTL to a configured maximum. A caller asking for a 30-day token gets minutes. The control point refuses to issue anything long-lived, full stop.
* **Audited:** every issuance — success, denial, or error — writes one immutable record capturing who asked, for what, for how long, and the outcome. The token value itself is never stored.
* **No leakage:** the long-lived source credential never appears in a response or a log.

---

## The Solution

### Step 1: An on-demand function, not a running service

The broker is a function invoked only when needed. A broker that isn't running has no secret in memory to steal. It reads its source credential from a managed secrets store at call time, mints a token, and returns.

### Step 2: Clamp the TTL — never trust the caller

```text
granted_ttl = min(requested_ttl, MAX_TTL)
```

One line, but it's the whole fail-safe posture. The broker is the control point; it does not let a bug or a compromised caller talk it into a long-lived token.

### Step 3: Audit on every path, immutably

The audit record is written whether the request succeeds, is denied, or errors. Immutability is enforced by the *permission set*, not by good intentions: the broker's role can write new audit records but cannot update or delete them. A record, once written, is permanent.

### Step 4: Enforce expiry actively

Here's where theory met reality (see below). The broker records each minted token's host-side id and expiry in a tracking table, and a scheduled "reaper" deletes any token whose time has elapsed, writing a `revoked` audit record as it does. Expiry becomes something the platform *enforces*, not something it hopes the caller honors.

---

## When the real world fought back

The textbook version of this (mint a token that carries its own short expiry and tight scope, like cloud STS) assumes a cooperative host. Our self-hosted Git host disagreed in three ways, and the honest lessons are worth more than the happy path.

**1. The tokens don't expire on their own.** The host's user tokens have no native TTL. "The token expires" would have been a *promise* ("the worker pinky-swears to stop using it"), not a *guarantee*. We refused to ship a promise. The fix was the reaper: track every token and actively delete it on expiry. More moving parts, but now expiry is real — if the reaper stops, that's a monitored failure, not a silent one.

**2. Scopes aren't per-repository.** The host scopes tokens to route groups (e.g. "repository write"), which applies across *every* repo the owning account can reach — not to a single repo. You cannot narrow a user token to one repository. So the real blast-radius boundary became the **bot account's membership**: the identity the broker mints under only has access to the repos agents are allowed to touch. We validate and record the single requested repo, but we're honest in the docs that the hard boundary is account membership, not token scope.

**3. Creating a token requires password auth, not a token.** The endpoint that creates tokens wouldn't accept a bearer token — it demanded HTTP Basic auth as the user. That forced an uncomfortable tradeoff: the broker's source credential had to become the bot account's **password** rather than a scoped, revocable token. We took it as a documented, temporary "Option A," with a clear follow-up to pivot to per-repo deploy keys (which *are* per-repo and *do* accept token auth). We explicitly rejected a tempting shortcut — presenting a token over Basic auth — because on this host that path bypasses scope enforcement entirely (a known CVE). "It works" is not the same as "it's safe."

---

## A bug worth telling on ourselves

During live testing, the full lifecycle worked: issue a token, see it audited, force it past expiry, watch the reaper delete it and write a `revoked` record. Then we looked closer and found the audit log was lying.

The audit table was keyed by request id alone. The reaper's `revoked` record used the *same* request id as the original `issued` record — so writing the revocation **overwrote the issuance**. After reaping, the "we issued this" event was simply gone. The one guarantee the audit table exists to provide — an immutable record *per issuance* — was broken by the revocation path.

The fix was a composite key: `request_id` + an `event_ts` sort key, so `issued` and `revoked` are separate, coexisting, immutable items. DynamoDB key schemas can't change in place, so it meant recreating the table — cheap here, but a real lesson: **design the audit key for the whole lifecycle, not just the first event.** We verified the fix live: one request now yields both an `issued` and a `revoked` record, each keyed distinctly, neither storing the token.

```text
Before (single key):  request_id ──► [issued]  ──reaper──►  [revoked]   (issued overwritten!)
After (composite):     request_id + event_ts ──► [issued @ t0]
                                               └► [revoked @ t1]         (both survive)
```

---

## Results & Benchmarks

Verified end-to-end against the live Git host in a non-production account:

* **Credential the agent holds:** none long-lived — only a minted, short-lived token (minutes).
* **Requested-vs-granted TTL:** a day-long request ➔ clamped to the configured max.
* **Audit records per lifecycle:** `issued` + `revoked`, both immutable, token value never stored.
* **Expiry:** actively enforced — reaper deletes expired tokens and records the revocation.
* **Secret exposure in Terraform state:** none — the source secret is created out of band and only *looked up*, never managed by IaC.

---

## Lessons Learned

* **What went well:** Writing the request/response contract first meant the broker is a clean brick the next story can consume without knowing its internals. Enforcing immutability through permissions (write-only, no update/delete) rather than discipline caught the overwrite bug conceptually — the data model, not the IAM, was the weak point, and the fix was obvious once framed that way.
* **Gotchas to watch out for:** Don't assume your Git host behaves like cloud STS — verify native TTL, scope granularity, and the auth model of the *token-creation* endpoint before you design around them. And keep a credential's power proportional to its exposure: being forced to store a reusable password is a smell, so treat it as temporary and write down the path off it.

---

## Conclusion

The broker gives agents real Git write access without ever handing them a credential worth stealing: short-lived, scoped, audited, and actively expired. Getting there meant working around a host that had no native token expiry, no per-repo scope, and a password-only token-creation API — and fixing an audit model that quietly overwrote its own history. The result is the foundation brick the PR-first workflow will consume next.

**Call-to-Action:** How do you hand short-lived credentials to automation that runs untrusted code — broker-and-mint, or something else? And have you been bitten by an audit log that overwrote itself? I'd like to hear it.

*Next up — Part 4: PR-First Change Control — turning an agent's diff into a pull request a human can trust.*
