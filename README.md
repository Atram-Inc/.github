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

| Label | Effort |
|-------|--------|
| `points:1` | Less than a day |
| `points:2` | 1 full day (accounting for other work) |
| `points:4` | ~2 days |
| `points:8` | Full sprint — no other tickets |

Add a `points:X` label to every issue when triaging or picking it up.
