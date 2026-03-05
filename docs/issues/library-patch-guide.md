# Library Patch Guide: global_phone

## Target

**File**: `lib/global_phone/database_generator.rb`

## Background

Google replaced `<possibleNumberPattern>` with `<possibleLengths>` in their [`PhoneNumberMetadata.xml`](https://github.com/google/libphonenumber/blob/master/resources/PhoneNumberMetadata.xml):

```
BEFORE (removed):
  <generalDesc>
    <possibleNumberPattern>\d{6,11}</possibleNumberPattern>
  </generalDesc>

AFTER (current):
  <fixedLine>
    <possibleLengths national="10" localOnly="[6-8]"/>
  </fixedLine>
  <mobile>
    <possibleLengths national="10,11" localOnly="[6-8]"/>
  </mobile>
```

The `possibleLengths` element uses:
- `national` attribute: comma-separated numbers and/or bracket ranges (e.g., `"10,11"`, `"[5-12]"`)
- `localOnly` attribute (optional): same format, for local-only valid lengths

Note: `<possibleLengths>` elements are NOT inside `<generalDesc>` — they're inside sub-elements like `<fixedLine>`, `<mobile>`, `<tollFree>`, etc.

## What Was Changed

### 1. Modified `compile_territory` method

Replaced `pattern(node, "generalDesc possibleNumberPattern")` with `possible_lengths_pattern(node)`.

### 2. Added two new methods

**`possible_lengths_pattern(node)`** — Collects all `<possibleLengths>` elements from the territory node (across all sub-types), combines `national` and `localOnly` values, and generates a `\d{min,max}` regex.

**`parse_length_values(value)`** — Parses the attribute format into integer arrays:

```
"10"                    -> [10]
"10,11"                 -> [10, 11]
"[6-8]"                 -> [6, 7, 8]
"10,[14-16]"            -> [10, 14, 15, 16]
"[5-8],10,11,[14-16]"   -> [5, 6, 7, 8, 10, 11, 14, 15, 16]
```

## Why This Works

The old `<possibleNumberPattern>` was always a simple `\d{min,max}` regex. The min/max range approach:

1. Preserves the **length validation** behavior
2. Preserves the **national prefix transform guard** (rejects over-long transformed strings)
3. Is slightly more permissive for gaps (e.g., national="7,9,10" allows 8 too) — this is acceptable because the `national_pattern` regex provides the precise validation

## Verification

After patching, all 254 territories produce valid possible length patterns that are fully consistent with the [nyaruka/phonenumbers](https://github.com/nyaruka/phonenumbers) Go library's metadata.

Regenerate and verify:
```bash
rake generate
```
