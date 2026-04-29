# Atram-Inc Org Configuration

This repo contains org-wide GitHub configuration for Atram.

## Issue Templates

When you create a new issue in any Atram repo, you'll see these templates:

- **Feature** — new capabilities, integrations, screens, workflows
- **Bug Report** — something broken or behaving unexpectedly
- **Chore / Infrastructure** — CI/CD, tooling, deps, config, DevOps
- **ETL / Data Pipeline** — weather data, forecasts, data ingestion, exports

### Writing good tickets

Use our [AI Prompt for Writing Tickets](https://linear.app/atram/document/engineering-ticket-templates-agent-ready-standard-ed4030f0e9e7) — paste it into Claude or ChatGPT along with your template to get help filling in every field.

## Pointing System

We use a Fibonacci-based point scale to estimate effort on issues:

| Points | Effort |
|--------|--------|
| 1 | Less than a day |
| 2 | 1 full day (accounting for other work) |
| 4 | ~2 days |
| 8 | Full sprint — no other tickets |

Set the **Points** field on the issue's Engineering project item when triaging or picking it up. The legacy `points:*` labels were retired in 2026-04 and should not be re-added — see [`AGENTS.md`](AGENTS.md#label-deletion-gotcha) for context.
