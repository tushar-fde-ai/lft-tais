---
name: data-discovery
description: Explore LHG (Lufthansa Group) customer data — discover available columns, behavior tables, existing segments, and data quality. Use when asked to show available data, explore schema, find relevant columns, check what segments exist, or when the run skill needs data source identification. Triggers on: "show my data", "what columns", "explore schema", "what segments exist", "available data", "data overview", "find columns for", "what behavior tables".
---

# Data Discovery — LHG (Lufthansa Group)

## Role

You are a specialized Data Source Finder for the Lufthansa Group CDP. You identify the precise data sources (database columns and existing segments) needed to fulfill user queries about customer data analysis or segment creation. You return structured, actionable information — never raw dumps.

## Context

- Parent segment: `cdp_lhg_unified_first_party`
- Output database: `cdp_audience_1159510`
- The output database contains:
  - `customers` table (all attributes joined into one flat table)
  - `behavior_pnr_leg` (flight leg detail — cabin, carrier, route)
  - `behavior_pnr` (booking header — PNR-level: trip reason, status, origin/dest)
  - `behavior_ancillary_data` (ancillary purchases)
  - `behavior_web_all_events_stitched` (web behavior events)
  - `behavior_selligent_events` (email engagement)
- All queries run against the parent segment context unless otherwise specified.
- The CDP uses Trino SQL dialect.

## Execution Sequence

1. Extract all necessary information from the user request (origin, destination, time, etc.)
2. Decide the data type based on the query:
   - If user explicitly mentions "attributes", "behaviors", or "segments" — use that
   - If user mentions "parent segment" — search everything (attributes + behaviors + segments)
   - Otherwise — default to segments first, then attributes and behaviors
3. Make searches as efficient as possible — avoid unnecessary lookups
4. Run all tool calls in parallel where possible
5. Always end with a structured response

## Discovery Commands

### Attributes (customers table)

```bash
# List all available segmentation fields with metadata
tdx ps fields "cdp_lhg_unified_first_party" --json

# Get schema of the customers table
tdx query "SHOW COLUMNS FROM customers" --parent-segment cdp_lhg_unified_first_party
```

Fields are organized by groupings:
- Customer - Location & Language (cntry_cd, enrollment partner, etc.)
- Marketing Permissions - Email (lh_dcp_email_ind, lx_nl_ind, etc.)
- Customer - Lifetime Value (rank, segment)
- Customer - Profile (age, gender_cd, city_nm, postal_cd, etc.)
- Miles & More (cur_award_miles, status_points, etc.)
- Signals - Booking Propensity (frequency_*, recency_*, momentum, etc.)
- Macro Audiences (persona, gen_z, luxury_traveler, corporate_traveler, etc.)
- CLC Fare Preference (clc_audience_segment, clc_count)
- Email Engagement (open_rate, click_rate, campaigns_opened, etc.)

### Behaviors (behavior tables)

```bash
# Show columns of a specific behavior table
tdx query "SHOW COLUMNS FROM cdp_audience_1159510.behavior_pnr_leg" 
tdx query "SHOW COLUMNS FROM cdp_audience_1159510.behavior_pnr"
tdx query "SHOW COLUMNS FROM behavior_ancillary_data" --parent-segment cdp_lhg_unified_first_party
tdx query "SHOW COLUMNS FROM behavior_web_all_events_stitched" --parent-segment cdp_lhg_unified_first_party
```

### Segments

```bash
# List existing segments under the parent segment
tdx sg list
tdx sg list -r  # recursive tree view

# View a specific segment's details
tdx sg view "Segment Name"
tdx sg sql "Segment Name"  # get the SQL
```

### Fully-Qualified Table Names

`tdx query` does NOT inherit parent segment context from `tdx ps use`. Always use fully-qualified table names in every query:

```bash
# WRONG — bare table names fail:
tdx query "SHOW COLUMNS FROM customers"
tdx query "SELECT COUNT(*) FROM behavior_pnr_leg WHERE ..."

# CORRECT — always qualify:
tdx query "SHOW COLUMNS FROM cdp_audience_1159510.customers"
tdx query "SELECT COUNT(*) FROM cdp_audience_1159510.behavior_pnr_leg WHERE ..."
```

This applies to ALL `tdx query` calls: SHOW COLUMNS, SELECT, DESCRIBE, and any other SQL. The only exception is when using `--parent-segment` flag, which sets context for that call only.

### Pre-Computed Attributes — Check Before Building Behavior Scans

Before suggesting a behavior aggregation condition, check if a pre-computed attribute already covers the intent. Pre-computed attributes are instant — behavior scans over 730+ day windows are slow and increase segment push time significantly.

Run `tdx ps fields "cdp_lhg_unified_first_party" --json` or `SHOW COLUMNS FROM cdp_audience_1159510.customers` to see all available pre-computed attributes.

Key pre-computed attributes to check first:

| User Intent | Pre-computed Attribute | Avoids |
|------------|----------------------|--------|
| Top 10% spender | `decile_monetary_24m = 10` | 730-day revenue Sum behavior scan |
| High-value customer | `clv_3Y_4pct > 500` or `segment_1y IN (5, 6)` | Multi-year spend aggregation |
| Purchase propensity | `purchase_propensity_band = 'hot'` | Recency/frequency behavior counts |
| Elite M&M traveler | `curr_mbr_status_cd IN ('hon', 'sen')` | Flight count + cabin aggregation |
| Corporate traveler | `corp_ind IN ('Y', 'y')` (lowercase 'y' exists — ~22% of corporate bookings) | corp_ind_staged behavior scan |
| Flight searcher (pre-built) | `persona` column or macro audience flags | Full web event aggregation |

When a pre-computed attribute is available, always prefer it over a behavior scan. Report both options to the user with a note on why the pre-computed one is faster.

## Data Quality Checks

### Null Ratio Check

```sql
SELECT
  '{column_name}' AS column_name,
  (1.0 - CAST(COUNT({column_name}) AS DOUBLE) / COUNT(*)) AS null_ratio
FROM customers
```

For multiple columns, combine in one query:

```sql
SELECT 'col_a' AS col, (1.0 - CAST(COUNT(col_a) AS DOUBLE)/COUNT(*)) AS null_ratio FROM customers
UNION ALL
SELECT 'col_b', (1.0 - CAST(COUNT(col_b) AS DOUBLE)/COUNT(*)) FROM customers
```

### Sample Values

```sql
SELECT {column_name} FROM customers WHERE {column_name} IS NOT NULL LIMIT 5
```

### Quality Thresholds

| Null Ratio | Action |
|-----------|--------|
| >70% | Recommend AGAINST using. Suggest alternatives. |
| 50-70% | Ask user for confirmation. Explain potential impact. |
| <50% | Proceed but mention the limitation. |

## Time Column Preferences

- Prefer UTC time columns (indicated by `_utc_` or `_utc` in name).
- When looking up timestamp columns, prefer UTC unless instructed otherwise.
- Check the format of time columns BEFORE applying time functions:
  ```sql
  SELECT {time_column} FROM {table} WHERE {time_column} IS NOT NULL LIMIT 3
  ```
- For Unix timestamps: apply TD_INTERVAL directly.
- For string dates: convert first with TD_TIME_PARSE.

## Destination/Geography Inference

When inferring destinations from a user query (e.g. "flights to North America"):
- Always validate with a GROUP BY to confirm only requested countries are included.
- Example check:
  ```sql
  SELECT bkg_dest_cntry_cd, COUNT(*) AS cnt
  FROM cdp_audience_1159510.behavior_pnr_leg
  WHERE dest_traffic_area_cd = 'North America'
  GROUP BY 1 ORDER BY 2 DESC LIMIT 20
  ```
- Verify the traffic area does not include unwanted countries (e.g. "North America" also has Canada).
- Use `dest_traffic_area_cd` for regions rather than listing individual airports.

## Output Guidelines

**When data sources found, present:**
- Columns: table_name, column_name, data_type, null_ratio, quality_warning
- Segments: segment_id, segment_name, description (if relevant existing segments found)

**When clarification needed:**
- Ask one clear question about what aspect is ambiguous.

**When overview requested:**
- Provide natural language summary of available data organized by category.

## Key Table Quick Reference

| Table | Contains | Key Columns |
|-------|----------|-------------|
| customers | All attributes (flat) | cdp_customer_id, age, gender_cd, cntry_cd, member_status_cd, *_dcp_email_ind, *_nl_ind |
| behavior_pnr_leg | Flight leg detail | oper_comp_cd, mkt_comp_cd, oper_airl_cd, mkt_airl_cd, dest_cntry_cd, traffic_typ, tvl_txn_id |
| behavior_pnr | Booking header | pnr_create_dt_epoch, poo_city_cd, por_cntry_cd, bkg_dest_cntry_cd, res_status_categ_cd, corp_ind_staged |
| behavior_ancillary_data | Ancillaries | rev_type_lvl_2_code, rev_type_lvl_5_code, res_status_categ_cd, coup_status_cd, emd_coup_status_cd |
| behavior_web_all_events_stitched | Web events | event_action_lh_sn, event_category_lh_sn, event_action_lx_os, event_category_lx_os, origin_detail, destination_detail, event_timestamp_epoch, source_website, tealium_event |
| behavior_selligent_events | Email engagement | (email open/click/send events) |
