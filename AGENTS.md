# AGENTS.md — Atram-Inc org notes for AI agents

Operational notes for AI agents working across Atram-Inc repos. Read before performing automated work on issues, projects, or labels.

## GitHub Projects v2 — invisible item gotcha

**Symptom**: An issue is added to the Engineering project (`Atram-Inc/projects/1`) via the API. Querying the project item by ID, or querying the issue's `projectItems`, both confirm it is on the project and not archived. But the item does **not** appear in the project's web UI (board or table), and `filterQuery` searches do not return it. The viewer cannot see Sprint or Points fields on the issue's project sidebar either — only Status.

**Diagnosis**: GitHub maintains separate indexes for the two directions of the `Issue ↔ ProjectV2Item` relationship.

| Direction | Source of truth |
|---|---|
| `Issue.projectItems` | Per-issue index — updates immediately on add |
| `ProjectV2.items` | Per-project index — updates lazily; powers the web UI |

When the project-side index lags, the item exists but is invisible everywhere except a direct `node(id: ...)` lookup. The issue's project sidebar also fails to render Sprint/Points pickers because it relies on the same lagging join.

**Reproduction** (encountered 2026-04-28):

- Created `Atram-Inc/.github#3` and added it to project 1 in rapid succession via `gh issue create` + `gh project item-add` + three `gh project item-edit` calls (Status, Points, Sprint).
- The Status edit succeeded; the Points and Sprint edits returned no error but the values never landed.
- The item was retrievable as `PVTI_lADOC2Ltt84BSRVnzgrLEzg` and listed under the issue's `projectItems`, but `ProjectV2.items` (totalCount=84) did **not** include it. Web UI: invisible.

**Safe creation pattern**:

1. Create the issue.
2. Add it to the project with `gh project item-add` and capture the item ID.
3. **Verify** the item appears in `ProjectV2.items` before setting any custom fields. Example:
   ```bash
   gh api graphql -f query='query { organization(login:"Atram-Inc") { projectV2(number:1) { items(first:100) { nodes { id } } } } }' \
     | grep "<item-id>"
   ```
   If the item is missing, do not proceed — the index is stale. Wait, or recreate.
4. Once the item is visible in the project-side index, set Status/Points/Sprint with `gh project item-edit`.
5. **Verify field values landed** by reading them back via `node(id: "...") { fieldValues }`.

**Recovery for an already-broken item**:

- The reliable fix is **delete and recreate** in the same repo (or, if the repo is suspect, a different one).
- Toggling fields or removing/re-adding to the project sometimes nudges the index, but is not guaranteed.

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
