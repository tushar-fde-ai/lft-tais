---
name: segment-lhg
description: LHG segment creation — full YAML syntax, operators, Composite/expr patterns, behavior conditions, validation, and push workflow. All segment construction knowledge in one place; no external skill references needed.
---

# LHG Segment Creation

**MANDATORY FIRST STEP — run this once per session to create the project folder:**
```bash
tdx sg pull "cdp_lhg_unified_first_party"
```
This creates a `segments/cdp_lhg_unified_first_party/` folder with a `tdx.json` file. The `tdx.json` is required for `tdx sg validate` and `tdx sg push` to work. Run this once — subsequent segments go into the same folder.

---

## Section 1: File Location

All segment YAML files go into the folder created by `tdx sg pull`:
```
segments/cdp_lhg_unified_first_party/<segment-name-kebab-case>.yml
```

If the folder doesn't exist yet, run the pull command from the mandatory first step above.

---

## Section 2: Workflow

**Process one segment at a time.** For each segment:

1. **Pull** (once per session) — `tdx sg pull "cdp_lhg_unified_first_party"` to create project folder with `tdx.json`
2. **Write** the YAML file into the pulled folder (`segments/cdp_lhg_unified_first_party/`)
3. **Pre-push nesting check** — count Composite levels in the YAML before proceeding:
   - If any Composite contains another Composite that itself contains another Composite → 3 levels → **stop, flatten to DNF first** (see Section 11)
   - Maximum 2 levels. Pushing a 3-level structure will succeed but render blank in Console UI
4. **Validate** with `tdx sg validate <file>`
5. **Count check** — run `tdx sg sql --path <file> | tdx query -` and verify count > 0
   - If count is 0, the rule is too restrictive — revise before proceeding
6. **Preview** with `preview_segment` tool — get user approval before proceeding
7. **Push** with `tdx sg push -y "<file>"` — always specify the file path explicitly

Never batch multiple segments in validate or push operations.

After push succeeds, display the Console link:
```
https://console.treasuredata.com/app/audiences/<parent_id>/segments/<segment_id>
```

---

## Section 3: Core Commands

```bash
tdx sg pull "cdp_lhg_unified_first_party"  # Run once — creates project folder with tdx.json
tdx sg validate <file>               # Local YAML validation (file must be inside pulled folder)
tdx sg push --dry-run "<file>"       # Server-side validation
tdx sg push -y "<file>"              # Push specific file (-y for non-interactive)
tdx sg list                          # List segments
tdx sg list -r                       # Recursive tree view
tdx sg fields                        # List available fields
tdx sg sql --path <file> | tdx query -  # Count check (file must be inside pulled folder)
```

For count checks, use `tdx sg sql --path <file> | tdx query -` to verify segment size > 0 before pushing. Alternatively, write a direct SQL query against `cdp_audience_1159510` using the same conditions from the YAML.

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

### Simple segment (all conditions AND'd together):

```yaml
name: High Value Customers          # Required
kind: batch                         # batch | realtime | funnel_stage

rule:
  type: And
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

### Multi-branch segment (OR of AND-groups):

```yaml
name: Premium Customers - Multi Channel
kind: batch

rule:
  type: Composite
  expr: "(1 or 2)"
  conditions:
    - type: And
      conditions:
        - type: Value
          attribute: country
          operator:
            type: Equal
            value: "DE"
        - type: Value
          attribute: ltv
          operator:
            type: Greater
            value: 1000
    - type: And
      conditions:
        - type: Value
          attribute: country
          operator:
            type: Equal
            value: "AT"
        - type: Value
          attribute: ltv
          operator:
            type: Greater
            value: 500
```

**`kind` values:**
- `batch` — standard scheduled evaluation
- `realtime` — real-time profile evaluation
- `funnel_stage` — used inside funnel definitions

---

## Section 6: Condition Types

What is valid inside `conditions` depends on the **parent context**:

### When `rule.type` is `Composite` (top-level with `expr`):

| Type | Purpose |
|------|---------|
| `And` | Branch where all conditions must match (NO `expr` field) |
| `Or` | Branch where any condition must match (NO `expr` field) |
| `Value` | Filter by attribute column; also used for behavior queries (with `source`) |
| `include` | Reference another segment (include its members) |
| `exclude` | Reference another segment (exclude its members) |

**`type: Composite` with `expr` is NOT valid here** — the server rejects nested Composite inside a top-level Composite.

### When `rule.type` is `And` or `Or` (top-level without `expr`):

| Type | Purpose |
|------|---------|
| `Value` | Filter by attribute column; also used for behavior queries (with `source`) |
| `Composite` | Boolean grouping with `expr` (numbers for conditions) |
| `include` | Reference another segment (include its members) |
| `exclude` | Reference another segment (exclude its members) |

**`type: And` / `type: Or` are NOT valid here** — they render as blank/empty rules in the Console UI.

### Inside an `And`/`Or` branch (inside a top-level Composite):

| Type | Purpose |
|------|---------|
| `Value` | Filter by attribute column; also used for behavior queries (with `source`) |
| `include` | Reference another segment (include its members) |
| `exclude` | Reference another segment (exclude its members) |

**No further nesting** — you cannot put `Composite`, `And`, or `Or` inside a branch's conditions. All OR logic must be fully flattened into separate top-level branches.

### Summary table

| Level | Valid types | `expr` field |
|-------|------------|--------------|
| `rule:` | `And`, `Or`, `Composite` | Only when `type: Composite` |
| Inside `rule.conditions` when rule is `Composite` | `And`, `Or`, `Value`, `include`, `exclude` | **NOT valid** |
| Inside `rule.conditions` when rule is `And`/`Or` | `Composite`, `Value`, `include`, `exclude` | Only on `Composite` |
| Inside a branch (`And`/`Or`) of a top-level `Composite` | `Value`, `include`, `exclude` | **NOT valid** |

**`filter.type: And` inside behavior conditions is SAFE — and it is the ONLY supported filter type.** `filter.type: Or` is not supported server-side and will cause a validation error.

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
  source: behavior_pnr                    # behavior_<table_name> (prefix required)
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

## Section 11: Composite/expr — Multi-Branch Segments

### The Two Valid Patterns

**Pattern A — Simple (single boolean, no branching):**
```yaml
rule:
  type: And          # or Or
  conditions:
    - type: Value
      ...
    - type: Value
      ...
```

**Pattern B — Multi-branch (OR of AND-groups, or complex boolean):**
```yaml
rule:
  type: Composite
  expr: "(1 or 2 or 3)"          # ALWAYS numbers at top level
  conditions:
    - type: And                   # branch 1 — And/Or, NO expr field
      conditions:
        - type: Value
          ...
        - type: Value
          ...
    - type: And                   # branch 2
      conditions:
        - type: Value
          ...
    - type: And                   # branch 3
      conditions:
        - type: Value
          ...
```

### Hard Rules

1. **`expr` is only valid on the top-level `rule` when `type: Composite`** — never on branches inside it
2. **Branches inside a top-level Composite must be `type: And` or `type: Or`** — never `type: Composite`
3. **No nesting inside branches** — branches contain only `Value`, `include`, and `exclude` conditions. If you need OR logic within a branch (e.g. country = IT OR phone = +39), flatten it into separate top-level branches instead
4. **Top-level `expr` uses numbers** (1-indexed): `"(1 or 2 or 3)"`, `"1 and (2 or 3)"`
5. **NEVER use letters in top-level `expr`** — causes HTTP 400 "Referenced rule set does not exist"

### `filter.type: And` inside behavior conditions

This is SAFE and is the ONLY supported filter type. `filter.type: Or` is not supported server-side. The nesting restrictions above apply only to `rule.conditions` — behavior filter blocks are unaffected.

### Flattening — OR Within a Branch

If your logic has OR conditions within an AND-group, you MUST flatten by creating separate branches.

**Example:** "Italian customers (by country OR phone) AND top spenders AND web searchers"

Logical structure: `(A OR B) AND C AND D`

This cannot be expressed as one branch with nested OR. Flatten to separate branches:

```
(A AND C AND D) OR (B AND C AND D)
```

```yaml
name: Italian High-Spender Flight Searchers
kind: batch
description: >
  Italian customers (by country or phone prefix) who are top 10% spenders
  and searched flights on the web in last 90 days.

rule:
  type: Composite
  expr: "(1 or 2)"
  conditions:
    # Branch 1: Italian by country + Top spender + Web searcher
    - type: And
      conditions:
        - type: Value
          attribute: cntry_cd
          operator:
            type: Equal
            value: "it"
        - type: Value
          attribute: ""
          source: behavior_pnr_leg
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
        - type: Value
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

    # Branch 2: Italian by phone + Top spender + Web searcher (same C and D, different A)
    - type: And
      conditions:
        - type: Value
          attribute: phone_cntry_cd
          operator:
            type: Equal
            value: "+39"
        - type: Value
          attribute: ""
          source: behavior_pnr_leg
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
        - type: Value
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
```

**Key points:**
- Top level: `Composite` with `expr: "(1 or 2)"` — numbers only
- Each branch: `type: And` — NO `expr` field, NO `type: Composite`
- The Italian OR (country vs phone) is flattened into 2 separate branches, each duplicating the shared conditions (top spender + web searcher)
- Behavior `filter.type: And` inside each Value condition — safe, unaffected by nesting rules
- Maximum depth: `Composite` → `And` → `Value` (exactly 2 levels, no deeper)

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
| `NESTED_CONDITION_GROUP` | Nested `And`/`Or` inside another `And`/`Or` | Restructure: use `Composite` at rule level with `And`/`Or` branches, or use `In` operator for same-attribute OR |
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
| Console shows blank/empty rules | Used `type: And`/`Or` inside a top-level `And`/`Or` conditions array | At top-level `And`/`Or`, use `Composite` for grouping. Inside a top-level `Composite`, use `And`/`Or` for branches (see Section 6 & 11) |
| HTTP 400 "Referenced rule set does not exist" | Used letters in top-level `expr`, or used `Composite` with `expr` inside a top-level `Composite` | Top-level `expr` must use numbers; branches must be `type: And`/`Or` without `expr` |
| Nested Composite rejected by server | Put `type: Composite` with `expr` inside a top-level `Composite`'s conditions | Branches inside a top-level Composite must be `type: And` or `type: Or` — never `Composite` |
| OR logic within a branch needed | Cannot nest `Or` inside an `And` branch | Flatten: create separate top-level branches for each OR path (see Section 11) |
| Behavior condition not matching | Wrong `filter` condition type | Use `type: Column` with `column` (not `type: Value` with `attribute`) |
| `filter.type: Or` validation error | OR not supported in behavior filter | `filter.type` must always be `And`; split OR branches into separate behavior conditions |
| `NESTED_CONDITION_GROUP` error | Nested `And`/`Or` inside another `And`/`Or` | Use `In` operator for same-attribute values, or restructure with `Composite` at rule level |
| `MISSING_TIME_UNIT` | Plural time unit | Use singular: `day` not `days`, `month` not `months` |
| `MISSING_BETWEEN_BOUNDS` | `Between` with no bounds | At least one of `min` or `max` required |
| Segment reference not found on server | Name mismatch or segment not pushed | Segment must exist on server; verify exact name in TD Console |
| Push slow or times out | Many branches or long behavior windows | Use `include` for shared base conditions; use pre-computed attributes (e.g. `decile_monetary_24m`) instead of 730-day scans |
| Blank segment count | Rule too restrictive or wrong field name | Run a direct SQL query against `cdp_audience_1159510` with same conditions to debug |
