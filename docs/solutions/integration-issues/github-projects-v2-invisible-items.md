---
title: "GitHub Projects v2 — items invisible after add (eventual-consistency lag)"
problem_type: integration_issue
component: github-projects-v2
date_solved: 2026-04-28
tags:
  - github-api
  - graphql
  - projectv2
  - eventual-consistency
  - agent-workflows
symptoms:
  - "Item added to a ProjectV2 retrievable by `node(id: ...)` but missing from web UI"
  - "Project's `filterQuery` search returns 0 results despite the item existing"
  - "Insights / Burn-up / Developer Velocity charts under-count points and items"
  - "`ProjectV2.items.totalCount` frozen across multiple add operations"
  - "Issue page's Project sidebar shows Status only, no Sprint/Points pickers"
related_issues:
  - "Atram-Inc/.github#6"
  - "Atram-Inc/.github#7"
  - "Atram-Inc/meta-utilities#1"
---

# GitHub Projects v2 — items invisible after add (eventual-consistency lag)

## Symptom

An issue added to a ProjectV2 (org-level project) is accepted by every API surface but does not appear in:

- the project's **web UI** (board or table)
- the **filterQuery** search box
- the project's **Insights** charts (Burn up, Developer Velocity, etc.)
- a paginated query of `ProjectV2.items`

The item is retrievable by direct `node(id: ...)` lookup, and `Issue.projectItems` correctly lists the project. The bidirectional link is half-present: issue→project works, project→items does not. Sprint and Points custom-field pickers also fail to render in the issue's project sidebar (because the same lagged join powers them).

Concretely, on 2026-04-28 we saw `ProjectV2.items.totalCount` frozen at 84 across ~2 hours of adds and field updates. Five items added through four different paths were all invisible. The Developer Velocity chart was missing 2 points of work for the active sprint.

## Investigation steps tried (none of these were the fix)

Each was attempted on the same broken state and proved orthogonal to the cause:

1. `gh project item-add` → `gh project item-edit` per field — invisible.
2. GraphQL `addProjectV2ItemById` mutation directly — invisible.
3. Web UI: pick the project from the New Issue form's sidebar — invisible.
4. Web UI: "Add item" from the project board — invisible.
5. Setting all of Status / Sprint / Points / Assignees on a stranded item — still invisible (`totalCount` did not move).
6. Delete and recreate the issue + item from scratch — still invisible.
7. Visiting the issue page in an authenticated browser session (theory: nudge the index) — no effect.
8. Switching the host repo (`yaya-engine` instead of `.github`) — still invisible. Older items in the same repo were fine; the cutoff was time-based, not repo-based.
9. Polling `ProjectV2.items.totalCount` for ~15 min after each attempt — frozen at 84.

The "missing fields" theory is intuitive but wrong. Working items in the same project sometimes have no Sprint / Points / PID / Type / labels and are still visible. Field shape is not the differentiator.

## Root cause

GitHub maintains two indexes for the `Issue ↔ ProjectV2Item` relationship, and they are eventually consistent:

| Direction | Update behavior |
|---|---|
| `Issue.projectItems` | Per-issue. Updates immediately on add. |
| `ProjectV2.items` | Per-project. Asynchronously rebuilt. Powers the web UI, `filterQuery`, and Insights. |

The project-side index normally lags by seconds. On 2026-04-28 it lagged by hours. While stale, the item logically exists (fetchable by ID, linked from the issue) but every surface that reads through `ProjectV2.items` is blind to it.

When the index caught up later in the day, `totalCount` jumped 84 → 87 with no client action — both `meta-utilities#1` (stranded ~hours) and `.github#6` (created post-recovery) appeared together. The exact same `gh issue create` + `addProjectV2ItemById` + three `item-edit` sequence that produced an invisible item earlier produced a visible one later.

## Working "solution"

There is no client-side fix. The lag clears on its own. The defensive practice is to **detect the lag and not declare success during it**.

After adding an item:

```bash
ITEM_ID=$(gh api graphql -f query="mutation {
  addProjectV2ItemById(input: {projectId: \"$PROJECT_ID\", contentId: \"$ISSUE_NODE_ID\"}) {
    item { id }
  }
}" --jq '.data.addProjectV2ItemById.item.id')

# (optional: set Status / Sprint / Points via gh project item-edit)

# Verify the item appears in ProjectV2.items — paginate if the project has > 100 items
gh api graphql -f query='query {
  organization(login: "Atram-Inc") {
    projectV2(number: 1) {
      items(first: 100) { totalCount pageInfo { endCursor hasNextPage } nodes { id } }
    }
  }
}' --jq '.data.organization.projectV2.items.nodes[].id' | grep -q "$ITEM_ID" \
  && echo "visible — safe to report success" \
  || echo "WARNING: stranded by indexing lag — surface the lag, do not retry"
```

If the item is missing:

- Surface the lag explicitly to the requester.
- Do **not** delete and recreate. Recreation lands in the same lag window.
- Do **not** churn fields trying to nudge the index. They don't help.
- Either wait and re-check, or accept the inconsistency and move on.

## Prevention

- **Always verify with `ProjectV2.items`** before reporting "added to project" as complete. `Issue.projectItems` updates instantly and lies about UI / Insights visibility.
- Paginate `ProjectV2.items` (the GraphQL `first:` cap is 100). Do not rely on a single page when the project has more.
- For agent-driven workflows: model "added but invisible" as a known-impossible-now condition, not a recoverable error. The corrective action is not retry, it is wait or escalate.
- When debugging an "issue not showing up" report, check for this lag before assuming user error or looking for a hidden filter. The signature is: `Issue.projectItems` lists the project, but `ProjectV2.items` does not include the item ID, and `totalCount` is not increasing as you add.

## Cross-references

- [`AGENTS.md`](https://github.com/Atram-Inc/.github/blob/main/AGENTS.md) — the org-wide AI agent notes file documents this gotcha and the related "label deletion" zombie-label gotcha.
- [`Atram-Inc/.github#6`](https://github.com/Atram-Inc/.github/issues/6) — duplicate created to replace an issue (`#3`) stranded in the lag window.
- [`Atram-Inc/meta-utilities#1`](https://github.com/Atram-Inc/meta-utilities/issues/1) — also stranded; recovered when the backend caught up.
- [`Atram-Inc/.github#7`](https://github.com/Atram-Inc/.github/issues/7) — sibling cleanup issue from the same session, documenting the deprecated `points:*` label retirement and the label-deletion gotcha.
