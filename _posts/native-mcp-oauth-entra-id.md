# From Credential Vending to One-Click SSO: Simplifying MCP Server Authentication with Entra ID

## The Platform

We run an internal MCP (Model Context Protocol) server platform on AWS that gives engineering teams AI-powered access to internal tools — source control, service management, project portfolio data — all through their IDE.

The platform runs entirely behind VPN. Multiple MCP servers sit on ECS Fargate behind an internal Application Load Balancer with path-based routing. Traffic flows from the user's machine through the corporate VPN, over a Transit Gateway, to the ALB, and into the relevant container. No component is internet-facing. The servers are thin proxies — they expose backend systems as MCP tools that AI assistants can call.

```
User (on VPN) → Transit Gateway → Internal ALB → ECS Fargate
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
              /server-a/*      /server-b/*       /server-c/*
                    │                │                │
              Internal API     S3 Bucket        External SaaS
```

Adding a new server is a single entry in a Terraform map — a `for_each` pattern creates the ECR repo, target group, listener rule, task definition, ECS service, auth scope, and health alarm automatically.

## The Problem

Authentication was our biggest pain point. Here's what a user had to do to connect:

1. Log into AWS via SSO
2. Invoke a credential-vending Lambda (SigV4 signed) to get a Cognito `client_id` and `client_secret`
3. Use those credentials to request a JWT from Cognito's token endpoint
4. Paste the client credentials into their `mcp.json` configuration
5. Repeat when credentials expire or access changes

Behind the scenes, this required:

- A **Cognito User Pool** with a custom domain
- A **Cognito Resource Server** defining per-server scopes
- A **credential-vending Lambda** that creates per-user Cognito app clients with the right scopes
- A **Lambda Function URL** (IAM-authenticated) as the entry point
- A **DynamoDB table** mapping users to their allowed servers
- A **Terraform variable** maintaining the same mapping declaratively

Seven infrastructure components just for auth. Every time we onboarded a user, it was a Terraform change, a pipeline run, and a setup guide walkthrough. Every time someone left, we had to remember to remove them from the map and re-apply.

It worked, but it was a workaround. We built it because we assumed MCP clients couldn't do interactive OAuth flows. That assumption was wrong.

## The Realisation

The MCP specification (2025-03-26 revision) defines a native OAuth 2.1 authorization flow. Claude Desktop, Claude Code, and Kiro all support it. When a client connects to a remote MCP server and gets a `401`, it:

1. Reads the `WWW-Authenticate` header to find the protected resource metadata URL
2. Fetches `/.well-known/oauth-protected-resource` to discover the authorization server
3. Opens the user's browser for login
4. Receives the auth code via a callback (localhost for CLI/IDE clients, or a relay for desktop apps)
5. Exchanges the code for a token using PKCE
6. Reconnects to the MCP server with a Bearer token
7. Handles token refresh automatically

This is the same flow you experience when VS Code prompts "Sign in with Microsoft" or Slack opens a browser for SSO. The protocol handles everything — the user just clicks "Connect" and signs in.

## The Solution

We're replacing the entire credential-vending stack with native MCP OAuth 2.1, backed by Microsoft Entra ID (our existing corporate identity provider).

### How it works

When a user connects their IDE to one of our MCP servers for the first time:

1. The MCP server returns `401` with a pointer to its OAuth metadata
2. The client discovers that Entra ID is the authorization server
3. A browser tab opens — the user sees the familiar Microsoft login (same SSO as email and Teams)
4. Since they're already signed into Entra via their desktop session, it's typically zero-interaction
5. The auth code returns to the client via a localhost callback (Claude Code, Kiro) or a relay (Claude Desktop)
6. The client exchanges the code for a JWT using PKCE — no client secret needed
7. The client stores the token and refreshes it automatically

From the user's perspective: click Connect, browser flashes, done. No AWS CLI, no Lambda invocations, no credential pasting.

### Access control without infrastructure

Instead of Cognito scopes + DynamoDB + Terraform maps, we use **Entra ID App Roles**. Each MCP server has a corresponding role. Users get roles assigned via AD group membership.

When someone joins the team, their AD group membership grants MCP access automatically. When they leave, disabling their AD account revokes access instantly — no Terraform change, no pipeline, no cleanup task.

The MCP servers validate the JWT (signature check against Entra's JWKS endpoint) and inspect the `roles` claim. If the user doesn't have the right role, they get a `403`.

### What we're eliminating

| Component | Fate |
|-----------|------|
| Cognito User Pool | Removed |
| Cognito Resource Server + scopes | Removed |
| Cognito Custom Domain | Removed |
| Credential Vending Lambda | Removed |
| Lambda Function URL | Removed |
| DynamoDB user-access table | Removed |
| Terraform user-mapping variable | Removed |
| Multi-step user setup guide | Replaced with "click Connect" |

### What stays the same

The compute layer is untouched. ECS Fargate services behind an internal ALB with path-based routing. Supporting Lambdas. S3 buckets, DynamoDB tables, ECR repos, monitoring. All unchanged.

### Security posture

The security model actually improves:

- **For Claude Code and Kiro** — the entire OAuth flow runs locally on the user's machine. The token is issued by Entra and sent directly to the internal ALB. No third party is involved at any point.
- **For Claude Desktop** — Anthropic's relay handles the OAuth callback (briefly sees the auth code), but never sees MCP tool call data. All actual traffic flows directly from the user's machine to the ALB over VPN.
- **Tokens are short-lived** (1 hour default), scoped exclusively to the MCP app registration, and bound to PKCE. They can't be used to access any other Entra-protected resource.

The trust model is identical to any "Sign in with Microsoft" integration — Slack, Jira, GitHub — where the provider briefly handles the OAuth redirect.

## What the MCP servers need

The code changes on each server are minimal:

1. **Return a proper `401`** with a `WWW-Authenticate` header pointing to the protected resource metadata endpoint
2. **Serve the protected resource metadata** at `/.well-known/oauth-protected-resource` — a static JSON document pointing to Entra as the authorization server
3. **Validate JWTs** against Entra's JWKS endpoint (issuer, audience, signature, expiry)
4. **Check the `roles` claim** — reject with `403` if the user lacks the appropriate role

That's roughly 50-80 lines of middleware per server.

## What Entra needs

A single App Registration:

- An Application ID URI (used as the token audience)
- App Roles defined (one per server or permission level)
- Redirect URIs for localhost callbacks (Claude Code, Kiro) and the `claude.ai` callback (Claude Desktop)
- "Allow public client flows" enabled (PKCE without a client secret)
- `roles` optional claim added to access tokens

User and group role assignments are then managed entirely within Entra by AD administrators — no infrastructure changes for access control.

### Redirect URI configuration

This is the most common gotcha. Localhost URIs (`http://localhost/callback`, `http://127.0.0.1/callback`) must be registered under the **Mobile and desktop applications** (public client) platform in Entra, not Web. The `https://claude.ai/api/mcp/auth_callback` URI for Claude Desktop goes under the **Web** platform. Getting this wrong is the #1 reason the flow silently fails.

## The outcome

| Before | After |
|--------|-------|
| 7 infrastructure components for auth | 1 Entra App Registration |
| Multi-step manual user onboarding | Click Connect, sign in with SSO |
| Terraform change for every user change | AD group membership (instant) |
| Token expires → user repeats credential flow | Auto-refresh handled by client |
| User leaves → manual cleanup | AD account disabled → access revoked |
| Custom Lambda code to maintain | Standard JWT validation middleware |

## Lessons and recommendations

If you're running remote MCP servers and considering authentication:

1. **Check if your MCP clients support OAuth 2.1 natively.** Claude Desktop, Claude Code, and Kiro do. If yours does, you don't need a credential-vending workaround.

2. **Use your existing corporate IdP.** If your org already uses Entra ID (or Okta, or any OIDC-compliant provider), point the MCP OAuth flow at it. Users are already authenticated — the SSO session means zero-friction login.

3. **App Roles > custom scope infrastructure.** If your IdP supports role claims in tokens, use them for per-server access control. It moves the user-management burden from your platform team to your AD administrators, where it belongs.

4. **Localhost redirects for desktop/CLI clients.** Register them as public client URIs under "Mobile and desktop applications," not Web. Enable "Allow public client flows." This is standard per RFC 8252 but trips up everyone the first time.

5. **PKCE is mandatory, client secrets are not.** MCP clients operate as public clients (the user's machine can't securely store a secret). PKCE (S256) is your security boundary. No client secret needed.

6. **Start with the spike.** Wire up one server, register the Entra app, test the flow end-to-end from one client. The whole thing takes a day if your AD admin is responsive. The protocol is well-defined — you're not inventing anything.

The platform becomes simpler to operate, easier to onboard onto, and harder to misconfigure. The auth layer goes from "custom infrastructure we maintain" to "standard corporate SSO that IT manages." Less code, fewer components, and users stop asking how to get their credentials working.
