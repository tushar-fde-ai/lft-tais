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
3. **Count check** — run `tdx sg sql --path <file> | tdx query -` and verify count > 0
   - If count is 0, the rule is too restrictive — revise before proceeding
4. **Preview** with `preview_segment` tool — get user approval before proceeding
5. **Push** with `tdx sg push -y "<file>"` — always specify the file path explicitly

Never batch multiple segments in validate or push operations.

After push succeeds, display the Console link:
```
https://console.treasuredata.com/app/audiences/<parent_id>/segments/<segment_id>
```

## Core Commands

```bash
# DO NOT RUN:
# tdx sg use "cdp_lhg_unified_first_party"   ← slow, unnecessary
# tdx sg pull "cdp_lhg_unified_first_party"  ← pulls 24.3M row segment

# Use these instead:
tdx sg validate <file>               # Validate specific file locally
tdx sg push --dry-run "<file>"       # Server-side validation
tdx sg push -y "<file>"              # Push specific file (-y for non-interactive)
tdx sg list                          # List segments
tdx sg list -r                       # Recursive tree view
tdx sg fields                        # List available fields
```

---

For all YAML syntax, operators, Composite/expr patterns, behavior conditions, and error codes — see the **segment** and **validate-segment** skills.
