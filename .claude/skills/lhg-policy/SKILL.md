---
name: lhg-policy
description: Single source of truth for all LHG segmentation rules and policy. Load when reviewing segmentation logic OR when any rule about booking filters, cabin targeting, web events, airline codes, or table selection is needed. Also contains the review checklist used to gate segment creation.
---

## NON-NEGOTIABLE RULES — apply to EVERY audience

(These are hard constraints. Never produce an audience that violates them.)

1. Booking-based audiences MUST filter on `res_status_categ_cd = 'K'` (confirmed/ticketed)
2. Newsletter (NL) audiences MUST include the NL permission attribute for the requesting airline
3. ALL audience filters MUST live in the SAME container
4. Express every recency/date window in DAYS — never "months" or "years"
5. When targeting by airline, combine `operating_airline OR marketing_airline`
6. When targeting by cabin class, combine `oper_comp_cd OR mkt_comp_cd`
7. At Flight_Leg level, use COUNT = 0 for exclusions — NEVER COUNT_DISTINCT = 0
8. Every audience MUST have a clear, human-readable description
9. Each web brand-family bucket must be self-contained; combine buckets with OR at the rule level

---

## Section 1: PERMISSIONS & CONSENT [MANDATORY]

### 1.1 Newsletter campaigns
- Include NL permission attribute for the requesting airline
- Use airline-specific permission code matching the campaign's target airline
- Fields: permission_code, airline_cd
- DCP email pattern: `{airline}_dcp_email_ind = 'Y' AND email IS NOT NULL`
  Replace {airline} with: lh, lx, os, sn, ew

### 1.2 Confirmed tickets filter
- Always filter: `res_status_categ_cd = 'K'`
- This implies the passenger flew the flight
- Never build a booking-based audience without this filter

---

## Section 2: TABLE & ATTRIBUTE SELECTION [SCHEMA]

### 2.1 When to use each table

**Bookings table (`behavior_pnr`) — booking-level:**
- Trip reason, origin & destination of booking, booking status
- Aggregation: COUNT

**Flight_Leg table (`behavior_pnr_leg`) — flight leg level:**
- Cabin, leg destination, operating carrier, any attribute at individual leg level
- Aggregation: COUNT_DISTINCT by tvl_txn_id

**Ancillary table (`behavior_pnr_leg_anc`) — ancillary purchases:**
- Seat upgrades, bags, lounge access, ancillary revenue
- Aggregation: COUNT

Rule: whole booking attributes = Bookings (COUNT). Flight leg attributes = Flight_Leg (COUNT_DISTINCT by tvl_txn_id).
Prefer `behavior_pnr_leg` over `behavior_pnr` unless a booking-level attribute is specifically needed.

### 2.2 Origin & Destination field mapping
- `airpt` in column name = Bookings table; `airport` = Flight_Leg table
- Default for flight destination: `bkg_dest_cntry_cd` or `bkg_dest_airpt_cd`
- `por_cntry_cd` = country passenger flies BACK FROM (NOT the booking destination in open-jaw bookings) — NEVER use for destination targeting
- `dest_cntry_cd` and `orig_cntry_cd` (without `bkg_` prefix) = leg-level only
- Default to `bkg_dest_cntry_cd` for destination

### 2.3 Traffic area codes vs airport lists
- Use `dest_traffic_area_cd` (e.g. "North America") instead of listing every airport in a region
- ALWAYS verify ALL requested countries are part of the traffic area
- ALWAYS verify the traffic area does not include unwanted countries (e.g. "North America" includes Canada)
- Validate with:
  ```sql
  SELECT bkg_dest_cntry_cd, COUNT(*)
  FROM cdp_audience_1159510.behavior_pnr_leg
  WHERE dest_traffic_area_cd = '<area>'
  GROUP BY 1
  ORDER BY 2 DESC
  LIMIT 20
  ```

### 2.4 Flight_Leg exclusions
- For exclusion cases at leg level: use COUNT = 0
- NEVER use COUNT_DISTINCT = 0 for exclusions
- Reason: COUNT_DISTINCT = 0 does not reliably express "this leg condition is absent"

### 2.5 Pre-computed attributes vs behavior scans
Before building any behavior aggregation condition, check if a pre-computed attribute already covers the intent. Pre-computed attributes are instant; behavior scans over 730+ days are slow and increase push time.

| Intent | Pre-computed attribute | Avoids |
|--------|------------------------|--------|
| Top 10% spender | `decile_monetary_24m = 10` | 730-day revenue Sum scan |
| High-value customer | `clv_3Y_4pct > 500` or `segment_1y IN (5,6)` | Multi-year spend aggregation |
| Purchase propensity | `purchase_propensity_band = 'hot'` | Recency/frequency behavior counts |
| Elite traveler | `curr_mbr_status_cd IN ('hon','sen')` | Flight count + cabin aggregation |
| Corporate traveler | `corp_ind IN ('Y','y')` (note: lowercase 'y' exists, ~22% of corporate bookings) | corp_ind_staged behavior scan |

Always run `tdx ps fields "cdp_lhg_unified_first_party"` and check for pre-computed candidates before building behavior conditions.

---

## Section 3: SEGMENT RULE STRUCTURE [MANDATORY]

### 3.1 Boolean logic
- Use Composite/expr pattern — never raw `type: And` or `type: Or` as condition types (see segment-lhg for full syntax)
- For OR between groups, each group is a Composite inside the top-level Composite

### 3.2 Date & time periods — convert to DAYS
- Always express years, months, and dates as DAYS
- Conversion lookup:
  - 1 month = 30 days
  - 3 months = 90 days
  - 6 months = 180 days
  - 1 year = 365 days
  - 2 years = 730 days
  - 3 years = 1095 days
- Prefer columns with `_utc_` or `_utc` in name
- Default booking time column: `bkg_tms_utc`
- For specific calendar dates, convert to "days in the past from today"
- For a duration window (between two dates), use "in between the past"

### 3.3 Known operator bug
- The `is not_multiple` operator is NOT working in Audience Studio
- Workaround: use nested NOT conditions, or a separate exclusion audience using COUNT = 0 on the target attribute

### 3.4 Audience descriptions
- Every audience MUST have a clear human-readable description
- Include: segment logic, use case, owning campaign, stakeholder/owner, creation date

---

## Section 4: AIRLINE & CABIN CLASS TARGETING [MANDATORY]

### 4.1 Prefixes
- `mkt_*` = Marketing/Marketed — what was SOLD to the customer
- `oper_*` = Operating/Operated — what was actually FLOWN

### 4.2 Always combine operating OR marketing airline
- When targeting by airline, always include BOTH:
  `operating_airline = <code> OR marketing_airline = <code>`
- Reason: a ticket may be marketed by LX but operated by a partner carrier

### 4.3 Always combine operating OR marketing cabin class
- `oper_comp_cd = <class> OR mkt_comp_cd = <class>`
- Cabin class codes: C = Business, F = First, E = Premium Economy, M = Economy

### 4.4 Query pattern example
```
Bucket A — Operated Cabin Class is Business or First (Match logic = ALL):
  - oper_comp_cd is IN ("F", "C")
  - res_status_categ_cd is "K"
  - Count Distinct by tvl_txn_id >= 1

Bucket B — Marketed Cabin Class is Business or First (Match logic = ALL):
  - mkt_comp_cd is IN ("F", "C")
  - res_status_categ_cd is "K"
  - Count Distinct by tvl_txn_id >= 1

Combine Bucket A OR Bucket B
```

---

## Section 5: TEALIUM TAG & DATA FRESHNESS [CONTEXT]

### 5.1 Tealium whitelist cutoff — June 17, 2025
- Before this date: ALL web events tracked in full
- On/after this date: ONLY whitelisted events captured
- Pre-cutoff and post-cutoff data are NOT directly comparable
- When building behavioral audiences from web events that straddle this date, account for the discontinuity

### 5.2 Flight search category change — May 18, 2026
- Before May 18, 2026: `event_category_*` = "flightmanager"
- From May 18, 2026 onwards: `event_category_*` = "flma"
- Always include BOTH values: `IN ("flma", "flightmanager")` unless the timeframe is entirely after May 18, 2026 (then "flma" only)

---

## Section 6: WEB ATTRIBUTES & EVENT LOGIC [MANDATORY]

### 6.1 Container structure
- When given an origin/destination (including region or country), populate `origin_detail`/`destination_detail` with lowercase IATA codes as IN list
- Only use IS NOT NULL when the request is entirely destination-agnostic
- Keep all attributes of the SAME bucket together — never split one bucket's conditions across containers

### 6.2 Web Searcher audiences
- A "Web Searcher" = user who pressed the flight-search button on the website (different from merely visiting)
- Source table: `behavior_web_all_events_stitched`
- Brand column suffixes: `lh` (Lufthansa), `sn` (Brussels), `lx` (Swiss), `os` (Austrian)

### 6.3 Brand-specific web data patterns

**Single brand (e.g. Lufthansa only):**
```
origin_detail is <origin> / IS NOT NULL
destination_detail is <destination> / IS NOT NULL
event_action_lh_sn is "submit"
event_category_lh_sn is IN ("flma", "flightmanager")
source_website is "lufthansa"
Count >= 1
# Valid source_website values: "lufthansa", "swiss", "austrian", "brusselsairlines", "miles-and-more"
```

**Multi-brand / non-specific brand (always two buckets):**
```
Bucket A — LH/SN family (Match logic = ALL):
  origin_detail is <origin> / IS NOT NULL
  destination_detail is <destination> / IS NOT NULL
  event_action_lh_sn is "submit"
  event_category_lh_sn is IN ("flma", "flightmanager")
  Count >= 1

Bucket B — LX/OS family (Match logic = ALL):
  origin_detail is <origin> / IS NOT NULL
  destination_detail is <destination> / IS NOT NULL
  event_action_lx_os is "submit"
  event_category_lx_os is IN ("flma", "flightmanager")
  Count >= 1
```

Combine Bucket A OR Bucket B.

### 6.4 Adding search detail (time, traffic)
- Time: `event_timestamp_epoch` (Unix epoch) — express window in DAYS
- Traffic/market type: `traffic_type` (with "e") in web data; `traffic_typ` (without "e") in leg data
- Add matching conditions to EACH bucket individually

### 6.5 Web vs App data — critical distinction
- `tealium_event = "click booking search button"` — ONLY generated by app, never by web
- `event_action_*` and `event_category_*` — ONLY set by web data, never by app
- Only distinguish web vs app when the request explicitly requires it

### 6.6 Lowercase IATA in web data
- `origin_detail` and `destination_detail` use LOWERCASE IATA codes
- NEVER use uppercase or LIKE with city names
- Multi-airport cities require an IN list of all airport codes:
  - NYC: IN ('nyc', 'jfk', 'ewr', 'lga')
  - London: IN ('lon', 'lhr', 'lgw', 'stn', 'lcy', 'ltn')
  - Paris: IN ('par', 'cdg', 'ory')
  - Single-airport cities (fra, zrh, muc, vie, bru, bcn, etc.) use the city code directly

---

## Section 7: ADDITIONAL STANDING RULES

- Point of booking must NOT be used for segmentation
- Use LOCAL time for scheduled departure/arrival when activation depends on timing
- When a user requests "frequent flyers" or similar, require a minimum flight count threshold to be defined
- Do not segment on internal airline employee identifiers unless explicitly requested
- If targeting Miles & More members by tier, verify the membership tier attribute is present

---

## Section 8: ANCILLARY RULES [MANDATORY]

For full product codes, revenue hierarchy, and audience sizes — see `lhg-ancillary`.

### 8.1 Triple Confirmed Filter

Every ancillary query against `behavior_pnr_leg_anc` MUST apply all three filters:

```sql
WHERE res_status_categ_cd = 'K'
  AND coup_status_cd IN ('F', 'I', 'A')
  AND emd_coup_status_cd IN ('F', 'I', 'A')
```

Never query ancillaries with only `res_status_categ_cd = 'K'` — the other two are required.

### 8.2 Voluntary upgrades only

When targeting upgrade buyers, always add `voluntry_upg_ind = 'Y'` to exclude involuntary operational upgrades that were not purchased by the passenger.

### 8.3 Discontinued products — do not target

These product codes have zero volume and return empty audiences:
`OIS`, `CATE`, `UPC`, `UPF`, `A20`, `OCC`, `OCL`, `OCP`, `SAX`, `INS`

### 8.4 Dead NL permission columns — do not use

These newsletter permission columns have 0% opt-in rate and return empty audiences:
`en_nl_ind`, `jp_nl_ind`, `lg_nl_ind`, `lo_nl_ind`, `ou_nl_ind`

### 8.5 Email campaign check — always both conditions

Permission alone is not sufficient — email address may be NULL (bounced/suppressed):
```
{airline}_dcp_email_ind = 'Y' AND email IS NOT NULL
```
Replace `{airline}` with: `lh`, `lx`, `os`, `sn`, `ew`, `ew4y`

---

## Review Checklist

This section is the gate check before any segment is created. Apply every rule above and report only violations.

**If all criteria met and policy-compliant:**
> Segmentation approved. All user criteria fulfilled and policy-compliant.

**If violations found:**
> Segmentation requires updates:
> - 1.2: Missing res_status_categ_cd = 'K' filter on booking behavior
> - 4.2: Only operating airline used, must also include marketing airline
> - User: Missing the "last 30 days" time constraint from the request

**Format rules:**
- One bullet per violation
- Format: `<Section>: <short issue description>` OR `User: <missing user criterion>`
- Max 15 words per violation
- Report only violations — do not explain policy or suggest fixes
- Order by section number, user criteria last
