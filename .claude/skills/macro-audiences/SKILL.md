---
name: macro-audiences
description: Pre-built macro audience definitions for Lufthansa Group — SQL templates and segment logic for common audience patterns. Use when asked to create Gen Z, Luxury, Corporate, Churning, Flight Searcher, Destination Affinity, Not Yet Departed, Open Booking, Suppy, or Henry audiences. Triggers on: "Gen Z audience", "luxury travelers", "corporate travelers", "lapsed flyers", "flight searchers", "destination affinity", "not yet departed", "open booking", "suppy", "henry", "churning customers", "macro audience", "pre-built audience".
---

# Macro Audiences — Lufthansa Group

## 1. Overview

12 pre-built macro audience definitions for LHG. These are TEMPLATES — adapt the origin,
destination, timeframe, and other parameters to the specific user request.

**Important Notes:**
- SQL references `cdp_audience_841858` (older audience DB). When using, adapt to current output database `cdp_audience_1159510`.
- Always validate with `segment-review` before creating.
- These can be used as starting points; combine or modify conditions as needed.
- All epoch comparisons use UTC day boundaries via `date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))`.
- Web event tables cover both LH/SN (`_lh_sn`) and LX/OS (`_lx_os`) brand variants.

---

## 2. Audience Index

Quick reference — match user requests to the right template:

| # | Audience | One-Line Description |
|---|----------|---------------------|
| 1 | Destination Affinity Broad | Customers showing interest in a destination region via bookings + web searches |
| 2 | Destination Affinity Specific | Customers with focused interest in a single route (e.g. FRA->NYC) |
| 3 | Not Yet Departed | Customers with upcoming flights who haven't departed yet |
| 4 | Open Booking | All customers with at least one active/upcoming booking |
| 5 | Gen Z Travelers | Customers aged 13-28 (CRM + web data sources) |
| 6 | Premium Leisure Flyers | Luxury & upgrade affinity — Business/First class, premium ancillaries |
| 7 | Flight Searcher | Searched for a flight on any LHG website within past 30 days |
| 8 | Corporate-Only Traveler | Exclusively corporate rate — no leisure trips |
| 9 | Corporate Traveler | At least one flight on corporate rate (may include leisure) |
| 10 | Churning Customers | Purchased 2-3 years ago, no purchases in last 12 months |
| 11 | Suppy | Senior Urban Professionals — established career, cultural curiosity, premium |
| 12 | Henry | High Earners Not Rich Yet — young ambitious professionals, eco class |

---

## 3. Audience Definitions

---

### 3.1 Destination Affinity Broad

**Summary:** Customers who have shown affinity for a specific destination AND similar destinations within the same country/region (e.g., FRA to any U.S. destination).

**Template Parameters:** Origin city, destination country, booking lookback (days), search lookback (days), residence country.

**Key Attributes:**
- `cntry_cd` = 'de' (residence)
- `pnr_create_dt_epoch` within past 1095 days
- `poo_city_cd` = 'FRA' (point of origin)
- `por_cntry_cd` = 'US' (destination country)
- Web: `event_category_lh_sn`/`event_category_lx_os` = 'flightmanager', `event_action` = 'submit'
- `origin_detail` = 'fra', `destination_detail` IN (list of US airports)
- `event_timestamp_epoch` within past 30 days
- `user_country_address` = 'de'

**Source Tables:** customers, behavior_pnr_data, behavior_web_all_events_stitched

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where (a."cntry_cd" = 'de'
  and a."cdp_customer_id" in (
    select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
    where _r1."pnr_create_dt_epoch" between to_unixtime(date_add('day', -1095, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
      and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1)
      and _r1."poo_city_cd" = 'FRA'
      and _r1."por_cntry_cd" = 'US'))
or (a."cdp_customer_id" in (
    select _r2."cdp_customer_id" from "cdp_audience_841858"."behavior_web_all_events_stitched" _r2
    where _r2."event_category_lh_sn" = 'flightmanager'
      and _r2."event_action_lh_sn" = 'submit'
      and _r2."origin_detail" = 'fra'
      and _r2."destination_detail" in ('ATL','LAX','ORD','DFW','DEN','JFK','SFO','SEA','LAS','MCO','CLT','PHX','MIA','IAH','BOS','MSP','EWR','DTW','PHL','LGA','BWI','SLC','SAN','DCA','TPA','PDX','HNL','STL','AUS','BNA')
      and _r2."event_timestamp_epoch" between to_unixtime(date_add('day', -30, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
        and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1)
      and _r2."user_country_address" = 'de')
  or a."cdp_customer_id" in (
    select _r3."cdp_customer_id" from "cdp_audience_841858"."behavior_web_all_events_stitched" _r3
    where _r3."event_category_lx_os" = 'flightmanager'
      and _r3."event_action_lx_os" = 'submit'
      and _r3."origin_detail" = 'fra'
      and _r3."destination_detail" in ('ATL','LAX','ORD','DFW','DEN','JFK','SFO','SEA','LAS','MCO','CLT','PHX','MIA','IAH','BOS','MSP','EWR','DTW','PHL','LGA','BWI','SLC','SAN','DCA','TPA','PDX','HNL','STL','AUS','BNA')
      and _r3."event_timestamp_epoch" between to_unixtime(date_add('day', -30, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
        and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1)
      and _r3."user_country_address" = 'de'))
```

**Purpose:** Expand destination-based targeting to include travelers showing interest in similar destinations.
**Related Campaigns:** Destination Expansion, Regional Interest Targeting, Cross-Destination Promotions.

---

### 3.2 Destination Affinity Specific

**Summary:** Customers with focused interest in a single route (e.g., FRA to NYC/JFK).

**Key Attributes:** Same as Broad but with single destination (`por_city_cd` = 'NYC', `destination_detail` = 'jfk').

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where (a."cntry_cd" = 'de'
  and a."cdp_customer_id" in (
    select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
    where _r1."pnr_create_dt_epoch" between to_unixtime(date_add('day', -1095, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
      and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1)
      and _r1."poo_city_cd" = 'FRA' and _r1."por_city_cd" = 'NYC'))
or (a."cdp_customer_id" in (
    select _r2."cdp_customer_id" from "cdp_audience_841858"."behavior_web_all_events_stitched" _r2
    where _r2."event_category_lh_sn" = 'flightmanager' and _r2."event_action_lh_sn" = 'submit'
      and _r2."origin_detail" = 'fra' and _r2."destination_detail" = 'jfk'
      and _r2."event_timestamp_epoch" between to_unixtime(date_add('day', -30, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
        and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1)
      and _r2."user_country_address" = 'de')
  or a."cdp_customer_id" in (
    select _r3."cdp_customer_id" from "cdp_audience_841858"."behavior_web_all_events_stitched" _r3
    where _r3."event_category_lx_os" = 'flightmanager' and _r3."event_action_lx_os" = 'submit'
      and _r3."origin_detail" = 'fra' and _r3."destination_detail" = 'jfk'
      and _r3."event_timestamp_epoch" between to_unixtime(date_add('day', -30, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
        and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1)
      and _r3."user_country_address" = 'de'))
```

**Purpose:** Destination-specific marketing and route-focused campaigns.
**Related Campaigns:** Route Launch, City-Pair Promotions, Seasonal Route Marketing.

---

### 3.3 Not Yet Departed

**Summary:** Customers with upcoming flight scheduled today or future, not yet taken first flight leg.

**Key Attributes:**
- `pnr_sched_dep_utc_date` >= today (epoch comparison)

**Source Tables:** customers, behavior_pnr_data

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where a."cdp_customer_id" in (
  select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
  where to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) <= _r1."pnr_sched_dep_utc_date")
```

**Purpose:** Pre-departure ancillary upsell (baggage, seat upgrades, lounge access).
**Related Campaigns:** Pre-Trip Communication, Ancillary Upsell, Check-In Reminders.

---

### 3.4 Open Booking

**Summary:** Customers with at least one active booking (departing or returning today or future).

**Key Attributes:**
- `pnr_sched_dep_utc_date` >= today OR `pnr_sched_arr_utc_date` >= today

**Source Tables:** customers, behavior_pnr_data

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where a."cdp_customer_id" in (
  select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
  where to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) <= _r1."pnr_sched_dep_utc_date"
    or to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) <= _r1."pnr_sched_arr_utc_date")
```

**Purpose:** Pre-trip communication, ancillary upsell, check-in reminders.
**Related Campaigns:** Booking Confirmation Flow, Trip Preparation, In-Trip Services.

---

### 3.5 Gen Z Travelers

**Summary:** Customers aged 13-28 from CRM and web data.

**Key Attributes:**
- `age` BETWEEN 13 AND 28 (customers table)
- `user_age` BETWEEN 13 AND 28 (web events table)

**Source Tables:** customers, behavior_web_all_events_stitched

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where (a."age" between 13 and 28)
  or a."cdp_customer_id" in (
    select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_web_all_events_stitched" _r1
    where _r1."user_age" between 13 and 28)
```

**Purpose:** Youth travel discounts, student offers, social media campaigns.
**Related Campaigns:** Student Fares, Youth Discovery, Social-First Campaigns.

---

### 3.6 Premium Leisure Flyers (Luxury)

**Summary:** Premium/luxury travel affinity — Business/First class, premium ancillaries (extra legroom, upgrades, lounges), excludes corporate-only travelers.

**Key Attributes:**
- `curr_mbr_status_cd` IN ('hon','sen') — Miles & More elite status
- `oper_comp_cd_string` contains 'C' (Business) or 'F' (First)
- Ancillary codes in behavior_pnr_data:
  - ASR types: 'OCP', 'SAE', 'SAL', 'SAS' (premium seating)
  - 'LOUN' (lounge access)
  - Upgrade codes: '042', '04D', '04E', '060', '061', '062', '06Z', '07I'
- Web page_view on pages containing 'first', 'business', 'upgrade', 'luxury', 'premium'
- `segment_1y` BETWEEN 3 AND 6 (high CLV)
- Active booking in past 1095 days
- EXCLUDE: corporate-only travelers (`corp_ind_staged` = 'N' count = 0)

**Source Tables:** customers, behavior_pnr_data, behavior_web_all_events_stitched

**SQL Pattern:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where (
  -- Condition 1: Elite M&M status
  a."curr_mbr_status_cd" in ('hon','sen')
  -- OR Condition 2: Business/First cabin history in last 3 years
  or a."cdp_customer_id" in (
    select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
    where (_r1."oper_comp_cd_string" like '%C%' or _r1."oper_comp_cd_string" like '%F%')
      and _r1."pnr_create_dt_epoch" between to_unixtime(date_add('day', -1095, ...)) and ...)
  -- OR Condition 3: Premium ancillary purchases
  or a."cdp_customer_id" in (
    select _r2."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r2
    where _r2."asr_type" in ('OCP','SAE','SAL','SAS','LOUN')
      or _r2."upgrade_code" in ('042','04D','04E','060','061','062','06Z','07I'))
  -- OR Condition 4: Web browsing premium content
  or a."cdp_customer_id" in (
    select _r3."cdp_customer_id" from "cdp_audience_841858"."behavior_web_all_events_stitched" _r3
    where (_r3."page_path" like '%first%' or _r3."page_path" like '%business%'
      or _r3."page_path" like '%upgrade%' or _r3."page_path" like '%luxury%'
      or _r3."page_path" like '%premium%'))
  -- OR Condition 5: High CLV segment
  or a."segment_1y" between 3 and 6
)
-- EXCLUDE corporate-only travelers
and (select count(*) from "cdp_audience_841858"."behavior_pnr_data" _rx
  where _rx."corp_ind_staged" = 'N' and _rx."cdp_customer_id" = a."cdp_customer_id") > 0
```

**Purpose:** Premium cabin offers, lounge access, upgrade campaigns, high-value loyalty.
**Related Campaigns:** First Class Launch, Premium Ancillary Upsell, Loyalty Tier Retention.

---

### 3.7 Flight Searcher

**Summary:** Searched for a flight on any LHG website within past 30 days.

**Key Attributes:**
- `event_action_lh_sn`/`event_action_lx_os` = 'submit'
- `event_category_lh_sn`/`event_category_lx_os` = 'flightmanager'
- `origin_detail` IS NOT NULL, `destination_detail` IS NOT NULL
- `event_timestamp_epoch` within 30 days

**Source Tables:** customers, behavior_web_all_events_stitched

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where a."cdp_customer_id" in (
    select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_web_all_events_stitched" _r1
    where _r1."event_action_lh_sn" = 'submit' and _r1."event_category_lh_sn" = 'flightmanager'
      and _r1."origin_detail" is not null and _r1."destination_detail" is not null
      and _r1."event_timestamp_epoch" between to_unixtime(date_add('day', -30, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
        and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1))
  or a."cdp_customer_id" in (
    select _r2."cdp_customer_id" from "cdp_audience_841858"."behavior_web_all_events_stitched" _r2
    where _r2."event_action_lx_os" = 'submit' and _r2."event_category_lx_os" = 'flightmanager'
      and _r2."origin_detail" is not null and _r2."destination_detail" is not null
      and _r2."event_timestamp_epoch" between to_unixtime(date_add('day', -30, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
        and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1))
```

**Purpose:** Search retargeting, flight offers, booking conversion.
**Related Campaigns:** Abandoned Search, Price Drop Alerts, Route Retargeting.

---

### 3.8 Corporate-Only Traveler

**Summary:** ONLY corporate rate flights — no leisure trips whatsoever.

**Key Attributes:**
- `corp_ind_staged` = 'Y' (at least 1 corporate booking)
- `corp_ind_staged` = 'N' (count = 0 — zero non-corporate bookings)

**Source Tables:** customers, behavior_pnr_data

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where (select count(*) from "cdp_audience_841858"."behavior_pnr_data" _r1
    where _r1."corp_ind_staged" = 'N' and _r1."cdp_customer_id" = a."cdp_customer_id") = 0
  and a."cdp_customer_id" in (
    select _r2."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r2
    where _r2."corp_ind_staged" = 'Y')
```

**Purpose:** B2B engagement, corporate loyalty, business travel retention.
**Related Campaigns:** Corporate Program, TMC Partnerships, Business Travel Incentives.

---

### 3.9 Corporate Traveler (Mixed)

**Summary:** At least one flight on corporate rate (can include leisure trips too).

**Key Attributes:**
- `corp_ind_staged` = 'Y' (at least 1)

**Source Tables:** customers, behavior_pnr_data

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where a."cdp_customer_id" in (
  select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
  where _r1."corp_ind_staged" = 'Y')
```

**Purpose:** Corporate engagement, B2B retargeting, business travel incentives.
**Related Campaigns:** Bleisure Offers, Corporate Leisure Cross-Sell, Mixed Travel Loyalty.

---

### 3.10 Churning Customers

**Summary:** Purchased flights 2-3 years ago but NOT in the last 12 months.

**Key Attributes:**
- `pnr_create_dt_epoch` between 730-1095 days ago (had activity 2-3 years back)
- Count of bookings in last 365 days = 0

**Source Tables:** customers, behavior_pnr_data

**SQL:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where (a."cdp_customer_id" in (
    select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
    where _r1."pnr_create_dt_epoch" between to_unixtime(date_add('day', -1095, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
      and to_unixtime(date_add('day', -730, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC')))))
  or a."cdp_customer_id" in (
    select _r2."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r2
    where _r2."pnr_create_dt_epoch" between to_unixtime(date_add('day', -730, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
      and to_unixtime(date_add('day', -365, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))))
and (select count(*) from "cdp_audience_841858"."behavior_pnr_data" _r3
    where _r3."pnr_create_dt_epoch" between to_unixtime(date_add('day', -365, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
      and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1)
    and _r3."cdp_customer_id" = a."cdp_customer_id") = 0
```

**Purpose:** Reactivation campaigns, customer win-back, retention optimization.
**Related Campaigns:** Win-Back Offers, Lapsed Flyer Reactivation, Miles Expiry Reminders.

---

### 3.11 Suppy (Senior Urban Professionals)

**Summary:** Age 30-59, Switzerland, DE/FR/EN languages, FTL/SEN/HON or CLV 4-5-6, intercontinental history, family travelers.

**Key Attributes:**
- `age` >= 30 (customers table)
- `cntry_cd` = 'ch' (Switzerland)
- `lang_cd` IN ('de','fr','en')
- `curr_mbr_status_cd` IN ('ftl','sen','hon') OR `segment_5y` IN ('4','5','6')
- Booked in last 1095 days (`pnr_create_dt_epoch`)
- Has infant/child in PNR (`infant_in_pnr_cnt` > 0 OR `child_in_pnr_cnt` > 0) OR `adult_in_pnr_cnt` = 2
- Has intercontinental flights (`traffic_typ_string` contains 'I')

**Source Tables:** customers, behavior_pnr_data

**SQL Pattern:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where a."age" >= 30
  and a."cntry_cd" = 'ch'
  and a."lang_cd" in ('de','fr','en')
  -- Membership/CLV gate
  and (a."curr_mbr_status_cd" in ('ftl','sen','hon') or a."segment_5y" in ('4','5','6'))
  -- Recency: booked in last 3 years
  and a."cdp_customer_id" in (
    select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
    where _r1."pnr_create_dt_epoch" between to_unixtime(date_add('day', -1095, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
      and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1))
  -- Family indicator
  and a."cdp_customer_id" in (
    select _r2."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r2
    where _r2."infant_in_pnr_cnt" > 0 or _r2."child_in_pnr_cnt" > 0 or _r2."adult_in_pnr_cnt" = 2)
  -- Intercontinental travel
  and a."cdp_customer_id" in (
    select _r3."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r3
    where _r3."traffic_typ_string" like '%I%')
```

**Purpose:** Cultural discovery, premium lifestyle, luxury leisure campaigns.
**Related Campaigns:** Family Premium Packages, Cultural Destinations, Long-Haul Leisure Offers.

---

### 3.12 Henry (High Earners Not Rich Yet)

**Summary:** Age 18-29, Switzerland, DE/FR languages, BASE membership or CLV 1-2-3, economy class only, no family, active in last 3 years.

**Key Attributes:**
- `age` BETWEEN 18 AND 29
- `cntry_cd` = 'ch'
- `lang_cd` IN ('de','fr','en')
- `curr_mbr_status_cd` = 'base' OR `segment_5y` IN ('1','2','3')
- No Business/First/Premium Economy in last 3 years (`oper_comp_cd_string` must NOT contain 'E', 'C', or 'F')
- Economy bookings present (`oper_comp_cd_string` contains 'M')
- No infants or children in PNR (`infant_in_pnr_cnt` = 0 AND `child_in_pnr_cnt` = 0)
- Active in past 1095 days

**Source Tables:** customers, behavior_pnr_data

**SQL Pattern:**
```sql
select a.* from "cdp_audience_841858"."customers" a
where a."age" between 18 and 29
  and a."cntry_cd" = 'ch'
  and a."lang_cd" in ('de','fr','en')
  -- Base membership or low CLV
  and (a."curr_mbr_status_cd" = 'base' or a."segment_5y" in ('1','2','3'))
  -- Has economy bookings in last 3 years
  and a."cdp_customer_id" in (
    select _r1."cdp_customer_id" from "cdp_audience_841858"."behavior_pnr_data" _r1
    where _r1."oper_comp_cd_string" like '%M%'
      and _r1."pnr_create_dt_epoch" between to_unixtime(date_add('day', -1095, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
        and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1))
  -- EXCLUDE premium cabin history
  and (select count(*) from "cdp_audience_841858"."behavior_pnr_data" _r2
    where (_r2."oper_comp_cd_string" like '%C%' or _r2."oper_comp_cd_string" like '%F%' or _r2."oper_comp_cd_string" like '%E%')
      and _r2."pnr_create_dt_epoch" between to_unixtime(date_add('day', -1095, date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))))
        and (to_unixtime(date_trunc('day', from_unixtime(to_unixtime(current_timestamp), 'UTC'))) - 1)
      and _r2."cdp_customer_id" = a."cdp_customer_id") = 0
  -- EXCLUDE family travelers
  and (select count(*) from "cdp_audience_841858"."behavior_pnr_data" _r3
    where (_r3."infant_in_pnr_cnt" > 0 or _r3."child_in_pnr_cnt" > 0)
      and _r3."cdp_customer_id" = a."cdp_customer_id") = 0
```

**Purpose:** Lifestyle aspiration, young professionals, brand consideration campaigns.
**Related Campaigns:** Young Professional Offers, Eco-Friendly Travel, Aspiration Messaging.

---

## 4. Usage Notes

**Adapting Templates:**
- Replace origin/destination with user's requested cities/countries
- Adjust lookback windows (days) per user request (default: 1095 for bookings, 30 for web)
- Change residence country filter (`cntry_cd`, `user_country_address`) as needed
- For post-May 2026 web events, use `'flma'` instead of `'flightmanager'` for `event_category`
- Airport code lists for "Broad" audiences should be adapted per destination country

**Database Adaptation:**
- All SQL above references `cdp_audience_841858` (older output DB) where `behavior_pnr_data` is valid
- When executing against `cdp_audience_1159510`, table names differ:
  - `behavior_pnr_data` → `behavior_pnr_leg` (for flight leg detail) or `behavior_pnr` (for booking header)
  - `behavior_flight_leg` → `behavior_pnr_leg`
  - `behavior_ancillary_data` → `behavior_pnr_leg_anc`
- Column names are largely the same across both databases

**Validation:**
- Always run through `segment-review` before creating
- Check segment size with a COUNT(*) query before pushing to production
- Confirm with user if adapting parameters significantly from the template
- For complex audiences (6, 11, 12), validate each condition independently first

**Combining Audiences:**
- Use INTERSECT for "must match both" (e.g., Gen Z AND Flight Searcher)
- Use UNION for "match either" (e.g., Corporate-Only OR Corporate Mixed)
- Use EXCEPT for exclusions (e.g., Flight Searcher EXCEPT Open Booking)
- Prefer subquery-based combination within a single SELECT for TreasureData compatibility
