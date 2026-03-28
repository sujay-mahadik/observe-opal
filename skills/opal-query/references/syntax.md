# OPAL Syntax Reference

OPAL (Observe Processing and Analytic Language) is a pipeline-based query language. Clauses are chained with `|` — each produces a table that feeds the next. All verbs are lowercase; functions appear inside expressions.

---

## Pipeline basics

```
filter namespace = "production"
| make_col service: string(attributes["service.name"])
| statsby count:count(1), group_by(service)
```

`|` is optional at the start of a line when the statement is on its own line.

---

## Row filtering — `filter`

```
filter body ~ /error|fail|exception/i        # regex match (case-insensitive)
filter service_name = "payment"              # exact string match
filter pod ~ /payment/                       # partial regex match
filter error = true                          # boolean
filter status_code >= 500                    # numeric comparison
filter body ~ /error/i and pod ~ /payment/   # AND
filter namespace = "prod" or namespace = "staging"  # OR
filter not (error = false)                   # NOT
filter isnull(user_id)                       # null check
filter not isnull(user_id)                   # not null
```

**`filter_last`** — filter resources based on their most recent value:
```
filter_last status = "active"
```

**`always` / `ever` / `never`** — resource-level temporal filters:
```
always error = true, frame(back: 30m)   # resource had error at ALL points in last 30m
ever   error = true, frame(back: 30m)   # resource had error at SOME point in last 30m
never  error = true, frame(back: 30m)   # resource had error at NO point in last 30m
```

---

## Column manipulation

### `make_col` — add or transform columns
```
make_col service: string(attributes["service.name"])
make_col duration_ms: duration / 1000000       # nanoseconds → ms
make_col is_error: case(error = true, 1, true, 0)
make_col upper_name: upper(pod)
make_col full_path: concat_strings(namespace, "/", pod)
```

### `pick_col` — keep only specified columns (drops all others)
```
pick_col timestamp, body, pod, namespace
pick_col start_time, end_time, duration, service_name, error
# With rename:
pick_col ts:timestamp, msg:body, svc:service_name
```
> **Spans gotcha**: Always include `start_time` and `end_time` — omitting either fails with `need to pick 'valid from' column`.

### `drop_col` — remove columns
```
drop_col debug_info, raw_payload
drop_col timestamp, _ts          # multiple columns
```

### `rename_col` — rename columns
```
rename_col service:service_name, ts:timestamp
```

---

## Field extraction from raw log strings

### `extract_regex` verb — creates columns from named capture groups
**Syntax:** `extract_regex column, /(?P<name>pattern)/, [flags: "i"]?`

This is the preferred approach — one statement creates multiple typed columns:
```
# Basic: all fields from a CLF access log
extract_regex body, /(?P<client_ip>[\d.]+) \S+ \S+ \[(?P<ts_str>[^\]]+)\] "(?P<method>\w+) (?P<path>\S+) [^"]+" (?P<status::int64>\d+) (?P<bytes::int64>\d+)/

# Bracket-structured app log: [2026-03-28 10:14:32] [ERROR] [RequestHandler] ...
extract_regex body, /\[(?P<ts_str>[^\]]+)\] \[(?P<level>[^\]]+)\] \[(?P<component>[^\]]+)\] (?P<log_body>.*)/

# Auth log
extract_regex body, /(?P<log_level>\w+)\s+\[(?P<service>\w+)\].*user:\s+(?P<user_email>\S+@\S+).*IP:\s+(?P<client_ip>[\d.]+)/
```

Named group: `(?P<column_name>pattern)` → creates a string column.
Inline typecast: `(?P<name::int64>pattern)` — supported casts: `float64`, `int64`, `string`, `parse_isotime` (→ timestamp), `duration`, `duration_ms`, `duration_sec`, `duration_min`, `duration_hr`, `parse_json`.
Flags: `"i"` = case-insensitive, `"m"` = multiline, `"s"` = dot matches newline. Pass as second argument: `extract_regex body, /pattern/, flags: "i"`.

#### ⚠️ Regex engine constraints — forbidden constructs

OPAL's regex engine **does not support** the following. Using them causes `Invalid regular expression: '...', no argument for repetition operator: ?`.

| Forbidden | Error reason | Safe replacement |
|---|---|---|
| `(?:...)` non-capturing group | `?` seen as bare quantifier | Remove grouping, or use a named group `(?P<x>...)` |
| `(?:foo\|bar)?` optional group | same | Use `\w*` / `[^\s]*` or restructure around delimiters |
| `(?i)` inline flags | same | Use `flags: "i"` argument |
| `(?=...)` lookahead | not supported | Anchor on surrounding characters instead |
| `(?<=...)` lookbehind | not supported | Use a named group to capture the preceding token |
| `\1` backreference | not supported | Restructure — capture each part with distinct named groups |

**Key rule:** every `(?` must be followed by `P<name>` — named capture groups are the only supported `(?` form.

#### Timestamps — always use the two-step pattern

Timestamp formats that include optional fractional seconds or timezone offsets require `(?:...)` groups in a single regex, which OPAL rejects. Use two steps instead:

**Step 1 — extract raw timestamp string** (pick the pattern that matches your format):
```
// Bracket-delimited: [2026-03-28 10:14:32] or [28/Mar/2026:10:22:15 +0000]
extract_regex body, /\[(?P<ts_str>[^\]]+)\]/

// ISO prefix, no brackets: 2026-03-28T10:14:32Z or 2026-03-28 10:14:32
extract_regex body, /(?P<ts_str>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2})/

// With milliseconds: 2026-03-28 10:14:32.456
extract_regex body, /(?P<ts_str>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d+)/
```

**Step 2 — parse to timestamp column** using the right function:

```
// ISO 8601 (handles Z, +hh:mm, fractional seconds automatically)
make_col timestamp: parse_isotime(ts_str)

// Non-ISO formats — use parse_timestamp with explicit format string
make_col timestamp: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")   // Apache CLF
make_col timestamp: parse_timestamp(ts_str, "MON DD HH24:MI:SS", "UTC")         // Syslog (no tz in log)
make_col timestamp: parse_timestamp(ts_str, "YYYY-MM-DD HH24:MI:SS.FF3")        // custom with millis, assumes UTC
make_col timestamp: parse_timestamp(ts_str, "DY, DD MON YYYY HH24:MI:SS TZHTZM") // RFC-2822

drop_col ts_str
```

### `parse_isotime()` vs `parse_timestamp()`

| | `parse_isotime(str)` | `parse_timestamp(str, format, [tz]?)` |
|---|---|---|
| **Use when** | Format is ISO 8601 (any variant) | Format is non-ISO (CLF, Syslog, custom) |
| **Format arg** | None — auto-detects | Required string constant |
| **Timezone** | Read from string or defaults to UTC | Optional 3rd arg (IANA name or offset) |
| **On mismatch** | Error | Returns NULL (safe) |
| **Inline cast** | `::parse_isotime` in `extract_regex` | Not available as inline cast |

### `parse_timestamp()` format specifiers (Snowflake standard)

| Specifier | Meaning | Example |
|---|---|---|
| `YYYY` | 4-digit year | `2026` |
| `YY` | 2-digit year | `26` |
| `MM` | Month number, zero-padded | `03` |
| `MON` | Abbreviated month name | `Mar` |
| `MMMM` | Full month name | `March` |
| `DD` | Day of month, zero-padded | `28` |
| `DY` | Abbreviated day name | `Sat` |
| `HH24` | Hour, 24-hour clock (00–23) | `14` |
| `HH12` / `HH` | Hour, 12-hour clock (01–12) | `02` |
| `AM` / `PM` | Meridiem indicator | `PM` |
| `MI` | Minute (00–59) | `22` |
| `SS` | Second (00–59) | `15` |
| `FF3` | Milliseconds (3 digits) | `059` |
| `FF6` | Microseconds (6 digits) | `059123` |
| `FF9` | Nanoseconds (9 digits) | `059123456` |
| `FF` | Nanoseconds (alias for FF9) | |
| `TZH:TZM` | Timezone offset with colon | `+07:00` |
| `TZHTZM` | Timezone offset no colon | `+0700` |
| `TZH` | Timezone hour only | `+07` |

Common log format strings:
```
Apache CLF:  "DD/MON/YYYY:HH24:MI:SS TZHTZM"    // 28/Mar/2026:10:22:15 +0000
Syslog:      "MON DD HH24:MI:SS"                  // Mar 28 10:22:15
ISO+ms+tz:   "YYYY-MM-DD HH24:MI:SS.FF3TZH:TZM"  // 2026-03-28 10:14:32.059+05:30
ISO+ms:      "YYYY-MM-DD HH24:MI:SS.FF3"          // 2026-03-28 10:14:32.059 (→ UTC)
RFC-2822:    "DY, DD MON YYYY HH24:MI:SS TZHTZM"  // Tue, 18 Oct 2022 13:41:26 -0700
```

### `get_regex()` function — for use inside `make_col`
**Signature:** `get_regex(input, /pattern/, [group: int64]?, [flags: string]?) -> string`

Returns the Nth capture group as a string, or null if no match. Group is 1-indexed (0 = full match).

**Convention:** After the initial `extract_regex` step, the captured message body should be named `extracted_message`. All subsequent `get_regex()` calls operate on `extracted_message`, not `body`.
```
make_col log_level:    get_regex(body, /\d{2}:\d{2}:\d{2}\s+(\w+)/, 1)
make_col service_name: get_regex(body, /\[(\w+)\]/, 1)
// Extract key=value pairs from extracted_message (not body):
make_col user_id:      int64(get_regex(extracted_message, /user_id=(\d+)/, 1))
make_col duration_ms:  int64(get_regex(extracted_message, /duration=(\d+)ms/, 1))
make_col http_status:  int64(get_regex(extracted_message, /status=(\d+)/, 1))
```
Use `get_regex()` when you need the result inside a `case()` or other expression. Use the `extract_regex` verb when creating standalone columns. The same `(?:...)` constraint applies — patterns must not contain non-capturing groups.

### Other regex functions
```
make_col matches_all: get_regex_all(body, /\b\d{3}\b/)    # returns array of all matches
make_col n_errors:    count_regex_matches(body, /error/i)  # count of matches
filter match_regex(body, /timeout|refused/)                 # bool test (also: body ~ /.../)
make_col clean:       replace_regex(body, /\d{4}-\d{2}-\d{2}/, "DATE")
```

---

## JSON / object field access

```
FIELDS["key"]                          # bracket notation
FIELDS["nested"]["key"]                # nested access
FIELDS.key                             # dot notation (equivalent)
string(FIELDS["host"]["name"])         # with type cast
float64(FIELDS["latency_ms"])          # numeric cast
```

### Object construction — `make_object()`
**Signature:** `make_object(key1: string, val1: storable, key2: string, val2: storable, ...) -> object`
```
make_col tags: make_object(
  "host",    string(FIELDS["host"]["name"]),
  "env",     string(FIELDS["fields"]["env"]),
  "service", string(FIELDS["service"]["type"])
)
```

### Object field manipulation
```
# Drop specific keys from an object
make_col clean: drop_fields(FIELDS, "@version", "message", "@timestamp")

# Select only specific keys
make_col dims: select_fields(FIELDS, "host", "app", "environment")

# Get a field by dynamic name
make_col val: get_field(obj, field_name_col)

# Get all keys of an object
make_col keys: object_keys(FIELDS)
```

### JSON parsing
```
make_col parsed: parse_json(raw_json_string)
make_col valid:  check_json(raw_json_string)     # returns bool
make_col kvp:    parse_kvp(body, "=", " ")       # parse "k1=v1 k2=v2" → object
make_col url:    parse_url(FIELDS["url"])         # parse URL → {scheme, host, path, ...}
make_col ua:     parse_user_agent(FIELDS["user_agent"])  # parse UA string → object
```

### Flattening nested objects/arrays
```
flatten_leaves FIELDS              # leaf values only — _c_FIELDS_path, _c_FIELDS_value
flatten_leaves FIELDS, true        # also produces _c_FIELDS_type (e.g. "float64", "string")
flatten_single FIELDS              # first level only
flatten       FIELDS               # all levels including intermediates (nulls for non-leaves)
flatten_all   FIELDS               # all levels including intermediates (no nulls)
```
> **Performance:** `flatten_leaves` is the most efficient — skips intermediate object nodes. All flatten variants are accelerable.

---

## Aggregation

### `statsby` — group and aggregate
```
statsby count:count(1), group_by(pod)
statsby total:count(1), errors:sum(is_error), group_by(service_name)
statsby avg_dur:avg(duration), p99:percentile(duration, 99), group_by(service_name)
statsby max_lat:max(latency_ms), group_by(host, env)
statsby unique_users:count_distinct(user_id), group_by(service_name)
statsby nicks:array_agg(nickname), group_by(uid)
```

### `timechart` — time-bucketed aggregation
```
timechart 1h, count:count(1)
timechart 5m, count:count(1), group_by(pod)
timechart 1m, avg_dur:avg(duration), p99:percentile(duration,99), group_by(service_name)
timechart options(bins:300), requests:count(), group_by(service)
```
Common intervals: `1m`, `5m`, `15m`, `30m`, `1h`, `6h`, `1d`.

### `timestats` — aggregate at every point in time
```
timestats count:count(), group_by(server_name)
```

### `aggregate` — aggregate metrics across dimensions
```
aggregate tx_bytes:sum(tx_bytes), group_by(podName, namespace)
```

### Aggregate functions
| Function | Signature | Description |
|---|---|---|
| `count` | `count([item]?) -> int64` | Count non-null items |
| `count_distinct` | `count_distinct(expr) -> int64` | Count distinct non-null items (approx) |
| `count_distinct_exact` | `count_distinct_exact(expr) -> int64` | Exact distinct count |
| `sum` | `sum(value) -> value` | Sum of group |
| `avg` | `avg(value) -> float64` | Average of group |
| `min` | `min(value) -> value` | Minimum of group |
| `max` | `max(value) -> value` | Maximum of group |
| `median` | `median(value) -> float64` | Median of group |
| `percentile` | `percentile(value, p: float64) -> numeric` | Pth percentile |
| `stddev` | `stddev(value) -> float64` | Standard deviation |
| `variance` | `variance(value) -> float64` | Variance |
| `first` | `first(expr) -> expr` | First value in group |
| `last` | `last(expr) -> expr` | Last value in group |
| `first_not_null` | `first_not_null(expr) -> expr` | First non-null value |
| `last_not_null` | `last_not_null(expr) -> expr` | Last non-null value |
| `any` | `any(expr) -> expr` | Any value in group |
| `any_not_null` | `any_not_null(expr) -> expr` | Any non-null value |
| `array_agg` | `array_agg(expr, [order_by(...)]?) -> array` | Collect values into array |
| `array_agg_distinct` | `array_agg_distinct(expr) -> array` | Collect distinct values into array |
| `array_union_agg` | `array_union_agg(arr) -> array` | Union of arrays across group |

---

## Sorting and limiting

```
sort asc(pod_name), desc(cluster_id)        # sort ascending/descending
limit 100                                    # first N rows
topk 10                                      # top 10 groups (by count or custom score)
topk 10, score:sum(errors), group_by(service)
bottomk 5                                    # bottom 5 groups
dedup vf, message                            # collapse identical rows
```

---

## Combining datasets

### `union` — stack rows from multiple datasets
```
union @dataset2, @dataset3

# Inline union to create one row per metric (unpivot pattern):
union
  (make_col metric: "latency_ms",    value: float64(FIELDS["latency_ms"])),
  (make_col metric: "response_code", value: float64(FIELDS["response_code"])),
  (make_col metric: "is_error",      value: float64(bool(FIELDS["error"])))
```

### `join` / `leftjoin` / `fulljoin` — temporal joins
```
join on(host_uid=@hosts.uid), hostname:@hosts.name, region:@hosts.region
leftjoin on(pod_uid=@pods.uid), pod_name:@pods.name    # left rows always included
fulljoin on(host_uid=@hosts.uid), hostname:@hosts.name  # all rows from both sides
```

### `lookup` — enrich with reference table
```
lookup on(service_name=@ref.name), tier:@ref.tier, owner:@ref.team_email
```

### `follow` / `follow_not` / `exists` / `not_exists` — existence joins
```
follow sensor_id=@right.sensor_id               # keep rows with a match in window
follow_not sensor_id=@right.sensor_id           # keep rows WITHOUT a match
exists sensor_id=@right.sensor_id               # like follow
not_exists sensor_id=@right.sensor_id           # like follow_not
```

### `surrounding` — outer join by time frame
```
filter body ~ /panic/ | surrounding frame(back:2s, ahead:2s), @logs
```

---

## Data reshaping

### `unpivot` — columns → rows
```
unpivot name_column:"metric", value_column:"value", cpu_pct, mem_pct, disk_pct
```

### `pivot` — rows → columns
```
pivot name:product, value:sales, laptop:sales(product="laptop"), tablet:sales(product="tablet")
```

### `dedup` / `distinct` — remove duplicate rows
```
dedup timestamp, message
```

---

## Time operations

### Setting dataset timestamps
```
# Event-shaped data (Logstash, logs, spans):
make_col ts: timestamp_s(string(FIELDS["@timestamp"]))
set_timestamp options(max_time_diff:4h), ts
drop_col ts

# Resource/Interval data:
set_valid_from options(max_time_diff:4h), start_time
set_valid_to end_time
```

### Timestamp conversion functions
| Function | Description |
|---|---|
| `timestamp_s(int64)` | Seconds since epoch → timestamp |
| `timestamp_ms(int64)` | Milliseconds since epoch → timestamp |
| `timestamp_us(int64)` | Microseconds since epoch → timestamp |
| `timestamp_ns(int64)` | Nanoseconds since epoch → timestamp |
| `parse_isotime(string)` | ISO 8601 string → timestamp |
| `format_time(ts, format)` | Format timestamp as string |
| `to_seconds(ts)` | Timestamp → seconds since epoch (int64) |
| `to_milliseconds(ts)` | Timestamp → milliseconds since epoch |
| `to_nanoseconds(ts)` | Timestamp → nanoseconds since epoch |

### Duration functions
```
make_col dur_ms: duration_ms(duration_col)    # duration → float64 milliseconds
make_col dur_s:  duration_sec(duration_col)   # duration → float64 seconds
make_col dur_hr: duration_hr(duration_col)    # duration → float64 hours
```

### Time manipulation
```
timeshift 1h               # shift all timestamps forward by 1h
timewrap 1d, 4, "label"    # create 4 parallel time series (for week-over-week comparison)
```

---

## Type conversion functions

```
string(value)              # → string
int64(value)               # → int64
float64(value)             # → float64
bool(value)                # → bool
duration(value)            # → duration
object(value)              # → object (or null)
array(value)               # → array (or null)
```

Null-typed constructors (for columns that must exist but may be null):
```
string_null()   float64_null()   int64_null()   bool_null()
duration_null() timestamp_null() object_null?() array_null?()
```

---

## Conditional functions

```
case(cond1, val1, cond2, val2, ..., true, default)
if(condition, true_val, false_val)
if_null(value, fallback)              # return fallback if value is null
coalesce(a, b, c)                     # first non-null argument
nullif(val1, val2)                    # return null if val1 = val2, else val1
isnull(value)                         # bool: is value null?
```

Examples:
```
make_col tier: case(
  loyalty_level = "gold",   "premium",
  loyalty_level = "silver", "standard",
  true, "basic"
)
make_col safe_latency: if_null(latency_ms, 0.0)
make_col svc: coalesce(service_name, attributes["service.name"], "unknown")
```

---

## String functions

| Function | Signature | Description |
|---|---|---|
| `upper` | `upper(s) -> string` | Uppercase |
| `lower` | `lower(s) -> string` | Lowercase |
| `trim` | `trim(s) -> string` | Strip leading/trailing whitespace |
| `ltrim` | `ltrim(s) -> string` | Strip leading whitespace |
| `rtrim` | `rtrim(s) -> string` | Strip trailing whitespace |
| `strlen` | `strlen(s) -> int64` | String length |
| `substring` | `substring(s, start, [len]?) -> string` | Substring (1-indexed) |
| `strpos` | `strpos(haystack, needle) -> int64` | Position of needle (1-indexed, 0=not found) |
| `contains` | `contains(s, substr) -> bool` | True if substr present |
| `starts_with` | `starts_with(s, prefix) -> bool` | True if starts with prefix |
| `ends_with` | `ends_with(s, suffix) -> bool` | True if ends with suffix |
| `concat_strings` | `concat_strings(s1, s2, ...) -> string` | Concatenate strings |
| `replace_string` | `replace_string(s, old, new) -> string` | Replace literal substring |
| `replace_regex` | `replace_regex(s, /pat/, repl) -> string` | Replace regex match |
| `split_string` | `split_string(s, delim) -> array` | Split by delimiter |
| `lpad` | `lpad(s, len, [pad]?) -> string` | Left-pad to length |
| `rpad` | `rpad(s, len, [pad]?) -> string` | Right-pad to length |
| `encode_base64` | `encode_base64(s) -> string` | Base64 encode |
| `decode_base64` | `decode_base64(s) -> string` | Base64 decode |
| `encode_uri` | `encode_uri(s) -> string` | URI encode |
| `decode_uri` | `decode_uri(s) -> string` | URI decode |
| `editdistance` | `editdistance(s1, s2) -> int64` | Levenshtein distance |

---

## Numeric functions

```
abs(x)           # absolute value
floor(x)         # round down
ceil(x)          # round up
pow(base, exp)   # exponentiation
sqrt(x)          # square root
exp(x)           # e^x
ln(x)            # natural log
log10(x)         # log base 10
log2(x)          # log base 2
mod(x, y)        # modulo
degrees(rad)     # radians → degrees
radians(deg)     # degrees → radians
```

Metric/time-series specific:
```
delta(value)           # difference from previous row
delta_monotonic(value) # monotonic counter delta (handles resets)
deriv(value)           # rate of change per second
ewma(value, alpha)     # exponential weighted moving average
```

---

## Array functions

| Function | Description |
|---|---|
| `array_length(arr)` | Length of array |
| `array_contains(arr, val)` | True if array contains val |
| `array_distinct(arr)` | Remove duplicates from array |
| `array_max(arr)` | Maximum element |
| `array_min(arr)` | Minimum element |
| `array_to_string(arr, delim)` | Join array into string |
| `append_item(arr, val)` | Append element to array |
| `concat_arrays(arr1, arr2)` | Concatenate two arrays |
| `arrays_overlap(arr1, arr2)` | True if arrays share any element |
| `array_pivot(arr, key, val)` | Pivot array of objects into object |
| `array_unpivot(arr)` | Unpivot object into array of key-value pairs |
| `zip_arrays(arr1, arr2)` | Zip two arrays into array of pairs |
| `get_item(arr, index)` | Get element at index (0-based) |
| `json_array(v1, v2, ...)` | Construct a JSON array literal |

---

## Window functions

Window functions operate over ordered groups. Wrap in `window(fn, group_by(col), order_by(col))`:

```
make_col prev_val:   window(lag(value, 1),       group_by(host), order_by(timestamp))
make_col next_val:   window(lead(value, 1),      group_by(host), order_by(timestamp))
make_col row_num:    window(row_number(),         group_by(service))
make_col rank_val:   window(rank(),               group_by(service), order_by(desc(error_rate)))
make_col dense_rank: window(dense_rank(),         group_by(service), order_by(desc(count)))
make_col cum_sum:    window(sum(value),           group_by(host), order_by(timestamp))
make_col pct:        window(ntile(4),             group_by(service))
make_col run_avg:    window(avg(value),           group_by(host), order_by(timestamp))
make_col nth:        window(nth_value(value, 3),  group_by(host))
```

---

## Metadata and interface verbs

### Shape declarations
```
make_event                                # → Event dataset
make_resource options(expiry:1h)          # → Resource dataset
make_interval validFrom:start, validTo:end  # → Interval dataset
make_table                                # → Table dataset
```

### Keys and links
```
set_primary_key service_name, region      # declare primary key (alias: set_pk)
add_key cluster_uid, resource_uid         # add candidate key
set_label device_name                     # column to use as display label
set_link "Hosts", host_uid:@hosts.uid     # outbound foreign key
unset_link "Hosts"                        # remove a link
unset_all_links                           # remove all links
```

### Interface declaration
```
interface "metric"                              # if columns already named metric, value
interface "metric", metric:name_col, value:val_col   # explicit binding
interface "log"
```

### Column metadata
```
set_col_visible col_name:false            # hide/show in UI
set_col_searchable col_name:false         # include/exclude from full-text search
set_col_immutable col_name:true           # mark as time-invariant on a Resource
set_col_enum col_name:true               # mark as enum for UI presentation
```

---

## Full pipeline examples

### Error rate by service (spans)
```
make_col is_error: case(error = true, 1, true, 0)
| statsby total:count(1), errors:sum(is_error), group_by(service_name)
| make_col error_rate: errors / total
```

### Parse CLF access log and count 401s by IP
```
extract_regex body, /(?P<client_ip>[\d.]+) \S+ \S+ \[[^\]]+\] "[^"]+" (?P<status::int64>\d+)/
| filter status = 401
| statsby attempts:count(1), group_by(client_ip)
| sort desc(attempts)
```

### Enrich with a reference table then timechart
```
lookup on(service_name=@services.name), tier:@services.tier, owner:@services.owner
| filter tier = "critical"
| timechart 5m, error_count:count(1), group_by(service_name)
```

### Extract and chart a JSON field
```
make_col latency: float64(attributes["http.response.latency_ms"])
| filter not isnull(latency)
| timechart 1m, p99:percentile(latency, 99), avg:avg(latency), group_by(service_name)
```

### Logstash: build tags/metrics objects and flatten
```
pick_col FIELDS
make_col _ts: string(FIELDS["@timestamp"])
make_col tags: make_object(
  "host", string(FIELDS["host"]["name"]),
  "env",  string(FIELDS["fields"]["env"])
)
make_col metrics: FIELDS["system"]["cpu"]
drop_col FIELDS
flatten_leaves metrics, true
filter _c_metrics_type in ("float64", "int64")
rename_col metric: _c_metrics_path
make_col value: float64(_c_metrics_value)
drop_col _c_metrics_type
make_col timestamp: parse_isotime(_ts)
set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp, _ts
interface "metric"
```
