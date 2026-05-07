---
applyTo: "**"
---

# Prospeo

End-to-end instructions for the Prospeo B2B contact API. Drives the in-repo Prospeo CLI at `.claude/skills/prospeo/scripts/prospeo.mjs` (zero deps, native Node 18+ fetch).

## At a glance

| You want to… | Run |
| --- | --- |
| Check plan + remaining credits (free) | `node .claude/skills/prospeo/scripts/prospeo.mjs account` |
| Search persons by ICP filters | `prospeo.mjs search-person --from-json filters.json --page 1 --output page1.json` |
| Search companies | `prospeo.mjs search-company --from-json filters.json --page 1` |
| Enrich one person → verified email | `prospeo.mjs enrich-person --from-json one.json --verified-email` |
| Enrich up to 50 people in one call | `prospeo.mjs bulk-enrich-person --from-json batch.json --verified-email` |
| Run search → bulk-enrich → leads.csv | `prospeo.mjs find-leads --from-json filters.json --target 200 --output leads.csv` |

Full help: `node .claude/skills/prospeo/scripts/prospeo.mjs help`

---

## Setup (first run only — skip if already done)

You're set up if all three are true:

1. `node --version` is **v18.x.x or higher**
2. `.env` in the project root contains a real `PROSPEO_API_KEY=...`
3. `node .claude/skills/prospeo/scripts/prospeo.mjs account` returns `error: false` and a `current_plan` field

### S1. Node 18+

```bash
node --version
```

If under 18 or not found, stop and tell the user to download from https://nodejs.org (LTS).

### S2. CLI executable

```bash
chmod +x .claude/skills/prospeo/scripts/prospeo.mjs
node .claude/skills/prospeo/scripts/prospeo.mjs help
```

### S3. API key

If `.env` doesn't exist:

```bash
cp .env.example .env
```

Send this message to the user:

> Now I need your Prospeo API key. Here's exactly where to find it:
>
> 1. Open https://app.prospeo.io in your browser and log in.
> 2. Click your account / avatar (usually top-right) and open **Settings**.
> 3. Open the **API** tab (sometimes labelled "API Access" or "Integrations").
> 4. Click **Generate API Key** if there isn't one yet, or copy the existing one.
> 5. Paste it here and send.

When the user pastes the key:
- Sanity-check it looks like an API key (long string, no spaces, ≥~20 chars).
- **Do not echo the key back in chat.**
- Edit `.env`, replacing the `PROSPEO_API_KEY=your-key-here` line with the real value.

### S4. Verify

```bash
node .claude/skills/prospeo/scripts/prospeo.mjs account
```

Expected response includes `"error": false` and `current_plan`. If you get `INVALID_API_KEY` (HTTP 401), the key is wrong — re-do S3.

Tell the user: "You're set. **{remaining_credits}** Prospeo credits available on the **{current_plan}** plan."

---

## CLI command reference

### `account`

Free. Returns plan + credit balance.

```bash
node .claude/skills/prospeo/scripts/prospeo.mjs account
```

### `search-person` and `search-company`

Filter-based search. Page size is fixed at **25 results**, max **1000 pages**. Costs **1 credit per page** when the page returns results.

```json
{
  "filters": {
    "person_seniority": { "include": ["C-Suite", "Vice President"] },
    "company_industry": { "include": ["Software Development"] },
    "company_headcount_range": ["51-100", "101-200"]
  }
}
```

```bash
node .claude/skills/prospeo/scripts/prospeo.mjs search-person --from-json filters.json --page 1 --output page1.json
```

**Hard rule:** at least one filter must use `include` / positive selection. Pure-exclude searches return `INVALID_FILTERS`.

**What's returned:** identity + employment fields only. **Email is NOT returned by search** — chain `bulk-enrich-person` or use `find-leads`.

**Before writing enum values, check `.claude/skills/prospeo/reference/` for canonical strings.** Every Prospeo enum is mirrored there as a flat-text file. Common surprises: `"Software Development"` not `"Software"`, `"Founder/Owner"` with a slash, headcount bands like `"51-100"` not `"51-200"`.

Useful filter keys:

| Filter | Shape | Reference file |
| --- | --- | --- |
| `person_seniority` | `{include: [...]}` | `reference/seniorities.txt` |
| `person_job_title` | `{include: [...], match_only_exact_job_titles: false}` | free-text, validate via Prospeo Search Suggestions API |
| `person_department` | `{include: [...]}` | `reference/departments.txt` (Normal Departments section) |
| `person_location_search` | `{include: [...]}` | free-text location strings |
| `company_industry` | `{include: [...]}` | `reference/industries.txt` |
| `company_headcount_range` | Array of band strings | `reference/employee-ranges.txt` |
| `company_funding` | `{stage, funding_date, last_funding, total_funding}` | `reference/funding-stages.txt` |
| `company_technology` | `{include: [...]}` | `reference/technologies.txt` |
| `company_naics` | `{include: [int]}` | `reference/naics-codes.txt` |
| `company_sics` | `{include: [int]}` | `reference/sic-codes.txt` |
| `company_headcount_growth` | `{timeframe_month, min, max, departments}` | `reference/departments.txt` (Headcount Growth Departments section — different from Normal Departments) |

### `enrich-person` and `bulk-enrich-person`

Looks up a person and returns identity + employment + (optionally) verified email + (optionally) mobile. Costs **1 credit per match**, **10 credits per match if `--mobile`**. No charge if no match. Re-enriching the same record is free for account lifetime.

Accepted input identifiers (any one is enough):
- `linkedin_url`
- `email` (reverse-enrichment)
- `person_id` from a search result
- `first_name + last_name + (company_name | company_website | company_linkedin_url)`

Single record:

```json
{
  "data": {
    "first_name": "Eva",
    "last_name": "Kiegler",
    "company_website": "intercom.com"
  }
}
```

```bash
node .claude/skills/prospeo/scripts/prospeo.mjs enrich-person --from-json one.json --verified-email
```

Bulk (up to **50** records per call):

```json
{
  "data": [
    { "linkedin_url": "https://linkedin.com/in/...", "identifier": "lead-1" },
    { "person_id": "abc123", "identifier": "lead-2" },
    { "first_name": "Sara", "last_name": "Lin", "company_website": "acme.com", "identifier": "lead-3" }
  ]
}
```

```bash
node .claude/skills/prospeo/scripts/prospeo.mjs bulk-enrich-person --from-json batch.json --verified-email
```

Always include `identifier` per record to correlate input → output.

**Keep `--verified-email` on by default** — it makes unverified hits unbillable. Add `--mobile` only when the campaign needs phone numbers.

### `find-leads` (high-level pipeline)

Runs `search-person` paginated → `bulk-enrich-person` chunked at 50 → CSV in one shot.

```bash
node .claude/skills/prospeo/scripts/prospeo.mjs find-leads \
  --from-json filters.json \
  --target 200 \
  --output leads.csv
```

Output CSV columns: `email, first_name, last_name, full_name, job_title, company_name, company_website, company_domain, linkedin_url, country, city`.

---

## Cost & rate-limit notes

| Action | Cost |
| --- | --- |
| `account` | free |
| `search-person` / `search-company` | 1 credit per page (25 results); free if identical query within 30 days |
| `enrich-person` / `enrich-company` | 1 credit per match; free if no match; free re-enrichment lifetime |
| `enrich-person --mobile` | **10 credits per match** |

On `429` the CLI prints how long until the limit resets and exits with code 2.

---

## Common errors

| `error_code` | HTTP | Cause | Fix |
| --- | --- | --- | --- |
| `INVALID_API_KEY` | 401 | Missing / wrong `PROSPEO_API_KEY` | Re-do Setup S3 |
| `INVALID_FILTERS` | 400 | Enum value mismatch — `filter_error` names the offending value | Fix value to match canonical enum in `reference/` |
| `INVALID_DATAPOINTS` | 400 | Enrich input doesn't satisfy any valid identifier combination | Add another identifier (linkedin_url or company_website) |
| `INSUFFICIENT_CREDITS` | 400 | Out of credits | Check `account`; upgrade plan or wait for renewal |
| `NO_MATCH` | 400 | No record found (single enrich) | Try a different identifier; unbilled |
| `RATE_LIMITED` | 429 | Per-minute or per-day limit hit | Wait `x-*-reset-seconds`, retry |
