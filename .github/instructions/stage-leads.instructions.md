---
applyTo: "**"
---

# Stage Leads

Phase 2b of the outbound system. Takes a raw CSV from any lead source (Apollo, Clay, Prospeo export, scraped data, etc.), normalizes it to the Smartlead column schema, deduplicates across campaigns, and writes a clean `leads.csv` ready for upload.

Run this phase whenever the user drops a CSV into the campaign folder rather than pulling leads via Smart Prospects or Prospeo directly.

---

## Inputs

1. **Raw CSV path** — user drops their file into the campaign folder, typically named `leads-raw.csv`
2. **Campaign folder path** — `workspace/clients/{client}/campaigns/{slug}/`
3. **Other active campaign folders** — to deduplicate against (e.g. if OZ campaign is running, deduplicate housing leads against it)

---

## Step 1 — Inspect the raw file

```bash
head -3 "workspace/clients/{client}/campaigns/{slug}/leads-raw.csv"
```

Read the column headers and report them to the user. Then map each column to the Smartlead schema below.

### Smartlead column schema

| Column             | Required | Notes                                                |
| ------------------ | -------- | ---------------------------------------------------- |
| `email`            | ✅       | Primary key — rows without a valid email are dropped |
| `first_name`       | ✅       | Used in `{{first_name}}` token                       |
| `last_name`        | ✅       |                                                      |
| `company_name`     | ✅       | Used in `{{company_name}}` token                     |
| `phone_number`     | optional |                                                      |
| `website`          | optional |                                                      |
| `linkedin_profile` | optional |                                                      |
| `company_url`      | optional |                                                      |
| `location`         | optional |                                                      |

Any column in the raw file that doesn't map to the above becomes a **custom field** — it is preserved in `custom_fields` during CLI upload, not dropped.

---

## Step 2 — Write and run the mapping script

Generate a Python script tailored to the exact column names in the raw file. Save it as `map-leads.py` in the campaign folder.

```python
import csv, json, re

INPUT  = "leads-raw.csv"
OUTPUT = "leads.csv"

# Column mapping — adjust left side to match raw file headers exactly
COLUMN_MAP = {
    "Email":        "email",
    "First Name":   "first_name",
    "Last Name":    "last_name",
    "Company":      "company_name",
    "Phone":        "phone_number",
    "Website":      "website",
    "LinkedIn URL": "linkedin_profile",
    "Company URL":  "company_url",
    "Location":     "location",
}

STANDARD = set(COLUMN_MAP.values())

EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

rows_in  = 0
rows_out = 0
dropped  = []

with open(INPUT, newline="", encoding="utf-8-sig") as f_in, \
     open(OUTPUT, "w", newline="", encoding="utf-8") as f_out:

    reader = csv.DictReader(f_in)
    fieldnames = list(STANDARD) + ["custom_fields"]
    writer = csv.DictWriter(f_out, fieldnames=fieldnames, extrasaction="ignore")
    writer.writeheader()

    for row in reader:
        rows_in += 1
        mapped = {}

        # Apply column map
        for raw_col, std_col in COLUMN_MAP.items():
            val = row.get(raw_col, "").strip()
            if val:
                mapped[std_col] = val

        # Drop rows with no valid email
        email = mapped.get("email", "")
        if not EMAIL_RE.match(email):
            dropped.append({"row": rows_in, "reason": "invalid or missing email", "raw": dict(row)})
            continue

        # Capture unmapped columns as custom fields
        mapped_raw_cols = set(COLUMN_MAP.keys())
        custom = {k: v.strip() for k, v in row.items() if k not in mapped_raw_cols and v.strip()}
        if custom:
            mapped["custom_fields"] = json.dumps(custom)

        # Fill missing standard fields with empty string so CSV is consistent
        for col in STANDARD:
            mapped.setdefault(col, "")

        writer.writerow(mapped)
        rows_out += 1

print(f"Input rows : {rows_in}")
print(f"Output rows: {rows_out}")
print(f"Dropped    : {len(dropped)}")
if dropped:
    print("Dropped details:")
    for d in dropped:
        print(f"  Row {d['row']}: {d['reason']} — {d['raw'].get('Email', d['raw'])}")
```

Run it:

```bash
cd "workspace/clients/{client}/campaigns/{slug}"
python3 map-leads.py
```

Report the output counts to the user. If more than 5% of rows were dropped, flag it before continuing.

---

## Step 3 — Deduplicate across campaigns

If the user is running multiple campaigns targeting the same audience (e.g. OZ + housing for the same church list), deduplicate by email address so no contact appears in more than one active campaign.

```python
import csv

# List all other leads.csv files to deduplicate against
OTHER_CAMPAIGNS = [
    "workspace/clients/olos-impact/campaigns/oz-churches-2026-q3/leads.csv",
    # add more paths as needed
]

seen_emails = set()
for path in OTHER_CAMPAIGNS:
    try:
        with open(path, newline="", encoding="utf-8") as f:
            for row in csv.DictReader(f):
                email = row.get("email", "").strip().lower()
                if email:
                    seen_emails.add(email)
    except FileNotFoundError:
        pass  # campaign not yet populated

INPUT  = "workspace/clients/{client}/campaigns/{slug}/leads.csv"
OUTPUT = "workspace/clients/{client}/campaigns/{slug}/leads.csv"  # overwrite in place

rows = []
dupes = []
with open(INPUT, newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    fieldnames = reader.fieldnames
    for row in reader:
        email = row.get("email", "").strip().lower()
        if email in seen_emails:
            dupes.append(email)
        else:
            rows.append(row)
            seen_emails.add(email)

with open(OUTPUT, "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(rows)

print(f"Kept  : {len(rows)}")
print(f"Dupes removed: {len(dupes)}")
```

Run it:

```bash
python3 dedup-leads.py
```

---

## Step 4 — Assign senders (multi-sender campaigns)

If the campaign has more than one sender (e.g. Chris and Tarik), split the deduplicated leads evenly and add a `sender_assignment` column. The CLI upload phase uses this to route leads to the correct Smartlead campaign variant.

```python
import csv

INPUT = "leads.csv"
SENDERS = ["chris-montes", "tarik-black"]  # adjust to match campaign senders

rows = []
with open(INPUT, newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    fieldnames = reader.fieldnames + ["sender_assignment"]
    rows = list(reader)

for i, row in enumerate(rows):
    row["sender_assignment"] = SENDERS[i % len(SENDERS)]

with open(INPUT, "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(rows)

print(f"Assigned {len(rows)} leads across {len(SENDERS)} senders")
for s in SENDERS:
    count = sum(1 for r in rows if r["sender_assignment"] == s)
    print(f"  {s}: {count}")
```

---

## Step 5 — Final check

```bash
wc -l leads.csv
head -3 leads.csv
```

Confirm:

- Row count matches expectations
- `email`, `first_name`, `last_name`, `company_name` are populated
- No obviously malformed rows

Report final counts to the user and proceed to Phase 3 (COPY) or Phase 4 (DEPLOY) depending on where they are in the workflow.

---

## Notes

- Save `map-leads.py` and `dedup-leads.py` in the campaign folder. They become the repeatable import scripts for subsequent lead batches from the same source.
- Never overwrite `leads-raw.csv` — it's the original source of truth.
- If the user's source format changes (different export from Apollo, new columns from Clay), update `map-leads.py` rather than creating a new script.
