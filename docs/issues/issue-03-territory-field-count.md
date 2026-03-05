# Issue 03: Territory Field Count Difference

## Summary

Old metadata territories have 4 fields; new metadata territories have 3 fields. The 4th field (`nationalPrefixFormattingRule`) is missing in the new output.

## Structure Comparison

**Old format** (4 fields):
```json
["AR", "\\d{6,11}", "11\\d{8}|[2368]\\d{9}|9\\d{10}", "$NP$FG"]
```

| Index | Field | Value |
|-------|-------|-------|
| 0 | name | "AR" |
| 1 | possible_pattern | "\\d{6,11}" |
| 2 | national_pattern | "11\\d{8}\|..." |
| 3 | national_prefix_formatting_rule | "$NP$FG" |

**New format** (3 fields):
```json
["AR", "\\d{6,11}", "(?:11|[89]\\d\\d)\\d{8}|[2368]\\d{9}"]
```

| Index | Field | Value |
|-------|-------|-------|
| 0 | name | "AR" |
| 1 | possible_pattern | "\\d{6,11}" |
| 2 | national_pattern | "(?:11\|[89]\\d\\d)\\d{8}\|..." |
| 3 | national_prefix_formatting_rule | (missing) |

## Why This Happens

The `nationalPrefixFormattingRule` is an attribute on the `<territory>` XML element. In the current XML, this attribute is absent for many territories, causing `compile_territory` to return `nil` for field 3, which then gets truncated by the `truncate()` method.

## Impact: NONE

The `global_phone` gem handles this gracefully via fallback. In `number.rb`:

```ruby
def national_prefix_formatting_rule
  format.national_prefix_formatting_rule || territory.national_prefix_formatting_rule
end
```

The format-level `nationalPrefixFormattingRule` (field 3 of each format entry) takes priority over the territory-level rule. In the new metadata, formatting rules are embedded at the format level. The fallback chain works correctly and produces correct national format output.

No patch required for this issue.
