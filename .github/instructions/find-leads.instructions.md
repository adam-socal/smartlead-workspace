---
applyTo: "**"
---

# Find Leads

Phase 2 of the outbound system. Turns an ICP block into a `leads.csv` ready for upload.

Supports two providers. The **output is identical** regardless of provider — the campaign folder always ends up with one `leads.csv` whose columns conform to the output schema below.

## Providers

| Provider                      | CLI                                                   | Auth                | What it gives you                                                                              | Cost model                                                                          |
| ----------------------------- | ----------------------------------------------------- | ------------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **Smart Prospects** (default) | `smartlead prospect ...`                              | `SMARTLEAD_API_KEY` | Verified-email rows in one search call, simple filter shape                                    | Bundled with your Smartlead plan                                                    |
| **Prospeo**                   | `node .claude/skills/prospeo/scripts/prospeo.mjs ...` | `PROSPEO_API_KEY`   | 200M+ contacts, 30+ filters incl. funding stage, technology stack, headcount growth, NAICS/SIC | Per-credit: 1 credit/search-page (25 rows) + 1 credit/verified-email enrichment hit |

## When to use which

Default to **Smart Prospects**. Switch to **Prospeo** when any of the following are true:

- The ICP requires filters Smart Prospects doesn't expose (funding stage, last-funding-date, technology stack, hiring-velocity, NAICS/SIC, headcount-growth-by-department).
- The user explicitly asks for Prospeo.
- A previous Smart Prospects pass returned `< 100` rows for an ICP you have reason to believe is broader.
- The user is trying to teach the workflow with Prospeo specifically.

If neither side has a clear edge, ask the user. Don't pick silently.

---

## Inputs

1. **Campaign folder path** — `workspace/clients/{client}/campaigns/{slug}/`
2. **ICP filters** — pulled from `campaign-strategy.md` (job titles, seniority, company size, industry, geography, plus any provider-specific extras)
3. **Volume target** — how many leads to fetch (default 200)
4. **Provider** — `smartlead` or `prospeo`. If absent, follow the rule above.

## Step 0 — Read the strategy

```bash
cat "workspace/clients/{client}/campaigns/{slug}/campaign-strategy.md"
```

Extract the ICP block. If anything is vague ("decision-makers", "tech companies"), stop and push back to the orchestrator. Garbage in, garbage out.

If the strategy mentions a provider-specific signal — e.g. "Series B SaaS" (funding stage), "companies using Snowflake" (tech), "hiring 10+ engineers in last 90d" (hiring velocity) — Prospeo is the right call.

## Step 1 — Pick the provider

Either honor the orchestrator's choice, or ask the user with a clear default. Then verify the provider is set up:

| Provider        | Verify command                                                   | If it fails                             |
| --------------- | ---------------------------------------------------------------- | --------------------------------------- |
| Smart Prospects | `smartlead campaigns list`                                       | Follow `install-smartlead` instructions |
| Prospeo         | `node .claude/skills/prospeo/scripts/prospeo.mjs account` (free) | Follow `prospeo` setup instructions     |

Do not skip the verify call. If the API key is missing or wrong, stop here.

---

## Path A — Smart Prospects (Smartlead CLI)

### A1. Build the search payload

Write `search.json` inside the campaign folder:

```json
{
  "filters": {
    "job_titles": ["VP Marketing", "Head of Growth", "CMO"],
    "company_size": ["51-200", "201-500"],
    "industries": ["Software", "SaaS"],
    "countries": ["United States", "Canada"]
  },
  "page": 1,
  "per_page": 100,
  "verified_emails_only": true
}
```

Run `smartlead prospect search --help` once to confirm field names if anything looks off.

### A2. Run the search and write CSV

```bash
cd "workspace/clients/{client}/campaigns/{slug}"

smartlead prospect search \
  --from-json search.json \
  --format csv \
  > leads.csv
```

If the volume target exceeds one page, paginate with `"page": 2, 3, ...` and concatenate (skip the header on subsequent pages):

```bash
smartlead prospect search --from-json search-p1.json --format csv > leads.csv
smartlead prospect search --from-json search-p2.json --format csv | grep -v "^email" >> leads.csv
```

### A3. Clean up

```bash
rm search.json
```

Skip to [Step 2 — Verify](#step-2--verify-applies-to-both-paths).

---

## Path B — Prospeo (in-repo CLI)

Prospeo separates search from enrichment: the search call returns identity + employment fields but **no email**. The high-level `find-leads` command does the whole pipeline in one call.

### B1. Build the filters payload

Write `filters.json` inside the campaign folder:

```json
{
  "filters": {
    "person_seniority": { "include": ["C-Suite", "Vice President", "Head"] },
    "person_job_title": {
      "include": ["VP Marketing", "Head of Growth", "CMO"],
      "match_only_exact_job_titles": false
    },
    "person_location_search": { "include": ["United States", "Canada"] },
    "company_industry": { "include": ["Software Development"] },
    "company_headcount_range": ["51-100", "101-200", "201-500"],
    "person_contact_details": {
      "email": ["VERIFIED"],
      "operator": "AND"
    }
  }
}
```

**Before writing enum values, check `.claude/skills/prospeo/reference/` for the canonical strings.** Common surprises: `"Software Development"` not `"Software"`, `"Founder/Owner"` with a slash, headcount bands like `"51-100"` not `"51-200"`. Guessing costs an `INVALID_FILTERS` roundtrip.

Hard rules from the API:

- **At least one filter must use `include` / positive selection** — pure-exclude searches are rejected.
- **Page size is fixed at 25.** The CLI handles pagination internally.

### B2. Run the pipeline

```bash
cd "workspace/clients/{client}/campaigns/{slug}"

node ../../../../.claude/skills/prospeo/scripts/prospeo.mjs find-leads \
  --from-json filters.json \
  --target 200 \
  --output leads.csv
```

(Adjust the relative path depth based on where the campaign folder sits.)

The CLI prints a final summary line like:

> `Done. Wrote 187 leads to leads.csv. Credits spent: 8 (search) + 187 (enrich) = 195. Unmatched: 13.`

Warn the user before running with `--mobile` — it's 10x the credit cost.

### B3. Clean up

```bash
rm filters.json
```

---

## Step 2 — Verify (applies to both paths)

```bash
wc -l leads.csv
head -3 leads.csv
awk -F',' 'NR>1 && $1!=""' leads.csv | wc -l
```

If the row count is far below target, tell the orchestrator the number you got and stop. Don't pad with low-quality leads.

---

## Output schema

Both paths produce a CSV that, at minimum, contains:

| Column         | Required |
| -------------- | -------- |
| `email`        | yes      |
| `first_name`   | yes      |
| `last_name`    | yes      |
| `company_name` | yes      |

Additional columns are allowed. The `create-campaign` instructions handle any superset of the four required columns safely.
