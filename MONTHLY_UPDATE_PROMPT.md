# GKT Campaign Attribution Report — Monthly Update Prompt

Copy and paste this prompt into a new Claude Code session each month to regenerate the dashboard.

---

## PROMPT TO USE

```
I need to regenerate the GKT Campaign Attribution dashboard for [MONTH] [YEAR].
The dashboard file is at: /Users/aranzazu.caballero/Downloads/[FOLDER]/dashboard.html

## Data Sources (the ONLY two sources of truth)
- PostHog (signup events via MCP)
- HubSpot (contact creation, deals, original source via MCP)

## The 3 campaigns to track (exact UTM values):
- Meta B2C: GKT-BOFU-META-CV-US-B2C-AON
- Google DG B2C: GKT-BOFU-GG-CV-US-B2C-DG-AON
- Google Sembrand B2B: GKT-BOFU-GG-CV-US-B2B-sembrand-AON

## Step 1 — PostHog query (run via PostHog MCP)

Run this HogQL query replacing [YEAR-MM-01] and [YEAR-MM+1-01] with the month range:

SELECT
  person.properties.email AS email,
  person.properties.$initial_utm_campaign AS campaign_initial_utm,
  min(timestamp) AS signup_date_posthog
FROM events
WHERE event = '$identify'
  AND timestamp >= '[YEAR-MM-01]'
  AND timestamp < '[YEAR-MM+1-01]'
  AND properties.workspace_id IS NOT NULL
  AND person.properties.email IS NOT NULL
  AND person.properties.email != ''
  AND person.properties.email NOT LIKE 'geogatedctr%@gmail.com'
  AND person.properties.$initial_utm_campaign IN (
    'GKT-BOFU-META-CV-US-B2C-AON',
    'GKT-BOFU-GG-CV-US-B2C-DG-AON',
    'GKT-BOFU-GG-CV-US-B2B-sembrand-AON'
  )
GROUP BY person.properties.email, person.properties.$initial_utm_campaign
ORDER BY signup_date_posthog

Save result as: /tmp/posthog_sessions_[month][year].json

## Step 2 — HubSpot contact lookup (run via HubSpot MCP)

For each email from PostHog (batch in groups of 25):
- Search HubSpot contacts by email
- Get: contact ID, hs_createdate, hs_analytics_source, hs_analytics_source_data_1
- Classify:
  - NEW = hs_createdate >= [YEAR-MM-01]
  - RETARGETING = hs_createdate < [YEAR-MM-01]

## Step 3 — HubSpot deal lookup (run via HubSpot MCP)

For each contact that has deals, search HubSpot deals:
- Filter: dealcreatedate >= [YEAR-MM-01]
- Get: amount, hs_is_closed_won, dealname, pipeline
- Classify:
  - CW (Closed Won) = hs_is_closed_won = true
  - UC (Under Construction) = active deal, not closed

## Step 4 — Attribution logic

For each contact:
- Week assignment based on PostHog signup date:
  - W1 = day 1–8
  - W2 = day 9–15
  - W3 = day 16–22
  - W4 = day 23–last day of month
- Revenue:
  - cw = sum of all CW deal amounts for that contact
  - uc = sum of all UC deal amounts for that contact
  - hj = true if any deals found, false otherwise

## Step 5 — Update dashboard.html

Replace the CAMPS array in dashboard.html with the new data.
Keep the same structure: contacts[], sum{}, bw{} per campaign.
Update the header subtitle with the new month and cutoff date.
Update the week filter options with the correct month dates.

## Step 6 — Generate non-attributable list

Run this HogQL query for the same month:

SELECT
  coalesce(person.properties.$initial_utm_campaign, '(no UTM)') AS utm_campaign,
  count(DISTINCT person.properties.email) AS signups
FROM events
WHERE event = '$identify'
  AND timestamp >= '[YEAR-MM-01]'
  AND timestamp < '[YEAR-MM+1-01]'
  AND person.properties.email IS NOT NULL
  AND person.properties.email NOT LIKE 'geogatedctr%@gmail.com'
  AND (
    person.properties.$initial_utm_campaign IS NULL
    OR person.properties.$initial_utm_campaign NOT IN (
      'GKT-BOFU-META-CV-US-B2C-AON',
      'GKT-BOFU-GG-CV-US-B2C-DG-AON',
      'GKT-BOFU-GG-CV-US-B2B-sembrand-AON'
    )
  )
GROUP BY utm_campaign
ORDER BY signups DESC

Save as: non_attributable_[month][year].md (table format)
Flag any UTMs that look like GKT typo variants.

## Step 7 — Export validation file

Create march2026_validation_export.json → [month][year]_validation_export.json
Columns: campaign, email, hs_createdate, posthog_signup_date, attribution, week,
         has_job, original_source, total_revenue, closed_won, under_construction

The user will review this file line by line in HubSpot to validate CW/UC status.
```

---

## CLASSIFICATION RULES (do not change)

| Rule | Logic |
|---|---|
| Include in report | Must have PostHog $identify in target month + workspace_id NOT NULL |
| NEW | HubSpot hs_createdate ≥ first day of month |
| RETARGETING | HubSpot hs_createdate < first day of month |
| Closed Won (CW) | HubSpot deal: hs_is_closed_won = true + dealcreatedate ≥ first day of month |
| Under Construction (UC) | HubSpot deal: active (not closed) + dealcreatedate ≥ first day of month |
| No Job | No HubSpot deal found in target month |

## THINGS TO WATCH FOR

- **UTM typos**: Check non-attributable list for variants like `semnb` (vs `sembrand`) or truncated UTMs
- **Duplicate emails**: Same email in multiple campaigns → appears in each campaign separately
- **Renewals**: Without GoWork, cannot auto-classify renewals — user must validate in HubSpot
- **Pre-month deals**: Only count deals with dealcreatedate ≥ first day of month
- **AJ Poncin pattern**: If a contact's deal doesn't surface in the deal search but you know from HubSpot it exists, hard-code it with a note

## FILES TO UPDATE EACH MONTH

| File | What to update |
|---|---|
| `dashboard.html` | CAMPS array, header subtitle, week filter dates |
| `[month]_validation_export.json` | Full contact list for HubSpot validation |
| `non_attributable_[month].md` | Non-attributable signups breakdown |

## PIPELINE IDs (HubSpot reference)

| Pipeline | ID | Description |
|---|---|---|
| Sales Factory - Taxfyle | `830778483` | Main B2B pipeline |
| POS - Empower | `830764654` | Empower partner |
| Sales Factory - RocketTax | `828916844` | RocketTax partner |
| Sales Factory - Berkshire MM | `857602659` | Berkshire partner |
| B2C - Renewals | `667782980` | Renewal pipeline |

---

*Last updated: April 2026 | Methodology owner: GKT Team*
