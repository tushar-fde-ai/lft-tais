---
name: lhg-ancillary
description: LHG ancillary product reference — product codes, revenue hierarchy, upgrade types, lounge tiers, sports equipment, pet products, and permission architecture. Load when the user request mentions seats, bags, upgrades, lounges, pets, carbon offset, or any ancillary product.
---

# LHG Ancillary Products Reference

Source table: `behavior_pnr_leg_anc` (current DB: `cdp_audience_1159510`)

> **Note:** Some KB documentation references `behavior_ancillary_data` — this is the old name. The correct table in `cdp_audience_1159510` is `behavior_pnr_leg_anc`.

---

## Triple Confirmed Filter — ALWAYS Required

Every ancillary query must apply all three filters:

```sql
WHERE res_status_categ_cd = 'K'
  AND coup_status_cd IN ('F', 'I', 'A')
  AND emd_coup_status_cd IN ('F', 'I', 'A')
```

This yields ~14.7M clean rows from 16.7M total. Never query ancillaries without all three.

---

## Revenue Type Hierarchy (5 levels)

### Level 1
| Code | Description | Share |
|------|-------------|-------|
| FLT_ANC | Flight-related ancillaries | 99.96% |
| NONFLT_ANC | Non-flight-related | 0.04% |

### Level 2 — Product Categories
| Code | Description | Share | Revenue |
|------|-------------|-------|---------|
| ASR | Seat Reservation | 55% | €346M, avg €37 |
| BAG | Baggage | 30% | €217M, avg €43 |
| UPG | Upgrade | 10% | €579M, avg €351 — 50% of all ancillary revenue |
| OGS | Ground/Airport Services | 1.4% | €8.5M |
| NULL | Carbon Offset (0EO) + Upgrade with Miles (0NI) only | 3.2% | — |

### Level 3 — Key Subcategories
- **ASR**: ASR (standard seat), SPSE (extra legroom)
- **BAG**: FBAG (first bag), ABAG (additional), XBAG (excess), SBAG (second), PETC (pet cabin), SPEQ (sports equipment), AVIH (pet cargo), CBAG (trader SN)
- **UPG**: FUPG (fixed-price, €420M), BUPG (bid, €117M)
- **OGS**: LOUN (lounge, €2.2M)

### Level 5 — Top Products
| Code | Description | Volume | Revenue |
|------|-------------|--------|---------|
| 0B5 | Seat reservation | 9.15M | €342M |
| 0CC | First bag | 3.12M | €92M |
| 0BJ | Fixed-price upgrade | 867K | €335M avg €387 |
| 0GO | Additional bag | 521K | €29M |
| 0EO | Carbon offset | 433K | €5.3M — growing 8x since 2024 |
| UPP | Bid upgrade | 382K | €119M avg €313 |
| 0CD | Second bag | 295K | €14M |
| 0BT | Pet in cabin | 221K | €14M |
| 060 | Airport upgrade M→C | 215K | €55M |
| 0NI | Upgrade with miles | 106K | €0 (loyalty redemption) |

---

## Upgrade Types

| Type | Codes | Volume | Revenue | Description |
|------|-------|--------|---------|-------------|
| Fixed-price | 0BJ | 774K | €299M | Guaranteed at booking |
| Bid | UPP | 371K | €116M | Customer bids, accepted if threshold met |
| Airport | 060, 061, 062, 06Z | 238K | €77M | Day-of-departure at gate |
| Special | 04E, 042, 04D, 07I | 109K | €27M | Promotional offers |
| Miles | 0NI | 106K | €0 | Loyalty points redemption |
| PEP | PEC, PMC, PME | ~93 | €4.5K | New low-cost product (2026) |

Airport upgrade codes: 060 = M→C, 061 = E→C, 062 = M→F, 06Z = C→F

**Always use `voluntry_upg_ind = 'Y'`** to filter for intentional upgrades only — involuntary upgrades (operational necessity, not paid by passenger) will pollute upgrade buyer audiences otherwise.

---

## Sports Equipment (under SPEQ)

| Sport | Codes | Volume |
|-------|-------|--------|
| Golf | 0HR, 0HQ | 21.5K legs |
| Bike | 0FQ, 0EC, 0NZ | 19.7K |
| Winter sports | 07T | 7.9K |
| Weapons | — | 4.3K |
| Shortboards (surf/wake) | — | 4.0K |

---

## Lounge Access (under LOUN)

| Tier | Codes | Volume | Notes |
|------|-------|--------|-------|
| Business | CBL, OBW, PBL | ~53K | OBW = walk-in/paid ("lounge payers") |
| Senator | OCS, CSL | ~16K | Complimentary for FTL/SEN |
| First | HFT, CFT | ~800 | HON terminal access |

"Lounge payers" (revenue signal): `rev_type_lvl_5_code = 'OBW'`
"Premium lounge": `rev_type_lvl_4_code IN ('SLOU', 'FTER')`

---

## Pet Products

| Code | Description | Volume | Revenue |
|------|-------------|--------|---------|
| 0BT | Pet in cabin | 221K legs | €14M |
| 0A0 | Pet in hold — large | 13K | €3.6M |
| 0AZ | Pet in hold — medium | 7.4K | €1.1M |

---

## Carbon Offset (0EO)

- Level 5 code: `0EO`. NULL at Level 2–4 (added after hierarchy defined).
- Volume: ~389K legs, €5.3M, avg €10
- Growth: 3.5K/month (2024) → 22–32K/month (2026) — roughly 8x
- "Sustainability-conscious travelers" = customers who bought 0EO

---

## Ancillary Attach Rates by Cabin

| Cabin | Attach Rate | Top Product |
|-------|-------------|-------------|
| Premium Economy | 33.3% — highest | 22.3% seat, 8.6% upgrade |
| Economy | 15% | 8.9% seat, 5.2% bag |
| First | 9.1% | 7.9% upgrade (seat/bag mostly included) |
| Business | 8% | 5.6% upgrade |

Premium Economy is the best target for ancillary upsell campaigns.

---

## Discontinued Products — DO NOT TARGET

These products have zero volume — using them returns empty audiences:

| Code | Product | Dead Since |
|------|---------|-----------|
| OIS / CATE | À la carte inflight dining | Jan 2023 |
| UPC / UPF | Call center upgrades | Dec 2022 |
| A20 | Old bid upgrade (replaced by UPP) | Feb 2024 |
| OCC / OCL / OCP | Reseat products | Nov 2024 |
| SAX | Seat change | Jul 2025 |
| INS | Travel insurance | Apr 2022 |

---

## Approximate Audience Sizes (for expectation-setting)

| Audience | Size |
|----------|------|
| Upgrade buyers (2Y) | ~600K |
| Pet travelers (ever) | ~200K |
| Carbon offset buyers | ~350K |
| Business class (2Y) | ~2–3M |
| First class (2Y) | ~200–400K |
| SEN + HON members | ~180K |
