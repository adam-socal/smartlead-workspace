---
applyTo: "**"
---

# Create Campaign

Phase 4 of the outbound system. Takes a campaign folder that already has `copy.md` and `leads.csv`, and ships it to Smartlead.

## CLI commands used

- `smartlead campaigns create --name "..."`
- `smartlead campaigns save-sequence --id ID --from-json sequences.json`
- `smartlead mailboxes list` and `smartlead mailboxes add-to-campaign`
- `smartlead leads add --campaign-id ID --from-json leads.json`
- `smartlead campaigns set-status --id ID --status START`
- `smartlead campaigns list` (verification)

## Inputs

1. **Campaign folder path** — `workspace/clients/{client}/campaigns/{slug}/`
2. **Region** — for the schedule timezone (`US`, `EU`, or `IT` — defaults to `US`)

The folder must already contain `copy.md` and `leads.csv`. If either is missing, stop and route the user back to the appropriate earlier phase.

## Pre-flight checks

```bash
cd "workspace/clients/{client}/campaigns/{slug}"

[ -s copy.md ]    || { echo "missing copy.md"; exit 1; }
[ -s leads.csv ]  || { echo "missing leads.csv"; exit 1; }

smartlead mailboxes list --warmup-status ACTIVE --format json | python3 -c "import sys,json; print(len(json.load(sys.stdin)))"
```

If no warm mailboxes exist, stop and tell the user. Don't create a campaign that has nothing to send from.

## Workflow

### Step 1 — Create the campaign shell

```bash
SLUG="{slug}"
CAMPAIGN_ID=$(smartlead campaigns create --name "$SLUG" --format json | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "campaign id: $CAMPAIGN_ID"
```

### Step 2 — Convert copy.md → sequences.json and save

The CLI takes JSON. Convert the copy.md content to this shape:

```json
{
  "sequences": [
    {
      "seq_number": 1,
      "seq_delay_details": { "delay_in_days": 0 },
      "variant_distribution_type": "MANUAL_EQUAL",
      "seq_variants": [
        {
          "subject": "quick note about {{company_name}}",
          "email_body": "<p>Hi {{first_name}},</p><p>...</p>",
          "variant_label": "A"
        }
      ]
    },
    {
      "seq_number": 2,
      "seq_delay_details": { "delay_in_days": 3 },
      "subject": "",
      "email_body": "<p>Hi {{first_name}},</p><p>bumping this...</p>"
    }
  ]
}
```

Rules:
- One block per step in `copy.md`.
- `delay_in_days` is day-over-prior-step (step 1=0, step 2=3, step 3=4 if the schedule says Day 0/3/7).
- Wrap each line of the body in `<p>...</p>`. Smartlead renders HTML.
- Empty subject on follow-up steps auto-threads.
- Keep `{{first_name}}`, `{{last_name}}`, `{{company_name}}` tokens as-is.

```bash
smartlead campaigns save-sequence \
  --id "$CAMPAIGN_ID" \
  --from-json sequences.json
```

### Step 3 — Attach warm mailboxes

```bash
MAILBOX_IDS=$(smartlead mailboxes list --warmup-status ACTIVE --format json | python3 -c "import sys,json; print(' '.join(str(m['id']) for m in json.load(sys.stdin)))")

smartlead mailboxes add-to-campaign \
  --campaign-id "$CAMPAIGN_ID" \
  --account-ids $MAILBOX_IDS
```

### Step 4 — Convert leads.csv → leads.json and upload

**The top-level shape must be `{"lead_list": [...]}` — a bare array is rejected with `400 "value" must be of type object`.**

```bash
python3 -c "
import csv, json
rows = list(csv.DictReader(open('leads.csv')))
out = []
for r in rows:
    if not r.get('email'): continue
    standard = {'email','first_name','last_name','company_name','phone_number','website','linkedin_profile','company_url','location'}
    custom = {k:v for k,v in r.items() if k not in standard and v}
    lead = {k:r[k] for k in standard if r.get(k)}
    if custom: lead['custom_fields'] = custom
    out.append(lead)
json.dump({'lead_list': out}, open('leads.json','w'), indent=2)
print(f'wrote {len(out)} leads')
"

smartlead leads add \
  --campaign-id "$CAMPAIGN_ID" \
  --from-json leads.json
```

### Step 5 — Configure schedule and start

```bash
# Open the campaign in the Smartlead UI to confirm schedule, daily volume, etc.
echo "https://app.smartlead.ai/app/campaigns/$CAMPAIGN_ID"

# Once the schedule looks right:
smartlead campaigns set-status --id "$CAMPAIGN_ID" --status START
```

### Step 6 — Verify and record

```bash
smartlead campaigns list | grep "$SLUG"
smartlead stats campaign --id "$CAMPAIGN_ID"
```

Then write `campaign-status.md`:

```markdown
# Campaign status — {slug}

- **Smartlead campaign ID:** {CAMPAIGN_ID}
- **Created:** {YYYY-MM-DD}
- **Started:** {YYYY-MM-DD}
- **Region:** {US/EU/IT}
- **Mailboxes attached:** {N}
- **Leads uploaded:** {N}
- **State:** ACTIVE

## Notes

{Anything weird that happened during deploy: CSV cleaning, mailbox warmup gaps, schedule overrides.}
```

### Step 7 — Tidy

```bash
rm sequences.json leads.json
```

## Common pitfalls

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `save-sequence` rejects body | HTML not wrapped in `<p>` tags | Re-render markdown to HTML before building sequences.json |
| Lead upload reports rows skipped | Missing email or token mismatch | Check `leads.csv` for blanks; ensure tokens used in copy exist as columns |
| Campaign starts but nothing sends | No mailboxes attached, or all mailboxes still warming | `mailboxes list --warmup-status ACTIVE` and re-attach |
| Bounce rate spikes day 1 | Skipped verification on `find-leads` | Pause campaign, re-run with `verified_emails_only: true` |

## What this instruction does NOT do

- Generate copy. Copy is a Phase 3 artifact. Garbage copy will pass deploy and tank in the inbox.
- Source leads. That's `find-leads`.
- A/B variants. The sequence JSON above is single-variant for simplicity. Extend `seq_variants` if needed.
