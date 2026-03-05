# Phone Validation Investigation Summary

## Problem

Phone numbers valid in the [nyaruka/phonenumbers](https://github.com/nyaruka/phonenumbers) Go library fail parsing/validation in the [global_phone](https://github.com/sstephenson/global_phone) Ruby library when using metadata regenerated from Google's current [`PhoneNumberMetadata.xml`](https://github.com/google/libphonenumber/blob/master/resources/PhoneNumberMetadata.xml).

Example: `+5491164102945` (Argentina mobile) — parses and validates in Go, but fails completely in Ruby with regenerated metadata.

## Root Cause

Google removed `<possibleNumberPattern>` from the XML and replaced it with `<possibleLengths>` attributes. The Ruby `database_generator.rb` still looks for the old element, producing `null` for ALL 254 territories.

## Issues Found

| Issue | Severity | Details |
|-------|----------|---------|
| [#1: null possible_length](issue-01-null-possible-length.md) | **CRITICAL** | All 254 territories have null — breaks all parsing |
| [#2: National pattern changes](issue-02-national-pattern-changes.md) | Low | Updated regexes from Google — functionally equivalent |
| [#3: Territory field count](issue-03-territory-field-count.md) | None | Missing 4th field compensated by format-level fallback |

## Fix

Patch `database_generator.rb` to parse the new `<possibleLengths>` XML format. See [library-patch-guide.md](library-patch-guide.md) for implementation details.

**Single file change**: `lib/global_phone/database_generator.rb`

The patch converts `<possibleLengths national="10,11" localOnly="[6-8]"/>` into regex patterns like `\d{6,11}`, restoring the same behavior as the old `<possibleNumberPattern>` element. After patching, all 254 territories produce valid possible length patterns that are fully consistent with the Go library's metadata.
