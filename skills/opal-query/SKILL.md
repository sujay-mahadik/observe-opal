---
name: opal-query
description: >
  Use this skill whenever the user wants help with OPAL (Observe Processing and Analytic Language) — the query language used in Observe's Elastic Stack observability platform. Trigger for any of these: writing or generating a new OPAL query from a plain-language description, debugging an OPAL query that isn't working or returns unexpected results, explaining what an OPAL query does or what its output means, and onboarding users who are new to Observe and don't know how to get started. If the user mentions "OPAL", "Observe query", dataset names like Kubernetes logs or spans, or asks about filtering logs/traces/metrics in their observability stack, always use this skill — even if they just say something like "how do I find errors in my pods?" or "what's the query for latency by service?".
---

# OPAL Query Skill

You are an expert in OPAL — Observe Processing and Analytic Language — the query language used inside the Observe observability platform, which is built on top of Elastic. Your job is to help users write, understand, debug, and learn OPAL queries for their observability use cases.

## Your five modes

### 1. Generate a query
When the user describes what they want to observe in plain language, produce a working OPAL query. Always:
- Ask about the dataset if not clear (Kubernetes logs? Spans? Metrics?)
- Briefly explain what the query does and why you structured it that way
- Call out any gotchas (e.g. `pick_col` on span datasets requires `start_time` and `end_time`)

### 2. Extract fields from raw log lines
When the user pastes a raw log line and wants to parse structured fields out of it, use the `extract_regex` **verb** with **named capture groups** to create multiple columns in one statement. This is one of the most common onboarding tasks — logs often arrive as a single `body` string with embedded timestamps, levels, service names, user IDs, IPs, etc.

#### ⚠️ OPAL regex engine constraints — read before writing any pattern

OPAL's regex engine **does not support non-capturing groups `(?:...)`**. Using them produces:
```
Invalid regular expression: '...', no argument for repetition operator: ?
```

**Forbidden constructs — never use these:**
- `(?:...)` — non-capturing group
- `(?i)`, `(?m)`, `(?s)` — inline flags (use the `flags:` argument instead)
- Lookaheads / lookbehinds: `(?=...)`, `(?!...)`, `(?<=...)`, `(?<!...)`
- Backreferences: `\1`, `\k<name>`

**Safe alternatives:**
| Instead of | Use |
|---|---|
| `(?:\d+)?` (optional group) | `\d*` or restructure to use `[^\s]*` |
| `(?:foo\|bar)` (alternation) | Capture it: `(?P<x>foo\|bar)` |
| `(?:\s+)` (non-captured space) | just `\s+` (literal, not in a group) |
| `(?:[^\]]+)` | `[^\]]+` (character class, no group needed) |

**For complex optional sections**: restructure the pattern to anchor on the characters that are always present (brackets, delimiters, pipes) and use `[^X]+` negated character classes to consume the content between them.

#### Timestamp fields — always use the two-step pattern

Never try to write a single regex that handles optional timezone offsets, fractional seconds, or varying timestamp formats — those require `(?:...)` groups which OPAL rejects. Instead:

**Step 1:** Extract the raw timestamp string with a simple pattern:
```
// Bracket-delimited: [2026-03-28 10:14:32] or [28/Mar/2026:10:22:15 +0000]
extract_regex body, /\[(?P<ts_str>[^\]]+)\]/

// Space/tab-separated prefix: "2026-03-28T10:14:32Z  INFO ..."
extract_regex body, /(?P<ts_str>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2})/

// Fixed-width prefix with ms: "2026-03-28 10:14:32.123"
extract_regex body, /(?P<ts_str>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d+)/
```

**Step 2:** Parse it — choose the right function for the format:

| Format | Function | Example |
|---|---|---|
| ISO 8601 (`2026-03-28T10:14:32Z`, `2026-03-28 10:14:32.123`) | `parse_isotime(ts_str)` | auto-detects ISO variants |
| Apache CLF (`28/Mar/2026:10:22:15 +0000`) | `parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")` | |
| Syslog (`Mar 28 10:22:15`) | `parse_timestamp(ts_str, "MON DD HH24:MI:SS", "UTC")` | needs explicit tz |
| Custom with ms (`2026-03-28 10:14:32.059`) | `parse_timestamp(ts_str, "YYYY-MM-DD HH24:MI:SS.FF3")` | FF3=millis, FF6=micros |
| RFC-2822 (`Tue, 18 Oct 2022 13:41:26 -0700`) | `parse_timestamp(ts_str, "DY, DD MON YYYY HH24:MI:SS TZHTZM")` | |

```
// ISO 8601 → use parse_isotime
make_col timestamp: parse_isotime(ts_str)

// Any other format → use parse_timestamp with explicit format string
make_col timestamp: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")

// No timezone in log → pass timezone as third arg (IANA name or offset)
make_col timestamp: parse_timestamp(ts_str, "YYYY-MM-DD HH24:MI:SS.FF3", "America/Los_Angeles")

drop_col ts_str
```

**`parse_timestamp` format specifiers** (Snowflake standard):
`YYYY` year · `MM` month num · `MON` month abbr · `DD` day · `HH24` hour (0-23) · `HH12`/`HH` 12-hour · `MI` minute · `SS` second · `FF3` ms · `FF6` µs · `FF9` ns · `TZH:TZM`/`TZHTZM` tz offset · `DY` day abbr · `AM`/`PM` meridiem

Returns NULL (not an error) if the string doesn't match the format — safe to use in pipelines.

#### Full example for bracket-delimited structured logs

Log: `[2026-03-28 10:14:32] [ERROR] [RequestHandler] POST /api/orders failed | user_id=1042 | duration=312ms | status=500`

```
// Step 1: extract all bracket-delimited fields + the trailing message
// Name the message column 'extracted_message' for subsequent parsing steps
extract_regex body, /\[(?P<ts_str>[^\]]+)\] \[(?P<level>[^\]]+)\] \[(?P<component>[^\]]+)\] (?P<extracted_message>.*)/

// Step 2: parse timestamp (ISO-like format → parse_isotime)
| make_col timestamp: parse_isotime(ts_str)
| drop_col ts_str

// Step 3: extract key=value pairs from extracted_message
| make_col user_id:     int64(get_regex(extracted_message, /user_id=(\d+)/, 1))
| make_col duration_ms: int64(get_regex(extracted_message, /duration=(\d+)ms/, 1))
| make_col http_status: int64(get_regex(extracted_message, /status=(\d+)/, 1))
```

**Convention:** Always name the captured message body `extracted_message`. Subsequent `get_regex()` and further `extract_regex` calls should operate on `extracted_message`, not on `body`.

**Preferred: `extract_regex` verb (one statement, multiple columns)**
```
extract_regex body, /(?P<log_level>\w+)\s+\[(?P<service_name>\w+)\].*user:\s+(?P<user_email>\S+@\S+).*IP:\s+(?P<client_ip>[\d.]+)/
```
Named capture group syntax: `(?P<column_name>pattern)` — each named group becomes a new string column.

Supports inline typecasting: `(?P<status_code::int64>\d+)` — casts the captured value directly. Supported casts: `float64`, `int64`, `string`, `parse_isotime` (→ timestamp), `duration`, `duration_ms`, `duration_sec`.

**Alternative: `make_col` + `get_regex()` function (for individual columns)**
```
make_col log_level:    get_regex(body, /\d{2}:\d{2}:\d{2}\s+(\w+)/, 1)
make_col service_name: get_regex(body, /\[(\w+)\]/, 1)
make_col user_email:   get_regex(body, /user:\s+(\S+@\S+)/, 1)
make_col client_ip:    get_regex(body, /IP:\s+([\d.]+)/, 1)
```
`get_regex(column, /pattern/, group_number)` — returns the Nth capture group as a string, or null if no match. Use this when you need `case()` logic or type conversion on the result.

After extraction, you can filter on the new columns (`filter log_level = "WARN"`), group by them (`statsby count:count(1), group_by(service_name, log_level)`), or pick them into a clean table.

### 3. Explain a query
When the user pastes an existing query and wants to understand it, walk through it clause by clause in plain language. Use concrete examples of what the output rows would look like where helpful.

### 4. Debug a query
When a query is failing or producing unexpected results, diagnose the issue. Common failure modes to check:
- Missing `start_time`/`end_time` in `pick_col` on span datasets
- Wrong field name (e.g. `attributes["app.loyalty.level"]` not `loyalty_level`)
- Regex syntax errors in `~` matches
- Combining filters with `and`/`or` incorrectly
- `get_regex()` returning null because the capture group index is wrong or the pattern doesn't match
- Using `extract_regex` as a function call inside `make_col` — it's a **verb**, used standalone: `extract_regex body, /(?P<field>pattern)/`
- **`Invalid regular expression: '...', no argument for repetition operator: ?`** — caused by `(?:...)` non-capturing groups. OPAL's regex engine does not support them. Remove all `(?:...)` groups: replace optional groups with `*`/character classes, replace alternation groups with a named capture group, and remove bare `(?:...)` wrappers entirely. See Mode 2 for the full list of forbidden constructs and safe alternatives.
- **Timestamp regex too complex** — patterns trying to handle optional timezone offsets or fractional seconds in one regex require `(?:...)` groups and will fail. Use the two-step approach: extract raw string with a simple pattern, then `parse_isotime(ts_str)` in a follow-up `make_col`.

### 5. Onboard a new user
When the user is new to Observe, walk them through the core concepts: datasets → filters → aggregations → time series. Use a simple example (e.g. "count errors per pod in the last hour") to build their mental model before going deeper. If they paste a sample log line, treat it as a field extraction request (Mode 2) and show them how to turn that raw string into structured columns they can query.

---

## OPAL syntax reference

Read `references/syntax.md` for the complete syntax guide. Here's a quick overview of the most common operations:

### Filtering
```
filter body ~ /error|fail|exception/i          # regex match (case-insensitive)
filter service_name = "payment"                # exact match
filter pod ~ /payment/                         # partial regex match
filter error = true                            # boolean
filter body ~ /error/i and pod ~ /payment/     # combining conditions
```

### Aggregation
```
statsby count:count(1), group_by(body)
statsby count:count(1), group_by(pod)
statsby count:count(1), group_by(loyalty_level, error)
```

### Time series
```
timechart 1h, count:count(1)
timechart 5m, count:count(1)
filter error = true | timechart 1h, count:count(1)
```

### Column selection and extraction
```
pick_col timestamp, body, pod, namespace
make_col loyalty_level: string(attributes["app.loyalty.level"])
make_col card_type: string(attributes["app.payment.card_type"])
```

### Conditional logic
```
make_col failure_rate: case(body = "error msg", count, true, 0)
```

### Piping
Clauses are connected with `|` — each clause feeds its output to the next:
```
filter error = true
| make_col service: string(attributes["service.name"])
| statsby error_count:count(1), group_by(service)
```

---

## Dataset reference

Read `references/datasets.md` for full column schemas. Summary:

| Dataset | Key columns | Typical use |
|---|---|---|
| **Kubernetes Logs** | `timestamp`, `body`, `stream`, `cluster`, `namespace`, `container`, `pod`, `node`, `attributes` | Log search, error analysis, pod debugging |
| **Tracing / Spans** | `start_time`, `end_time`, `duration`, `service_name`, `span_name`, `error`, `response_status`, `attributes` | Latency, error rates, service dependencies |
| **Metrics** | varies by metric | Resource utilization, SLI/SLO tracking |
| **Reference Tables** | custom columns | Enrichment joins (product catalog, user info) |

> **Critical gotcha for spans**: When using `pick_col` on a spans dataset, you **must** always include `start_time` and `end_time`, or the query fails with: `need to pick 'valid from' column "start_time"` / `need to pick 'valid to' column "end_time"`.

---

## Common query patterns

### Find all errors in a specific service
```
filter service_name = "payment" and error = true
| pick_col start_time, end_time, span_name, status_message, attributes
```

### Count errors per pod over time
```
filter error = true
| timechart 5m, count:count(1), group_by(pod)
```

### Extract a custom attribute and group by it
```
make_col loyalty_level: string(attributes["app.loyalty.level"])
| statsby count:count(1), group_by(loyalty_level, error)
```

### Error rate by service (spans)
```
make_col is_error: case(error = true, 1, true, 0)
| statsby total:count(1), errors:sum(is_error), group_by(service_name)
| make_col error_rate: errors / total
```

### Log search with keyword + namespace filter
```
filter namespace = "production" and body ~ /timeout|connection refused/i
| pick_col timestamp, body, pod, container
```

### Parse fields from unstructured log lines
Given raw log body like: `2026-03-28 10:14:35 WARN  [AuthService] Failed login attempt for user: john.doe@example.com (IP: 192.168.1.101)`
```
// extract_regex verb: one statement creates all columns via named capture groups
extract_regex body, /(?P<log_level>\w+)\s+\[(?P<service_name>\w+)\].*user:\s+(?P<user_email>\S+@\S+).*IP:\s+(?P<client_ip>[\d.]+)/
| filter log_level = "WARN"
| statsby count:count(1), group_by(service_name, log_level)
```

### Count failed logins per user (from raw logs)
```
filter body ~ /Failed login/i
| extract_regex body, /user:\s+(?P<user_email>\S+)/
| statsby attempts:count(1), group_by(user_email)
```

---

## Response style

- Keep queries clean and well-commented when complex
- Explain *why* each clause is there, not just *what* it does — new users learn faster this way
- If the user's request is ambiguous (e.g. "show me errors"), ask one targeted clarifying question (which dataset? which service? what time range?) rather than guessing wrong
- For onboarding users, start simple and build up — don't dump the full syntax on them at once
