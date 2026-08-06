---
layout: page
title: "Post Moretem - Gitea Permission Mode Invalid"
date: 2026-08-05
description: ""permission mode invalid" — Gitea Team Update Failure"
---

**Date:** 2026-08-05  
**Severity:** Medium — Pipeline blocked, new repository inaccessible to non-admin users  
**Duration:** ~1 hour (from failed apply to root cause identification)  
**Affected Systems:** Terraform pipeline, Gitea team permissions  

---

## Summary

Adding a new repository to the Gitea configuration triggered a Terraform apply failure with the error `"permission mode invalid"` on two team resources(we maintain a cicd pipeline that deploys our gitea repos via terraform). The repository was successfully created but team membership updates failed, leaving non-admin users unable to see or access the new repository.

---

## Impact

- The repository was created successfully (visible to admins via `include_all_repositories = true`)
- The developers and cicd teams were NOT updated — the new repo was not added to their access list
- All users on these teams (non-admin) could not see or access the repository
- Branch protections and webhooks were applied successfully (these ran before the team update step)
- No data loss or security exposure

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 12:07:25 | Plan stage CodeBuild starts |
| 12:08:37 | Plan completes successfully — shows cicd and developers teams will be updated (adding new repository to list) |
| 12:15:57 | Apply stage CodeBuild starts |
| 12:17:50 | Repository, branch protection, webhooks all created successfully |
| 12:17:56 | `gitea_team.teams` for both teams fail with `"permission mode invalid"` |
| 12:17:56 | Apply exits with status 1 |

---

## Root Cause

The Gitea server was upgraded to **version 1.26.1** at some point prior to this apply. In Gitea 1.24+, the team permission model changed:

- **Before 1.24:** The API accepted `"read"`, `"write"`, and `"admin"` as team-level permission values directly.
- **After 1.24:** For non-admin teams, the API expects per-unit permissions via `units_map`. The team-level `permission` field of `"write"` is no longer accepted on its own — the API returns `"permission mode invalid"`.

The Terraform provider (`go-gitea/gitea` v0.7.0) sends the `permission` field on every update call. The team JSON(orgs, teams, users, repositories, etc. are all configured via json files) configs had `"permission": "write"` for both teams.

**Why it wasn't caught earlier:**

1. The `ignore_changes = [permission]` lifecycle rule in the Terraform resource prevented Terraform from detecting drift or planning permission changes.
2. These teams had not been modified via Terraform since before the Gitea upgrade — `ignore_changes` only suppresses *planning* a change, it doesn't prevent the provider from *sending* the permission value when other attributes change.
3. Adding our new repository to the repository list was the first modification to these teams post-upgrade, triggering the provider to call the Edit Team API endpoint.

**Terraform plan succeeded** because plan only validates syntax and state — it doesn't make API calls. The error only surfaces at apply time when the provider actually PATCHes the team via the Gitea API.

---

## Troubleshooting Steps

1. **Identified the error** — `"permission mode invalid"` on the two teams during `terraform apply`
2. **Inspected team JSON configs** — both had `"permission": "write"`, verified this is a valid value per the Terraform provider docs (`none`, `read`, `write`, `admin`, `owner`)
3. **Checked Gitea API version** — confirmed server is running `1.26.1` via `/api/v1/version`
4. **Reviewed Gitea 1.26 API docs** — `permission` enum lists `read`, `write`, `admin` as valid, but noted the introduction of `units_map` for granular per-unit permissions
5. **Inspected Gitea source code** (v1.22.1 `routers/api/v1/org/team.go`) — found that the `CreateTeam` and `EditTeam` handlers require `units` or `units_map` when permission is non-admin, and that the API now internally uses per-unit permissions
6. **Confirmed the error is at apply, not plan** — plan succeeded, error only occurs when the provider makes the HTTP PATCH request to the Gitea API
7. **Identified why other "write" teams didn't fail** — they had no changes in this apply, so no API call was made
8. **Confirmed user access issue** — the team update failure meant non-admin users could not see the new repo; admin users (with `include_all_repositories = true`) could see it
9. **Reviewed Terraform state** — confirmed the repo resource was created and exists in state, but team membership was not updated

---

## Resolution

Update the gitea tems module to translate `"write"` permission into the format Gitea 1.26 expects — team-level `"read"` with per-unit write access via `units_map`:

```hcl
resource "gitea_team" "teams" {
  for_each = local.team_repo_map

  name                     = each.value.name
  organisation             = each.value.organisation
  description              = each.value.description
  permission               = each.value.permission == "write" ? "read" : each.value.permission
  can_create_repos         = each.value.can_create_repos
  include_all_repositories = each.value.include_all_repositories
  repositories             = each.value.repositories
  units                    = each.value.units

  units_map = each.value.permission == "write" ? {
    "repo.code"     = "write"
    "repo.issues"   = "write"
    "repo.pulls"    = "write"
    "repo.releases" = "write"
    "repo.wiki"     = "write"
    "repo.projects" = "write"
    "repo.packages" = "write"
    "repo.actions"  = "write"
  } : null

  lifecycle {
    ignore_changes = [permission]
  }
}
```

This preserves the existing JSON config interface (`"permission": "write"` in team configs) while translating it to the Gitea 1.26 API-compatible format at the module level.

---

## Lessons Learned

1. **`ignore_changes` does not prevent the provider from sending values** — it only prevents Terraform from detecting drift and planning changes. When other attributes trigger an update, the full resource (including ignored fields) is still sent to the API.
2. **Gitea server upgrades can silently break Terraform workflows** — the plan stage won't catch API-level rejections since it doesn't make HTTP calls.
3. **Teams with `include_all_repositories = false` are brittle** — any failure in the team update leaves newly created repos inaccessible to those team members.

---

## References

- [Gitea PR #17811](https://github.com/go-gitea/gitea/pull/17811) — "Team permission allow different unit has different permission" (merged into Gitea 1.16, Jan 2022)
- [Gitea 1.24 Permissions Documentation](https://docs.gitea.com/1.24/usage/permissions)
- [Terraform Provider Source — resource_gitea_team.go](https://github.com/go-gitea/terraform-provider-gitea/blob/main/gitea/resource_gitea_team.go)
- [Gitea access_mode.go](https://github.com/go-gitea/gitea/blob/v1.22.1/models/perm/access_mode.go) — valid AccessMode values: none, read, write, admin, owner
