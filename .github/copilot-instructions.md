# Smartlead Outbound Workspace

This workspace runs cold-email outbound campaigns end-to-end using the public **Smartlead CLI** and optionally the **Prospeo** B2B contact API.

## Mental model

Five instruction files, four phases, one folder per campaign.

| Instruction file    | Role                                                                                                                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `install-smartlead` | One-time setup. Installs the Smartlead CLI and walks through saving the API key.                                                                                                           |
| `outbound-system`   | Orchestrator. Walks the phases, owns the campaign folder convention.                                                                                                                       |
| `find-leads`        | Queries the lead provider (Smart Prospects or Prospeo) and writes `leads-raw.csv`.                                                                                                         |
| `stage-leads`       | Normalizes any raw CSV to the Smartlead schema, validates emails, deduplicates across campaigns, writes `leads.csv`. Use when bringing in leads from Apollo, Clay, or any external source. |
| `prospeo`           | Operational skill for the Prospeo B2B contact API — setup and all CLI commands.                                                                                                            |
| `create-campaign`   | Creates the Smartlead campaign, saves the sequence, uploads leads, starts sending.                                                                                                         |

Phases: **strategy → leads → copy → deploy**. Each writes one file into the campaign folder.

## Folder convention

```
workspace/clients/{client}/campaigns/{campaign-slug}/
├── campaign-strategy.md   # who, why, what we're betting on
├── leads.csv              # output of find-leads
├── copy.md                # the email sequence
├── campaign-status.md     # campaign id, lead count, start date, live state
└── campaign-analysis.md   # post-run learnings (filled after results)
```

One client = one folder under `workspace/clients/`. One campaign = one folder under that client's `campaigns/`. Slug is short kebab-case (e.g. `saas-founders-2026-q2`).

## Prerequisites

- Node 18+ (for the Smartlead CLI and Prospeo script)
- `.env` file in the project root with API keys (copy from `.env.example`)

**First-time setup (Smartlead):** Say **"install smartlead"** and follow the `install-smartlead` instructions.

**Optional setup (Prospeo):** Say **"set up prospeo"** if you want Prospeo as a lead source.

## How to run a campaign

Say: **"start a new outbound campaign for {client}"** and follow the `outbound-system` instructions.

## Lead-provider choice

Default to **Smartlead Smart Prospects**. Switch to **Prospeo** when the ICP requires filters Smart Prospects doesn't expose — funding stage, technology stack, hiring velocity, NAICS/SIC, headcount growth — or when explicitly requested.
