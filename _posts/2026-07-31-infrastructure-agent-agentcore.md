---
layout: post
title: Building Infrastructure Agent Architecture on AWS Bedrock AgentCore
subtitle: A 6-plane Approach to Safe & Secure Agents That Scale
#cover-img: /assets/img/path.jpg
#thumbnail-img: /assets/img/thumb.png
#share-img: /assets/img/path.jpg
tags: [infrastructure-agents]
author: Justin Odel
---

# Building the Infrastructure Agent Architecture from IaCConf 2026 with AWS Bedrock AgentCore

I recently attended **IaCConf 2026** and sat in on a talk about Infrastructure Agents — not the "chat with your cloud" demo variety, but a serious architectural breakdown of what it takes to run AI agents against production infrastructure safely. The speaker laid out a six-plane architecture that treats the problem as what it actually is: a distributed system where untrusted language model output drives privileged infrastructure operations.

I walked out of that talk thinking: *I want to build this.* And I want to build it on AWS using **Amazon Bedrock AgentCore** as the foundation.

This post is my attempt to map that architecture onto real AWS services — what AgentCore gives us for free, what we build ourselves, and where the gaps are.

## The Architecture: Six Planes

The talk presented infrastructure agents as six interacting planes:

1. **Ingestion** — How work enters the system (webhooks, schedules, chat, alerts)
2. **Policy** — What agents are allowed to do before they do it
3. **Execution** — Where agents actually run (isolated, stateless, scalable)
4. **Integration** — How agents talk to cloud APIs, git, CI/CD
5. **Change Control** — The GitOps pipeline ensuring agents never apply directly to prod
6. **Observability** — Tracing every decision, tool call, and infrastructure change

The key insight was the separation of concerns. An infrastructure agent isn't one monolithic thing — it's an ingestion pipeline feeding a policy gate feeding an isolated runtime feeding a change control workflow. Each plane has distinct reliability and security requirements.

## Why AgentCore?

After the talk, I spent time evaluating what it would take to build this from scratch versus using a managed platform. The execution plane alone — session isolation, credential handling, auto-scaling, observability — is months of undifferentiated work.

AgentCore (GA since mid-2025, now with 12 components across 14+ regions) handles exactly the parts I don't want to build myself:

- **Runtime** — Serverless, per-session microVM isolation. Stateless workers that can crash without corrupting state. This is the entire Plane 3 in a managed service.
- **Gateway** — Exposes Lambda functions and APIs as MCP-compatible tools. The integration surface for everything the agent can touch.
- **Policy** — Cedar-based authorization that intercepts every tool call at the Gateway. Default-deny posture.
- **Memory** — Persistent context across sessions (semantic, summary, user preferences).
- **Identity** — OAuth 2.0 via Cognito for both user-facing and machine-to-machine auth.
- **Observability** — Built-in OpenTelemetry with CloudWatch integration.

That's three full planes (Execution, Observability, and most of Policy) handled out of the box.

## The Build Plan: Mapping Planes to AWS Services

Here's how I'm planning to implement each plane.

### Plane 1: Ingestion — EventBridge + API Gateway + Lambda

AgentCore Runtime accepts invocations but isn't an event router. The ingestion plane is ours to build.

```
GitHub Webhooks ──┐
CloudWatch Alarms ─┤
EventBridge Rules ─┼──▶ API Gateway ──▶ Signal Router Lambda ──▶ AgentCore Runtime
Slack Commands ────┤
Schedule (cron) ───┘
```

**AWS Services:**
- **Amazon EventBridge** — Central event bus for webhook delivery, schedule triggers, and cross-service events
- **API Gateway** — HTTP endpoint for external webhooks (GitHub, PagerDuty, Slack)
- **Lambda** (Signal Router) — Normalizes all signals into a common task format and invokes the appropriate AgentCore Runtime agent

The Signal Router Lambda handles priority classification, deduplication, and mapping each signal to the right agent configuration. Everything normalizes into a common dispatch format before hitting AgentCore.

### Plane 2: Policy — Cedar (Native) + OPA (Terraform Layer)

This is where it gets interesting. The talk emphasized autonomy tiers — not all agent actions carry equal risk. A read-only drift scan is different from a `terraform apply` to production.

**AgentCore Policy (Cedar)** handles agent-to-tool authorization:

```cedar
// Tier 0: Any agent can observe
permit(
  principal,
  action == Action::"tools/call",
  resource
) when {
  resource.toolName like "describe-*" ||
  resource.toolName like "list-*" ||
  resource.toolName like "get-*"
};

// Tier 2: Draft PRs allowed without approval
permit(
  principal == Agent::"infra-remediation-agent",
  action == Action::"tools/call",
  resource
) when {
  resource.toolName == "create-pull-request"
};

// Tier 4: Production apply requires human approval
forbid(
  principal,
  action == Action::"tools/call",
  resource
) when {
  resource.toolName == "terraform-apply" &&
  resource has environment &&
  resource.environment == "production"
} unless {
  context.humanApproval == true
};
```

**OPA/Rego** handles infrastructure policy at the Terraform layer — what resources are valid, what configurations are allowed:

```rego
# Deny unencrypted S3 buckets
deny[msg] {
  resource := input.planned_values.root_module.resources[_]
  resource.type == "aws_s3_bucket"
  not has_encryption(resource)
  msg := sprintf("S3 bucket '%s' must have encryption enabled", [resource.name])
}

# Enforce required tags
deny[msg] {
  resource := input.planned_values.root_module.resources[_]
  required_tags := {"Environment", "Owner", "CostCenter"}
  provided_tags := {tag | resource.values.tags[tag]}
  missing := required_tags - provided_tags
  count(missing) > 0
  msg := sprintf("Resource '%s' missing required tags: %v", [resource.name, missing])
}
```

The layered approach: Cedar controls what the agent *can do*, OPA controls what the infrastructure *should look like*. Both enforce before any change lands.

### Plane 3: Execution — AgentCore Runtime (Fully Managed)

This is where AgentCore shines. The talk described the ideal execution plane as:

- Stateless workers that never touch the database directly
- Session isolation so one agent run can't affect another
- Auto-scaling without managing queue infrastructure
- Framework-agnostic (bring your own agent framework)

AgentCore Runtime delivers all of this. Per-session microVMs, serverless scaling, support for Strands Agents, LangGraph, CrewAI — whatever framework fits the use case.

No Redis Streams, no BullMQ, no SQS consumer groups to manage. The entire task queue / worker pool architecture from the talk collapses into:

```bash
agentcore deploy --runtime-config runtime.yaml
```

The trade-off is less control over queue backpressure and priority routing — but we handle priority in the ingestion layer before invoking Runtime, so that's fine.

### Plane 4: Integration — Lambda Tools Behind Gateway

The integration plane is where the agent connects to the outside world. The talk's key principle: *the agent never holds credentials; it requests them just-in-time from a credential broker.*

**AgentCore Gateway** is the tool surface. Each integration becomes a Lambda function exposed through Gateway:

| Tool | Lambda Function | What It Does |
|------|----------------|--------------|
| `terraform-plan` | `infra-agent-tf-plan` | Clones repo, runs `terraform plan`, returns plan output |
| `terraform-apply` | `infra-agent-tf-apply` | Applies a previously approved plan |
| `opa-validate` | `infra-agent-opa-check` | Evaluates plan JSON against Rego policies |
| `git-create-branch` | `infra-agent-git-ops` | Creates branch, commits changes, pushes |
| `create-pull-request` | `infra-agent-pr` | Opens PR with plan output and change summary |
| `trigger-pipeline` | `infra-agent-cicd` | Kicks off CI/CD validation pipeline |
| `get-credentials` | `infra-agent-cred-broker` | Returns short-lived STS tokens scoped to the task |
| `prowler-scan` | `infra-agent-compliance` | Runs compliance checks against target account |

**Credential Brokering** is a Lambda that:
1. Receives the agent's task context (account, role, scope)
2. Calls STS `AssumeRole` with a session policy scoping to minimum required permissions
3. Returns short-lived credentials (15-minute TTL)
4. Logs every credential mint to CloudTrail

The agent itself never sees long-lived secrets. AgentCore Identity handles the auth *to* Gateway; the credential broker handles auth *from* Gateway tools to target AWS accounts.

### Plane 5: Change Control — Custom GitOps Pipeline

AgentCore has no opinion on GitOps. This is entirely our design, enforced through tool design and Cedar policies.

**The golden rule:** Agent writes code → creates PR → CI validates → human reviews → CI applies.

The flow implemented as Gateway tools:

```
Agent analyzes finding
    │
    ▼
git-create-branch (Lambda tool)
    │
    ▼
Agent writes Terraform fix
    │
    ▼
opa-validate (Lambda tool) ──── Fail? ──▶ Agent iterates
    │
    Pass
    ▼
terraform-plan (Lambda tool) ── Drift? ──▶ Agent iterates
    │
    Clean plan
    ▼
create-pull-request (Lambda tool)
    │
    ▼
trigger-pipeline (Lambda tool) ── CI runs plan + checkov + tests
    │
    ▼
Human reviews and merges
    │
    ▼
CI/CD applies (CodePipeline / GitHub Actions)
```

Cedar policies enforce this flow structurally:

```cedar
// Agent cannot call terraform-apply without a linked PR
forbid(
  principal,
  action == Action::"tools/call",
  resource
) when {
  resource.toolName == "terraform-apply" &&
  !(context has pullRequestId)
};

// Agent cannot merge its own PRs
forbid(
  principal,
  action == Action::"tools/call",
  resource
) when {
  resource.toolName == "merge-pull-request"
};
```

There's no path for the agent to bypass the PR workflow. It physically cannot call apply without a PR, and it cannot merge its own work.

### Plane 6: Observability — AgentCore Native + Custom Spans

AgentCore's built-in observability handles:
- Agent reasoning traces (tool calls, LLM interactions)
- Policy evaluation logs (what was allowed/denied and why)
- Latency, token usage, error rates
- All piped to CloudWatch via OpenTelemetry

We extend it with custom instrumentation for infrastructure-specific concerns:

**Additional AWS services:**
- **CloudWatch Logs** — Structured logs from all Lambda tools
- **CloudTrail** — Credential broker audit trail
- **X-Ray** — Distributed traces across Gateway → Lambda → target APIs
- **CloudWatch Dashboards** — Operational view: tasks processed, PRs created, policy denials, mean time to remediation

Custom metrics we'd track:

| Metric | Why |
|--------|-----|
| Tasks per trigger type | Understand workload distribution |
| Policy denial rate | Are policies too restrictive? |
| Mean iterations to clean plan | Agent efficiency |
| Time from finding to PR | Remediation SLA |
| Credential broker calls per task | Security audit |
| Token spend per task type | Cost management |

## The Full AWS Architecture

Putting it all together:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INGESTION PLANE                              │
│                                                                     │
│  GitHub Webhooks ──┐                                                │
│  EventBridge ──────┼──▶ API Gateway ──▶ Signal Router (Lambda)      │
│  CloudWatch Alarms─┤                         │                      │
│  Slack / Chat ─────┘                         │                      │
└──────────────────────────────────────────────┼──────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          POLICY PLANE                                │
│                                                                     │
│  AgentCore Policy (Cedar)          OPA (Terraform validation)       │
│  ├── Agent-to-tool authorization   ├── Resource policy checks       │
│  ├── Autonomy tier enforcement     ├── Tag compliance               │
│  ├── Human approval gates          └── Configuration guardrails     │
│  └── Default-deny posture                                           │
└─────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        EXECUTION PLANE                               │
│                                                                     │
│  AgentCore Runtime (Serverless, per-session microVM isolation)       │
│  ├── Strands Agent + Claude Sonnet                                  │
│  ├── Framework-agnostic                                             │
│  ├── AgentCore Memory (session context)                             │
│  └── Auto-scaling, zero infrastructure management                   │
└─────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       INTEGRATION PLANE                              │
│                                                                     │
│  AgentCore Gateway (MCP-compatible tool surface)                    │
│  ├── terraform-plan (Lambda)                                        │
│  ├── terraform-apply (Lambda)                                       │
│  ├── opa-validate (Lambda)                                          │
│  ├── git-operations (Lambda)                                        │
│  ├── create-pull-request (Lambda)                                   │
│  ├── credential-broker (Lambda + STS)                               │
│  ├── compliance-scan (Lambda + Prowler)                             │
│  └── trigger-pipeline (Lambda + CodePipeline)                       │
│                                                                     │
│  AgentCore Identity (Cognito — Web + M2M OAuth flows)               │
└─────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     CHANGE CONTROL PLANE                             │
│                                                                     │
│  Agent → Branch → OPA Check → Plan → PR → CI Validates → Human     │
│  (Cedar enforces: no apply without PR, no self-merge)               │
│                                                                     │
│  CodePipeline / GitHub Actions for final apply                      │
└─────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      OBSERVABILITY PLANE                             │
│                                                                     │
│  AgentCore Observability (OpenTelemetry → CloudWatch)               │
│  ├── Agent reasoning traces                                         │
│  ├── Policy evaluation logs                                         │
│  ├── Token usage and cost metrics                                   │
│  X-Ray (distributed traces)                                         │
│  CloudTrail (credential audit)                                      │
│  CloudWatch Dashboards (operational metrics)                        │
└─────────────────────────────────────────────────────────────────────┘
```

## What AgentCore Buys Us (and What It Doesn't)

| Plane | AgentCore Coverage | What We Build |
|-------|-------------------|---------------|
| Ingestion | ⚠️ Invocation API only | EventBridge + API Gateway + Signal Router Lambda |
| Policy | ✅ Cedar at Gateway | Autonomy tier model, OPA for Terraform validation |
| Execution | ✅ Fully managed | Nothing — Runtime handles isolation, scaling, state |
| Integration | ✅ Gateway + Identity | Lambda tools (git, Terraform, creds, CI/CD) |
| Change Control | ❌ No opinion | GitOps flow enforced via tool design + Cedar |
| Observability | ✅ Native OTel | Custom metrics, dashboards, credential auditing |

**Rough estimate:** AgentCore eliminates 60-70% of the undifferentiated infrastructure work. What remains is domain-specific: the git workflow tools, Terraform orchestration, OPA policy library, and the event routing layer.

## What I'm Building First

My plan is to start with a single use case — **compliance remediation** — and expand from there:

1. **Week 1-2:** AgentCore Runtime + a single Gateway tool (`terraform-plan`) + Cedar policy (read-only tier)
2. **Week 3-4:** Add git operations, PR creation, OPA validation tools. Move to Tier 2 (draft PRs).
3. **Week 5-6:** EventBridge integration for automated compliance finding ingestion. Credential broker.
4. **Week 7-8:** Observability dashboards, cost tracking, policy tuning based on real usage.

The beauty of AgentCore's modular design is that each component is independent. Start with Runtime and Gateway, add Policy when you're ready to open up more tools, add Memory when you need cross-session context.

## Final Thoughts

The IaCConf talk reframed how I think about infrastructure agents. They're not a chatbot with AWS credentials — they're a distributed system with real security boundaries, change control requirements, and observability needs.

AgentCore doesn't solve the whole problem, but it solves the hardest parts of the *platform* problem so I can focus on the *domain* problem: how do you safely automate infrastructure changes with AI?

The Cedar + OPA layered policy approach gives us defense in depth. The GitOps change control plane ensures agents follow the same workflows as humans. And the observability plane means we can actually trust what the agent is doing — or catch it when we can't.

I'll be writing follow-up posts as I build this out. Next up: the credential broker design and how to scope agent permissions with STS session policies.

---

*Attended IaCConf 2026. Building infrastructure agents on AWS. Opinions are my own.*
