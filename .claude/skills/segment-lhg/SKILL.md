---
name: segment-lhg
description: LHG segment creation — full YAML syntax, operators, Composite/expr patterns, behavior conditions, validation, and push workflow. All segment construction knowledge in one place; no external skill references needed.
---

# LHG Segment Creation

Parent segment context is already set to `cdp_lhg_unified_first_party`. Do NOT run `tdx sg use` or `tdx sg pull` — pulling a 24.3M row parent segment is slow and unnecessary.

---

## Section 1: File Location

Before writing any YAML, determine the output folder:

1. If a folder has already been established in this session, use it
2. Otherwise, ask the user: "Would you like me to create a new folder or use an existing one?"
3. Default folder name (if creating new): `Treasure AI Studio Segments`

All segment YAML files go into this folder as `<segment-name-kebab-case>.yml`.

---

## Section 2: Workflow

**Process one segment at a time.** For each segment:

1. **Write** the YAML file to the established folder
2. **Pre-push nesting check** — count Composite levels in the YAML before proceeding:
   - If any Composite contains another Composite that itself contains another Composite → 3 levels → **stop, flatten to DNF first** (see Section 11)
   - Maximum 2 levels. Pushing a 3-level structure will succeed but render blank in Console UI
3. **Validate** with `tdx sg validate <file>`
   - Run from any directory — pass the full or relative path. No `cd`, no `tdx.json` needed
4. **Server validate** with `tdx sg push --dry-run "<file>"` — catches field/schema errors
5. **Preview** with `preview_segment` tool — get user approval before proceeding
6. **Push** with `tdx sg push -y "<file>"` — always specify the file path explicitly

Never batch multiple segments in validate or push operations.

After push succeeds, display the Console link:
```
https://console.treasuredata.com/app/audiences/<parent_id>/segments/<segment_id>
```

---

## Section 3: Core Commands

```bash
tdx sg validate <file>               # Local YAML validation — works on any path, no tdx.json needed
tdx sg push --dry-run "<file>"       # Server-side validation (no pull needed)
tdx sg push -y "<file>"              # Push specific file (-y for non-interactive)
tdx sg list                          # List segments
tdx sg list -r                       # Recursive tree view
tdx sg fields                        # List available fields
```

If you need a count check, write a direct SQL query against `cdp_audience_1159510` using the same conditions from the YAML.

---

## Section 4: Performance — Minimize Push Time

Push time is proportional to two factors:

1. **Branch count** — more Composite OR branches = larger compilation payload
2. **Behavior condition count** — each behavior scan (especially 730+ day windows) adds server evaluation time

To reduce push time:

- **Sub-segment reuse via `include`**: if multiple segments share a heavy common condition (e.g. "Italian top spenders"), push that as a base segment first, then reference it via `include` in subsequent segments. This collapses repeated branches.
  ```yaml
  # Base segment already pushed: "Italian Top Spenders Base"
  rule:
    type: And
    conditions:
      - type: include
        segment: "Italian Top Spenders Base"   # references existing server segment
      - type: Value
        attribute: ...                          # additional condition
  ```
- **Pre-computed attributes over behavior scans**: prefer `decile_monetary_24m = 10` over a 730-day revenue Sum scan. See `lhg-policy` section 2.5 for the full lookup table.
- **Minimize DNF branch count**: if DNF produces 8+ branches, check if an `include` reference to a base segment can eliminate the repeated shared conditions from each branch.

---

## Section 5: YAML Structure

```yaml
name: High Value Customers          # Required
kind: batch                         # batch | realtime | funnel_stage

rule:
  type: Composite                   # Use Composite for all boolean grouping (see Section 11)
  expr: "1 and (2 or 3)"
  conditions:
    - type: Value
      attribute: country
      operator:
        type: In
        value: ["DE", "AT", "CH"]
    - type: Value
      attribute: ltv
      operator:
        type: Greater
        value: 1000
    - type: Value
      attribute: last_purchase_date
      operator:
        type: TimeWithinPast
        value: 30
        unit: day
```

**`kind` values:**
- `batch` — standard scheduled evaluation
- `realtime` — real-time profile evaluation
- `funnel_stage` — used inside funnel definitions

---

## Section 6: Condition Types

Five condition types can appear inside `conditions` arrays:

| Type | Purpose |
|------|---------|
| `Value` | Filter by attribute column; also used for behavior queries (with `source`) |
| `Composite` | Boolean grouping with `expr` — the ONLY way to nest logic |
| `include` | Reference another segment (include its members) |
| `exclude` | Reference another segment (exclude its members) |

**HARD RULE: `type: And` and `type: Or` are NOT valid inside `conditions` arrays.** They render as blank/empty rules in the Console UI. Use `type: Composite` with an `expr` field for all boolean grouping — always.

The only place `And` and `Or` appear in valid YAML is `rule.type: And` / `rule.type: Or` at the top level (not inside a conditions array), or `filter.type: And` inside behavior conditions. The latter is safe; the former causes blank rendering.

---

## Section 7: Operators (All 18)

**18 valid operator types** — any other value triggers `INVALID_OPERATOR_TYPE`.

| Category | Types | Required Fields | Example |
|----------|-------|----------------|---------|
| Comparison | `Equal`, `NotEqual`, `Greater`, `GreaterEqual`, `Less`, `LessEqual` | `value` (string or number) | `type: Equal, value: "active"` |
| Range | `Between` | `min` and/or `max` (at least one) | `min: 18, max: 65` |
| Set | `In`, `NotIn` | `value` (array) | `value: ["DE", "AT"]` |
| Text | `Contain`, `StartWith`, `EndWith` | `value` (string array) | `value: ["@lufthansa.com"]` |
| Pattern | `Regexp` | `value` (string) | `value: "^[A-Z]{2}[0-9]{4}$"` |
| Null | `IsNull` | (none) | `type: IsNull` |
| Time | `TimeWithinPast`, `TimeWithinNext` | `value` + `unit` | `value: 30, unit: day` |
| Time | `TimeRange` | `duration` + `from` | See example below |
| Time | `TimeToday` | (none) | Matches today's date only |

**Negation**: Any operator supports `not: true` (e.g., `type: IsNull, not: true` = "is not null"; `type: Contain, value: ["test"], not: true` = "does not contain").

**Units** (singular form only — common mistake is `days` → must be `day`):
```
year | quarter | month | week | day | hour | minute | second
```

### TimeRange Example

"7-day window starting from 1 month ago":

```yaml
operator:
  type: TimeRange
  duration:
    day: 7                       # Window length
  from:
    last: 1                      # Starting point offset
    unit: month
```

---

## Section 8: Behavior Conditions

Query behavior table data with aggregations. Use `type: Value` with `source`, `aggregation`, and optionally `timeWindow` and `filter` fields.

```yaml
# Sum revenue for confirmed PNR bookings in last 90 days
- type: Value
  attribute: ""                           # Always empty string for behavior aggregations
  source: behavior_pnr_data               # behavior_<table_name> (prefix required)
  aggregation:
    type: Sum                             # Count | Sum | Average | Min | Max
    column: revenue_amount                # Required for Sum/Average/Min/Max (omit for Count)
  operator:
    type: Greater
    not: false
    value: 500
  timeWindow:                             # Optional: restrict to recent events
    duration: 90
    unit: day
  filter:                                 # Required when using source
    type: And                             # ONLY And is supported here — see hard rule below
    conditions:
      - type: Column                      # Use Column (not Value) inside filter
        column: res_status_categ_cd       # Use column (not attribute) field
        operator:
          type: Equal
          not: false
          value: "K"
      - type: Column
        column: cabin_class
        operator:
          type: In
          value: ["C", "F"]
```

**HARD RULE: `filter.type` MUST be `And`.** `filter.type: Or` is not supported server-side and will cause a validation error. If you need OR logic across filter conditions, express it as separate behavior conditions joined at the Composite level.

**Key rules for behavior conditions:**
- `attribute` must be `""` (empty string) — not omitted, not a column name
- `source` must be prefixed with `behavior_` (e.g., `behavior_web_all_events_stitched`)
- Inside `filter.conditions`: use `type: Column` with `column` field — not `type: Value` with `attribute`
- `filter` is required when `source` is present

---

## Section 9: Segment References (Include/Exclude)

Reference segments that already exist on the server by their exact name.

```yaml
rule:
  type: And
  conditions:
    - type: include
      segment: "Italian Top Spenders Base"   # Must match name exactly as shown in TD Console
    - type: exclude
      segment: "Opted Out EU"
```

**Limitation**: Cannot reference unpushed local segments. The segment must already exist on the server.

`MISSING_SEGMENT_REFERENCE` error means the `segment:` field is absent — add it. If push fails with a not-found error, verify the exact name in the TD Console.

---

## Section 10: Array Matching

Add `arrayMatching` to `Value` conditions to control how array-type attributes are matched:

```yaml
- type: Value
  attribute: travel_classes
  operator:
    type: In
    value: ["C", "F"]
  arrayMatching: any               # any | all | { atLeast: N } | { atMost: N } | { exactly: N }
```

Invalid keys or formats trigger `INVALID_ARRAY_MATCHING`.

---

## Section 11: Composite/expr — Mandatory for All Boolean Grouping

**HARD RULE:** Inside any `conditions` array (top-level `rule.conditions` or nested `Composite.conditions`), the ONLY valid condition types are:

- `Value` (attributes and behavior aggregations)
- `Composite` (boolean grouping)
- `include` / `exclude` (segment references)

**NEVER use `type: And` or `type: Or` as a condition.** The Console UI renders them as blank/empty rules. This applies at every depth. If you need boolean grouping, use `type: Composite` with an `expr` field — always.

**`filter.type: And` inside behavior conditions is SAFE — and it is the ONLY supported filter type.** `filter.type: Or` is not supported server-side and will cause a validation error. The blank-render bug only applies to `rule.conditions` and `Composite.conditions` nesting — behavior filter blocks are unaffected.

### expr Reference

- **Top-level** `rule.type: Composite` — ALWAYS reference conditions by **number** (1-indexed), regardless of how many branches there are:
  - 2 branches: `"(1 or 2)"`
  - 6 branches: `"(1 or 2 or 3 or 4 or 5 or 6)"`
  - Mixed: `"1 and (2 or 3 or 4)"`
  - **NEVER use letters at the top level** — this causes HTTP 400 "Referenced rule set does not exist"
- **Nested** `Composite` inside another Composite — reference by **letter** (A=first, B=second…): `"(A or B or C)"`

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
  expr: "(1 or 2 or 3 or 4)"       # 4 OR branches, each is an AND-group (numbers at top level)
  conditions:
    - type: Composite               # branch 1
      expr: "(A and B)"             # X AND D
      conditions: [...]
    - type: Composite               # branch 2
      expr: "(A and B)"             # X AND E
      conditions: [...]
    - type: Composite               # branch 3
      expr: "(A and B and C)"       # X AND F1 AND F2
      conditions: [...]
    - type: Composite               # branch 4
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
  expr: "(1 or 2 or 3 or 4)"
  conditions:
    # Branch 1: Italian + Top spender + ≥2 web searches
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

    # Branch 2: Italian + Top spender + ≥2 app searches
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

    # Branch 3: Italian + Top spender + 1 web + 1 app
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

    # Branch 4: Italian + Top spender + searches on different LH brand websites
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
- Top level: `Composite` with `expr` using numbers (`"(1 or 2 or 3 or 4)"`) — OR of 4 branches
- Each branch: `Composite` with `expr` using letters (`"(A and B and C)"`) — AND of conditions
- Italian identification inside each branch: `Composite` with OR of 2 values (letters)
- Behavior `filter.type: And` — safe, not affected by render bug
- Maximum Composite depth: exactly 2 (top OR → branch AND → Italian OR)
- Conditions that repeat (Italian, Top spender) are duplicated in each branch — this is required by DNF flattening

---

## Section 12: Error Codes

| Code | Cause | Solution |
|------|-------|----------|
| `MISSING_NAME` | Segment name is empty or missing | Add `name:` field |
| `INVALID_RULE_TYPE` | Rule type is not `And`, `Or`, or `Composite` | Check `type:` spelling |
| `MISSING_CONDITIONS` | Rule or group has no `conditions` array | Add conditions array |
| `EMPTY_ATTRIBUTE` | Attribute is empty on a non-behavior Value condition | Provide attribute name (or `""` for behavior) |
| `INVALID_OPERATOR_TYPE` | Operator type not in the 18 valid types | Check operator spelling — see Section 7 |
| `MISSING_OPERATOR_VALUE` | Operator requires `value` but it is missing | Add `value:` field |
| `MISSING_BETWEEN_BOUNDS` | `Between` has neither `min` nor `max` | Add `min:` and/or `max:` |
| `MISSING_TIME_UNIT` | Time operator missing `unit` | Add `unit: day` (singular — not `days`) |
| `INVALID_ARRAY_MATCHING` | `arrayMatching` has invalid format | Use `any`, `all`, or object form `{ atLeast: N }` |
| `MISSING_SEGMENT_REFERENCE` | `include`/`exclude` missing `segment` field | Add `segment:` with exact server name |
| `NESTED_CONDITION_GROUP` | Raw nested `And`/`Or` condition group (without `Composite`) | Use `type: Composite` with `expr`, or use `In` operator for same-attribute OR |
| `SEGMENT_SCHEMA_ERROR` | Server rejected the schema | Check field names — `column` (not `attribute`) inside `filter.conditions` |

---

## Section 13: Local vs Server Validation

| Check | `tdx sg validate` | `tdx sg push --dry-run` |
|-------|-------------------|-------------------------|
| YAML syntax | Yes | Yes |
| Operator types | Yes | Yes |
| Required fields | Yes | Yes |
| Nested groups rejected | Yes | Yes |
| Segment references exist on server | No | Yes |
| Behavior table schema | Partial | Yes |
| Field availability in parent segment | No | Yes |

Always run both validations before pushing. Local validation is fast; server validation catches schema and reference errors that local cannot.

---

## Section 14: Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Console shows blank/empty rules | Used `type: And` or `type: Or` inside a `conditions` array | Replace with `type: Composite` + `expr` |
| HTTP 400 "Referenced rule set does not exist" | Used letters in top-level `expr` | Top-level `expr` must use numbers (`1`, `2`, `3`…); letters only inside nested Composite |
| Behavior condition not matching | Wrong `filter` condition type | Use `type: Column` with `column` (not `type: Value` with `attribute`) |
| `filter.type: Or` validation error | OR not supported in behavior filter | `filter.type` must always be `And`; split OR branches into separate behavior conditions |
| `NESTED_CONDITION_GROUP` error | Raw nested `And`/`Or` without `Composite` | Use `type: Composite` + `expr`, or `In` operator for same-attribute values |
| `MISSING_TIME_UNIT` | Plural time unit | Use singular: `day` not `days`, `month` not `months` |
| `MISSING_BETWEEN_BOUNDS` | `Between` with no bounds | At least one of `min` or `max` required |
| Segment reference not found on server | Name mismatch or segment not pushed | Segment must exist on server; verify exact name in TD Console |
| Push slow or times out | Many Composite branches or long behavior windows | Use `include` for shared base conditions; use pre-computed attributes (e.g. `decile_monetary_24m`) instead of 730-day scans |
| 3-level Composite depth | Logic not flattened before writing | Flatten to DNF (OR of AND-groups) so maximum depth is 2; see Section 11 |
| Blank segment count | Rule too restrictive or wrong field name | Run a direct SQL query against `cdp_audience_1159510` with same conditions to debug |
