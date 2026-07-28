---
name: segment-lhg
description: LHG-specific segment creation skill. Identical YAML syntax to the generic `segment` skill but skips `tdx sg use` / `tdx sg pull` scaffolding — parent segment context is already set to `cdp_lhg_unified_first_party` by the audience-agent. Use when the audience-agent delegates segment creation for LHG audiences.
---

# LHG Segment Creation

Parent segment context is already set to `cdp_lhg_unified_first_party`. Do NOT run `tdx sg use` or `tdx sg pull` — pulling a 24.3M row parent segment is slow and unnecessary.

## Workflow

**Process one segment at a time.** For each segment:

1. **Write** the YAML file directly
2. **Validate** with `tdx sg validate <file>`
3. **Server validate** with `tdx sg push --dry-run "<file>"` — catches field/schema errors
4. **Preview** with `preview_segment` tool — get user approval before proceeding
5. **Push** with `tdx sg push -y "<file>"` — always specify the file path explicitly

Never batch multiple segments in validate or push operations.

After push succeeds, display the Console link:
```
https://console.treasuredata.com/app/audiences/<parent_id>/segments/<segment_id>
```

## Core Commands

```bash
tdx sg validate <file>               # Local YAML validation (fast)
tdx sg push --dry-run "<file>"       # Server-side validation (no pull needed)
tdx sg push -y "<file>"              # Push specific file (-y for non-interactive)
tdx sg list                          # List segments
tdx sg list -r                       # Recursive tree view
tdx sg fields                        # List available fields
```

If you need a count check, write a direct SQL query against `cdp_audience_1159510` using the same conditions from the YAML.

---

## Composite/expr — Mandatory for All Boolean Grouping

**HARD RULE:** Inside any `conditions` array (top-level `rule.conditions` or nested `Composite.conditions`), the ONLY valid condition types are:

- `Value` (attributes and behavior aggregations)
- `Composite` (boolean grouping)
- `include` / `exclude` (segment references)

**NEVER use `type: And` or `type: Or` as a condition.** The Console UI renders them as blank/empty rules. This applies at every depth. If you need boolean grouping, use `type: Composite` with an `expr` field — always.

**`filter.type: And` inside behavior conditions is SAFE.** The blank-render bug only applies to `rule.conditions` and `Composite.conditions` nesting. Behavior filter blocks use `type: Column` conditions and are rendered independently by the Console.

### expr Reference

- **Top-level** `rule.type: Composite` — reference conditions by **number** (1-indexed): `"1 and 2 and (3 or 4)"`
- **Nested** `Composite` inside another Composite — reference by **letter** (A=first, B=second…): `"(A or B or C)"`
- **Third level** — same letter convention restarts: `"(A and B)"`

### Maximum Depth: 2 Composite Levels

The Console reliably renders up to **2 levels** of Composite nesting (top-level Composite containing nested Composites). If your logic requires 3+ levels, **flatten to 2 levels using DNF** (Disjunctive Normal Form — OR of AND-groups).

**Flattening pattern:**

Before (3 levels — risky):
```
X AND (D OR E OR (F1 AND F2) OR (G1 AND G2))
```

After (2 levels — safe):
```
(X AND D) OR (X AND E) OR (X AND F1 AND F2) OR (X AND G1 AND G2)
```

Then express the flattened form as:
```yaml
rule:
  type: Composite
  expr: "(A or B or C or D)"       # 4 OR branches, each is an AND-group
  conditions:
    - type: Composite               # A
      expr: "(A and B)"             # X AND D
      conditions: [...]
    - type: Composite               # B
      expr: "(A and B)"             # X AND E
      conditions: [...]
    - type: Composite               # C
      expr: "(A and B and C)"       # X AND F1 AND F2
      conditions: [...]
    - type: Composite               # D
      expr: "(A and B and C)"       # X AND G1 AND G2
      conditions: [...]
```

### Complex Example — Italian High-Spender Flight Searchers

Request: "Italian customers (by residency OR phone number) AND top 10% spenders (2 years) AND ≥2 flight searches in 90 days (web OR app OR 1 each OR different brand websites)"

Logical structure: `(A OR B) AND C AND (D OR E OR (F1 AND F2) OR (G1 AND G2))`

This has 3 levels — flatten the inner OR group by distributing `(A OR B) AND C` into each branch:

Final structure (2 levels max):
```
( (A OR B) AND C AND D ) OR ( (A OR B) AND C AND E ) OR ( (A OR B) AND C AND F1 AND F2 ) OR ( (A OR B) AND C AND G1 AND G2 )
```

Since `(A OR B)` repeats in every branch, keep it as a nested Composite inside each:

```yaml
name: Italian High-Spender Flight Searchers
kind: batch
description: >
  Italian customers (residency or phone) who are top 10% spenders over 2 years
  and had ≥2 flight searches in last 90 days across web/app/multi-brand.

rule:
  type: Composite
  expr: "(A or B or C or D)"
  conditions:
    # Branch A: Italian + Top spender + ≥2 web searches
    - type: Composite
      expr: "(A and B and C)"
      conditions:
        - type: Composite                    # A — Italian identification
          expr: "(A or B)"
          conditions:
            - type: Value
              attribute: cntry_cd
              operator:
                type: Equal
                value: "it"
            - type: Value
              attribute: phone_cntry_cd
              operator:
                type: Equal
                value: "+39"
        - type: Value                        # B — Top 10% spenders
          attribute: ""
          source: behavior_pnr_data
          aggregation:
            type: Sum
            column: revenue_amount
          operator:
            type: GreaterEqual
            value: 5000                      # Proxy for top 10% threshold
          timeWindow:
            duration: 730
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: res_status_categ_cd
                operator:
                  type: Equal
                  value: "K"
        - type: Value                        # C — ≥2 web searches (LH/SN bucket)
          attribute: ""
          source: behavior_web_all_events_stitched
          aggregation:
            type: Count
          operator:
            type: GreaterEqual
            value: 2
          timeWindow:
            duration: 90
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: event_action_lh_sn
                operator:
                  type: Equal
                  value: "submit"
              - type: Column
                column: event_category_lh_sn
                operator:
                  type: In
                  value: ["flma", "flightmanager"]

    # Branch B: Italian + Top spender + ≥2 app searches
    - type: Composite
      expr: "(A and B and C)"
      conditions:
        - type: Composite                    # A — Italian (same as above)
          expr: "(A or B)"
          conditions:
            - type: Value
              attribute: cntry_cd
              operator:
                type: Equal
                value: "it"
            - type: Value
              attribute: phone_cntry_cd
              operator:
                type: Equal
                value: "+39"
        - type: Value                        # B — Top 10% spenders (same)
          attribute: ""
          source: behavior_pnr_data
          aggregation:
            type: Sum
            column: revenue_amount
          operator:
            type: GreaterEqual
            value: 5000
          timeWindow:
            duration: 730
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: res_status_categ_cd
                operator:
                  type: Equal
                  value: "K"
        - type: Value                        # C — ≥2 app searches
          attribute: ""
          source: behavior_web_all_events_stitched
          aggregation:
            type: Count
          operator:
            type: GreaterEqual
            value: 2
          timeWindow:
            duration: 90
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: tealium_event
                operator:
                  type: Equal
                  value: "click booking search button"

    # Branch C: Italian + Top spender + 1 web + 1 app
    - type: Composite
      expr: "(A and B and C and D)"
      conditions:
        - type: Composite                    # A — Italian
          expr: "(A or B)"
          conditions:
            - type: Value
              attribute: cntry_cd
              operator:
                type: Equal
                value: "it"
            - type: Value
              attribute: phone_cntry_cd
              operator:
                type: Equal
                value: "+39"
        - type: Value                        # B — Top 10% spenders
          attribute: ""
          source: behavior_pnr_data
          aggregation:
            type: Sum
            column: revenue_amount
          operator:
            type: GreaterEqual
            value: 5000
          timeWindow:
            duration: 730
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: res_status_categ_cd
                operator:
                  type: Equal
                  value: "K"
        - type: Value                        # C — ≥1 web search
          attribute: ""
          source: behavior_web_all_events_stitched
          aggregation:
            type: Count
          operator:
            type: GreaterEqual
            value: 1
          timeWindow:
            duration: 90
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: event_action_lh_sn
                operator:
                  type: Equal
                  value: "submit"
              - type: Column
                column: event_category_lh_sn
                operator:
                  type: In
                  value: ["flma", "flightmanager"]
        - type: Value                        # D — ≥1 app search
          attribute: ""
          source: behavior_web_all_events_stitched
          aggregation:
            type: Count
          operator:
            type: GreaterEqual
            value: 1
          timeWindow:
            duration: 90
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: tealium_event
                operator:
                  type: Equal
                  value: "click booking search button"

    # Branch D: Italian + Top spender + searches on different LH brand websites
    - type: Composite
      expr: "(A and B and C and D)"
      conditions:
        - type: Composite                    # A — Italian
          expr: "(A or B)"
          conditions:
            - type: Value
              attribute: cntry_cd
              operator:
                type: Equal
                value: "it"
            - type: Value
              attribute: phone_cntry_cd
              operator:
                type: Equal
                value: "+39"
        - type: Value                        # B — Top 10% spenders
          attribute: ""
          source: behavior_pnr_data
          aggregation:
            type: Sum
            column: revenue_amount
          operator:
            type: GreaterEqual
            value: 5000
          timeWindow:
            duration: 730
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: res_status_categ_cd
                operator:
                  type: Equal
                  value: "K"
        - type: Value                        # C — ≥1 search on LH/SN
          attribute: ""
          source: behavior_web_all_events_stitched
          aggregation:
            type: Count
          operator:
            type: GreaterEqual
            value: 1
          timeWindow:
            duration: 90
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: event_action_lh_sn
                operator:
                  type: Equal
                  value: "submit"
              - type: Column
                column: event_category_lh_sn
                operator:
                  type: In
                  value: ["flma", "flightmanager"]
        - type: Value                        # D — ≥1 search on LX/OS
          attribute: ""
          source: behavior_web_all_events_stitched
          aggregation:
            type: Count
          operator:
            type: GreaterEqual
            value: 1
          timeWindow:
            duration: 90
            unit: day
          filter:
            type: And
            conditions:
              - type: Column
                column: event_action_lx_os
                operator:
                  type: Equal
                  value: "submit"
              - type: Column
                column: event_category_lx_os
                operator:
                  type: In
                  value: ["flma", "flightmanager"]
```

**Key points in this example:**
- Top level: `Composite` with OR of 4 branches
- Each branch: `Composite` with AND of conditions
- Italian identification inside each branch: `Composite` with OR of 2 values
- Behavior `filter.type: And` — safe, not affected by render bug
- Maximum Composite depth: exactly 2 (top OR → branch AND → Italian OR)
- Conditions that repeat (Italian, Top spender) are duplicated in each branch — this is required by DNF flattening

---

For all YAML syntax, operators, Composite/expr patterns, behavior conditions, and error codes — see the **segment** and **validate-segment** skills.
