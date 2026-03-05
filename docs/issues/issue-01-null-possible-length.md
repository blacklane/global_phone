# Issue 01: null `possible_length` in ALL territories

## Summary

The generated JSON has `null` for the `possible_length` field (territory index 1) in **ALL 254 territories**. This completely breaks phone number parsing.

## Root Cause

Google removed the `<possibleNumberPattern>` XML element from [`PhoneNumberMetadata.xml`](https://github.com/google/libphonenumber/blob/master/resources/PhoneNumberMetadata.xml) and replaced it with `<possibleLengths>` attribute-based elements.

**Old XML format** (no longer present):
```xml
<generalDesc>
  <possibleNumberPattern>\d{6,11}</possibleNumberPattern>
  <nationalNumberPattern>...</nationalNumberPattern>
</generalDesc>
```

**New XML format**:
```xml
<generalDesc>
  <nationalNumberPattern>(?:11|[89]\d\d)\d{8}|[2368]\d{9}</nationalNumberPattern>
</generalDesc>
<fixedLine>
  <possibleLengths national="10" localOnly="[6-8]"/>
  ...
</fixedLine>
<mobile>
  <possibleLengths national="10,11" localOnly="[6-8]"/>
  ...
</mobile>
```

The `database_generator.rb` still looked for the old element:

```ruby
# database_generator.rb (before patch)
def compile_territory(node)
  [
    territory_name(node),
    pattern(node, "generalDesc possibleNumberPattern"),  # Returns nil — element removed
    pattern(node, "generalDesc nationalNumberPattern"),
    squish(node["nationalPrefixFormattingRule"])
  ]
end
```

## How It Breaks Parsing

In `territory.rb`:

```ruby
def possible?(string)
  string =~ possible_pattern  # string =~ nil → nil (falsy)
end
```

When `possible_pattern` is nil, `possible?` always returns falsy. The `parse_national_string` method requires `possible?` to pass:

```ruby
def parse_national_string(string)
  string = normalize(string)
  Number.new(self, string) if possible?(string)  # Never creates Number
end
```

## The Dual-Purpose Guard Role

The `possible_length` regex isn't just a length check — it also guards against incorrect national prefix transforms. Full trace for `+5491164102945`:

```
1. Input: +5491164102945
2. strip_leading_plus → 5491164102945
3. region_for_string → Region 54 (Argentina)
4. strip_country_code → 91164102945 (11 digits)
5. Territory AR: parse_national_string("91164102945")
6.   normalize("91164102945") calls strip_national_prefix
7.   strip_national_prefix("91164102945"):
       - nationalPrefixForParsing: /^(?:0?(?:(11|2...|3...)15)?)/
       - nationalPrefixTransformRule: "9$1" → gsub $ to \ → "9\1"
       - "91164102945".sub(regex, "9\1")
       - The regex matches EMPTY STRING at position 0 (everything is optional)
       - Capture group $1 is nil → \1 is empty
       - Result: "9" inserted at position 0 → "991164102945" (12 digits!)
8.   possible?("991164102945") with \d{6,11} → 12 digits FAILS → falls back to original
9.   possible?("91164102945") with \d{6,11} → 11 digits PASSES
10.  Returns Number.new(self, "91164102945")
```

With `null` possible_pattern: step 8 returns nil → step 9 returns nil → parse fails.
With `\d+` (too permissive): step 8 returns truthy → keeps "991164102945" (12 digits) → no format matches → validation fails.

## Impact

- **254/254 territories** produce null `possible_length` → no phone number can be parsed at all.

## Fix

Patch `compile_territory` to collect `<possibleLengths>` from all sub-elements (`fixedLine`, `mobile`, etc.) and convert to a `\d{min,max}` regex. See [library-patch-guide.md](library-patch-guide.md).
