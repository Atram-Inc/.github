# AGENTS.md — Atram-Inc org notes for AI agents

Operational notes for AI agents working across Atram-Inc repos. Read before performing automated work on issues, projects, or labels.

## GitHub Projects v2 — invisible item gotcha

**Symptom**: An issue is added to the Engineering project (`Atram-Inc/projects/1`) via the API. Querying the project item by ID, or querying the issue's `projectItems`, both confirm it is on the project and not archived. But the item does **not** appear in the project's web UI (board or table), and `filterQuery` searches do not return it. The viewer cannot see Sprint or Points fields on the issue's project sidebar either — only Status.

**Diagnosis**: GitHub maintains separate indexes for the two directions of the `Issue ↔ ProjectV2Item` relationship, and they are eventually consistent.

| Direction | Source of truth |
|---|---|
| `Issue.projectItems` | Per-issue index — updates immediately on add |
| `ProjectV2.items` | Per-project index — eventually consistent; powers the web UI, filterQuery search, and Insights charts |

When the project-side index lags, the item exists and is retrievable by ID, but is invisible everywhere the web UI looks. The lag is normally seconds; on 2026-04-28 it was multiple hours.

**This is not caused by missing field values.** Tested empirically: setting Status, Sprint, Points, Assignees on a stranded item did **not** flush it into the project-side index. The fix was simply waiting for the backend to catch up.

**Reproduction** (encountered 2026-04-28):

- Created `Atram-Inc/.github#3`, `Atram-Inc/meta-utilities#1`, and a fresh `Atram-Inc/yaya-engine` test issue, all added to project 1.
- For ~2+ hours: items retrievable as `node(id: ...)` and listed under `Issue.projectItems`, but absent from `ProjectV2.items` (totalCount frozen at 84). Web UI, filterQuery, Insights charts, Developer Velocity bars all blind to them.
- After backend caught up: same items appeared with no client-side action. `totalCount` jumped 84 → 87. New items created post-recovery showed up immediately.

**Safe creation pattern**:

1. Create the issue and add it to the project (any path: `gh issue create` + `gh project item-add`, GraphQL `addProjectV2ItemById`, or web UI).
2. Set custom fields (Status/Points/Sprint) via `gh project item-edit` or the equivalent mutation.
3. **Verify visibility before reporting success.** Paginate through `ProjectV2.items` and confirm the new item ID is present:
   ```bash
   gh api graphql -f query='query { organization(login:"Atram-Inc") { projectV2(number:1) { items(first:100) { totalCount nodes { id } } } } }'
   ```
   If the item is missing, the index is lagging. Do **not** declare success to the user — flag it as eventually-consistent and either wait or schedule a follow-up check.

**Recovery for an item that's lagging**:

- Wait. The lag clears on its own, usually in seconds, occasionally in hours.
- Don't bother setting/clearing fields, deleting and recreating, or toggling project membership — none of those reliably nudge the index. They just churn IDs while the same lag continues.

## Points / story-point tracking

Story points are tracked via the project's **Points** custom field (a number field on `Atram-Inc/projects/1`). They are **not** tracked via `points:1` / `points:2` / `points:4` / `points:8` labels.

The label-based system was deprecated in 2026-04. AI agents must:

- Set the `Points` project field on the project item, never apply a `points:*` label.
- If you encounter a `points:*` label on a repo, treat it as legacy and do not propagate.

## Label deletion gotcha

A label cannot be safely "deleted" while any issue still has it applied. GitHub's UI permits the deletion, but the label definition reappears in the repo's label menu the next time an issue surfaces it.

To fully retire a label:

1. Find every issue that still uses it (`gh issue list --label <name> --state all --limit 1000`).
2. Remove the label from each one (`gh issue edit <num> --remove-label <name>`).
3. Then delete the label (`gh label delete <name> --yes`).

Skipping step 1–2 leaves zombie labels that pollute the label picker. This is documented in the issue raised for the org-wide `points:*` cleanup.
