---
name: grok-to-opal
description: >
  Use this skill whenever the user wants to convert a Logstash Grok pattern or
  Grok filter into an equivalent OPAL query using extract_regex. Trigger phrases
  include: "convert this grok", "translate grok to OPAL", "migrate grok filter",
  "grok pattern in OPAL", "what is the OPAL equivalent of this grok",
  "rewrite this grok as an extract_regex", or any time the user pastes a Grok
  expression containing %{PATTERN_NAME} syntax and wants the OPAL version.
metadata:
  version: "0.1.0"
---

# Grok → OPAL Skill

Convert Logstash Grok patterns and filters into OPAL `extract_regex` statements.
Read `references/grok-patterns.md` for the complete Grok-to-OPAL pattern mapping table and pre-built translations for common log formats.

---

## Grok syntax primer

Grok has three forms:

| Grok form | Meaning |
|---|---|
| `%{PATTERN}` | Match the pattern, discard (no capture) |
| `%{PATTERN:field_name}` | Match and capture into `field_name` |
| `%{PATTERN:field_name:type}` | Match, capture, and cast to `type` |

Types: `:int` (→ `int64`), `:long` (→ `int64`), `:float` (→ `float64`), `:double` (→ `float64`).

ECS-style field names use nested bracket notation: `[http][response][status_code]` — flatten to `http_response_status_code` for OPAL column names.

---

## Translation algorithm

Apply these steps in order to every Grok pattern the user provides:

**Step 1 — Recognize composite patterns**
Check `references/grok-patterns.md` "Pre-built OPAL translations" section. If the top-level pattern name matches a pre-built entry (e.g. `HTTPD_COMBINEDLOG`, `SYSLOGLINE`, `CATALINALOG`), emit that translation directly — do not expand manually.

**Step 2 — Expand `%{PATTERN:field:type}` tokens**
For each token in the pattern:
1. Look up `PATTERN` in the mapping table in `references/grok-patterns.md` to get the OPAL regex.
2. Normalize the field name:
   - Strip all `[` and `]`, replace `][` with `_` → `[http][response][status_code]` → `http_response_status_code`
   - Keep plain names as-is
3. Map the type: `:int` or `:long` → `::int64`; `:float` or `:double` → `::float64`
4. Produce: `(?P<field_name::opal_type>opal_regex)` with type, or `(?P<field_name>opal_regex)` without
5. For `%{PATTERN}` with no field name: emit the raw OPAL regex inline (no named group)

**Step 3 — Preserve literal characters**
All literal text between `%{...}` tokens (spaces, brackets, quotes, dashes) is kept exactly as-is. Escape any OPAL regex metacharacters that appear literally: `.` `(` `)` `[` `]` `{` `}` `+` `?` `*` `^` `$` `|` `\`.

**Step 4 — Handle optional/nullable fields**
Grok `(?:-|%{PATTERN:field})` means "a literal dash OR the pattern". In OPAL:
```
(?:-|(?P<field>pattern))
```

**Step 5 — Assemble the final statement**
```
extract_regex body, /assembled_pattern/
```

Always name the captured message/remainder field `extracted_message`. Subsequent `get_regex()` and further parsing calls operate on `extracted_message`, not on `body`.

If the pattern contains a timestamp capture, add a timestamp pipeline after extraction. Choose the parse function based on the Grok pattern:

| Grok timestamp pattern | Parse function | Format string |
|---|---|---|
| `TIMESTAMP_ISO8601` | `parse_isotime(ts_str)` | — (auto) |
| `HTTPDATE` | `parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")` | Apache CLF |
| `SYSLOGTIMESTAMP` | `parse_timestamp(ts_str, "MON DD HH24:MI:SS", "UTC")` | Syslog (no tz) |
| `HTTPDERROR_DATE` | `parse_timestamp(ts_str, "DY MON DD HH24:MI:SS.FF3 YYYY")` | Apache error log |
| `DATESTAMP_RFC2822` | `parse_timestamp(ts_str, "DY, DD MON YYYY HH24:MI:SS TZHTZM")` | RFC-2822 |

```
// ISO 8601 timestamps (TIMESTAMP_ISO8601):
| make_col _ts: parse_isotime(ts_str)

// Non-ISO timestamps (HTTPDATE, SYSLOGTIMESTAMP, etc.):
| make_col _ts: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")

| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts
```

`parse_timestamp()` returns NULL (not an error) when the string doesn't match — safe in pipelines. `parse_isotime()` auto-detects ISO variants including fractional seconds and timezone offsets.

---

## Type mapping

| Grok type | OPAL inline cast | OPAL `make_col` cast |
|---|---|---|
| `:int` | `::int64` | `int64(col)` |
| `:long` | `::int64` | `int64(col)` |
| `:float` | `::float64` | `float64(col)` |
| `:double` | `::float64` | `float64(col)` |
| `:boolean` | `::bool` (if supported) | `bool(col)` |
| _(none)_ | _(omit)_ | `string(col)` |

---

## ECS field name normalization

Grok ECS patterns use `[group][subgroup][field]` syntax. OPAL column names must be flat identifiers. Normalize by:
- Removing outer brackets: `[source][address]` → `source_address`
- Joining levels with `_`: `[http][response][status_code]` → `http_response_status_code`
- Special case — bare names with no brackets stay as-is: `timestamp`, `message`

Recommended short aliases (use when the ECS path is long):
| ECS path | Short alias |
|---|---|
| `[source][address]` | `client_ip` |
| `[http][response][status_code]` | `status_code` |
| `[http][response][body][bytes]` | `bytes_sent` |
| `[http][request][method]` | `http_method` |
| `[url][original]` | `url` |
| `[user_agent][original]` | `user_agent` |
| `[log][level]` | `log_level` |
| `[process][pid]` | `pid` |
| `[host][hostname]` | `hostname` |
| `[user][name]` | `username` |

---

## Output format

Always produce:
1. **The `extract_regex` statement** with complete regex
2. **Column list** — table showing each named capture → OPAL column name → type
3. **Timestamp pipeline** if any timestamp field was captured
4. **Notes** on any patterns that were simplified (e.g. full IPv6 regex replaced with `\S+`)

Example output format:

```
// Captures: client_ip (string), username (string), ts_str (string → timestamp),
//           http_method (string), url (string), status_code (int64), bytes_sent (int64),
//           extracted_message (string) — any trailing message field

extract_regex body, /(?P<client_ip>\S+) \S+ (?P<username>\S+) \[(?P<ts_str>[^\]]+)\] "(?P<http_method>\w+) (?P<url>\S+) [^"]+" (?P<status_code::int64>\d+) (?P<bytes_sent::int64>-|\d+)/

// HTTPDATE format → parse_timestamp with explicit format string
| make_col _ts: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

## Common pitfalls

- **`GREEDYDATA` at end of pattern**: maps to `.*` — works fine at end, but can over-match in the middle. Use `DATA` (`.*?`) for non-greedy interior captures.
- **`QUOTEDSTRING`**: The full Grok definition handles escaped quotes; use `"[^"]*"` for simple cases or `"(?:[^"\\]|\\.)*"` for escaped-quote-aware.
- **IPv6 in `IP`/`IPV6`**: Full IPv6 regex is very long. Use `\S+` as a simplified substitute when the distinction doesn't matter.
- **`SYSLOGPROG`**: Captures both `process_name` and `pid` with format `name[pid]`. In OPAL: `(?P<process_name>[^\[]+)(?:\[(?P<pid::int64>\d+)\])?`
- **Nested `%{...}` in Grok**: Some patterns reference other patterns recursively (e.g. `HTTPD_COMBINEDLOG` includes `HTTPD_COMMONLOG`). Always use the pre-built translation from `references/grok-patterns.md` for these.
- **`(?<name>...)` raw regex in Grok**: Grok also supports inline named groups. These map directly to OPAL `(?P<name>...)` — just replace `?<` with `?P<`.
