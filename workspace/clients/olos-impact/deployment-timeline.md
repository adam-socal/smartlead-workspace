# Deployment timeline — olos-impact campaigns

**Campaigns:** oz-churches-2026-q3 (primary) → la-churches-2026-q2 (follow-up)
**Last updated:** 2026-05-07

---

## Now → May 9 — Inbox warmup finishing

- [ ] Confirm `app.olosimpact.com/apply` is live and functional (name, church, email capture at minimum — mobile-friendly)
- [ ] Verify all 25 inboxes are warming in Smartlead dashboard — warmup completes ~May 9

---

## May 9–30 — Leads + staging

- [ ] Source church lead list — 3,000–4,000 land-owning churches in LA + Ventura County (Apollo, Clay, or similar)
- [ ] Drop raw CSV into `oz-churches-2026-q3/` as `leads-raw.csv` and share column headers — mapping script will be generated
- [ ] Run stage-leads process: column mapping → email validation → deduplication → sender assignment
- [ ] Split leads across 3 senders (Chris / Tarik / Derrick) — ~1,000–1,300 per sender
- [ ] Confirm apply page handles inbound volume before deploy

---

## June 1 — OZ campaign deploy

- [ ] Run `create-campaign` for Chris Montes (6 inboxes — olosimpactrealestate.com + olosimpacthousing.com)
- [ ] Run `create-campaign` for Tarik Black (3 inboxes — olosimpactpartners.com + olosimpacthousing.com + olosimpactrealestate.com)
- [ ] Run `create-campaign` for Derrick Kelsey (2 inboxes — olosimpacthousing.com + olosimpactrealestate.com)
- [ ] Verify in Smartlead dashboard: 3 campaigns, all ACTIVE, 4 email steps each, no manual steps
- [ ] Confirm daily send totals: 11 inboxes × 40–50/day = 440–550 emails/day

**Hard deadline:** All first touches must send before July 1 nomination window opens. At 440–550/day, first touch through 3,000–4,000 leads takes ~7–9 days. Deploy by June 1 = first touch complete by ~June 10. ✅

---

## ~June 14 — OZ sequences complete

- [ ] Pull non-responders from all 3 OZ campaigns (anyone who completed all 4 steps without replying)
- [ ] Stage those leads → `la-churches-2026-q2/leads.csv`
- [ ] Deduplicate against any OZ replies already in pipeline

---

## Late June — Housing campaign deploy

- [ ] Run `create-campaign` for Chris Montes (9 inboxes — olosimpactpartners.com + olosimpactcommunity.com + olosimpactmission.com)
- [ ] Run `create-campaign` for Tarik Black (2 inboxes — olosimpactcommunity.com + olosimpactmission.com)
- [ ] Run `create-campaign` for Derrick Kelsey (3 inboxes — olosimpactpartners.com + olosimpactcommunity.com + olosimpactmission.com)
- [ ] Verify: 3 campaigns ACTIVE, no manual steps
- [ ] Confirm daily send totals: 14 inboxes × 40–50/day = 560–700 emails/day

---

## Open items / future improvements

- **Timezone-based send scheduling (not yet built):** Smartlead supports sending emails at optimal local time per recipient. Current `create-campaign` code only sets a single region timezone (`US`/`EU`/`IT`) for the whole campaign — it does not schedule by individual lead location. For the current LA/Ventura campaign this is fine (all leads are PDT). If campaigns expand to mixed-timezone lead lists (e.g. EST + PDT in the same campaign), the deploy step will need to either (a) split leads into timezone-segmented campaigns, or (b) use Smartlead's per-lead timezone API field if supported. Do not change the code until ready to tackle this.

---

## Rules — do not skip

- Never send both campaigns to the same contact simultaneously — OZ goes first, housing gets non-responders only
- Never send to a contact currently in an active OZ sequence — wait for the sequence to complete
- No manual steps in any sequence — if Smartlead shows a task queue building up, something is misconfigured
- Confirm `app.olosimpact.com/apply` is live before June 1 — all OZ emails link to it
