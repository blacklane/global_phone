# Issue 02: National Pattern Regex Changes

## Summary

The `nationalNumberPattern` regex changed between old and new metadata for many countries. The changes come from Google updating the patterns in [`PhoneNumberMetadata.xml`](https://github.com/google/libphonenumber/blob/master/resources/PhoneNumberMetadata.xml). These changes are mostly cosmetic/equivalent.

## Argentina Example (Country Code 54)

| Version | Pattern |
|---------|---------|
| Old | `11\d{8}\|[2368]\d{9}\|9\d{10}` |
| New | `(?:11\|[89]\d\d)\d{8}\|[2368]\d{9}` |

### Analysis

Both patterns match the test number `91164102945` (national part of `+5491164102945`):

**Old pattern** `11\d{8}|[2368]\d{9}|9\d{10}`:
- `9\d{10}` matches `91164102945` (9 + 10 digits = 11 total)

**New pattern** `(?:11|[89]\d\d)\d{8}|[2368]\d{9}`:
- `[89]\d\d` matches `911`, then `\d{8}` matches `64102945` (total: 11 digits)

The new pattern is more precise: instead of the broad `9\d{10}`, it uses `[89]\d\d\d{8}` which correctly models that Argentina mobile numbers start with `9` followed by area codes, and toll-free/premium numbers start with `8xx`.

## Impact

The national pattern changes do NOT cause validation failures on their own. The parsing failure (Issue 01) must be fixed first. Once parsing works, the updated patterns correctly validate all numbers that the [nyaruka/phonenumbers](https://github.com/nyaruka/phonenumbers) Go library accepts.

## Note on Regex Anchoring

The `global_phone` gem wraps patterns with `^` and `$` anchors (`territory.rb:8`):

```ruby
field 2, :national_pattern do |p| /^#{p}$/ end
```

Due to regex alternation precedence, `/^A|B$/` is `(^A)|(B$)`, not `^(A|B)$`. Both old and new patterns work correctly with this behavior because the first alternative is start-anchored and the last is end-anchored, and all alternatives produce the correct total length.
