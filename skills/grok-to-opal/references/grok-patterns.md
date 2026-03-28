# Grok → OPAL Pattern Reference

Complete mapping of every ECS-v1 Grok pattern to its OPAL `extract_regex` equivalent.

---

## Core pattern mapping table

For each `%{PATTERN_NAME:field_name}` token, replace with `(?P<field_name>OPAL_REGEX)`.
For `%{PATTERN_NAME}` with no field (no capture), inline the OPAL_REGEX directly.

### Generic matchers

| Grok pattern | OPAL regex | Notes |
|---|---|---|
| `WORD` | `\w+` | Word chars only |
| `NOTSPACE` | `\S+` | Anything non-whitespace |
| `DATA` | `.*?` | Non-greedy any chars |
| `GREEDYDATA` | `.*` | Greedy any chars — only use at end of pattern |
| `SPACE` | `\s*` | Zero or more spaces |
| `QUOTEDSTRING` / `QS` | `"(?:[^"\\]|\\.)*"` | Double-quoted string, backslash-escaped |

### Numbers

| Grok pattern | OPAL regex | Recommended type |
|---|---|---|
| `INT` | `[+-]?[0-9]+` | `::int64` |
| `POSINT` | `[1-9][0-9]*` | `::int64` |
| `NONNEGINT` | `[0-9]+` | `::int64` |
| `NUMBER` / `BASE10NUM` | `[+-]?(?:[0-9]+(?:\.[0-9]+)?|\.[0-9]+)` | `::float64` |
| `BASE16NUM` | `[+-]?(?:0x)?[0-9A-Fa-f]+` | string |
| `BASE16FLOAT` | `[+-]?(?:0x)?[0-9A-Fa-f]+(?:\.[0-9A-Fa-f]*)?` | `::float64` |

### Identifiers and users

| Grok pattern | OPAL regex |
|---|---|
| `USERNAME` / `USER` | `[a-zA-Z0-9._-]+` |
| `EMAILLOCALPART` | `[a-zA-Z0-9!#$%&'*+\-/=?^_{|}~]+` |
| `EMAILADDRESS` | `[a-zA-Z0-9!#$%&'*+\-/=?^_{|}~]+@[0-9A-Za-z][0-9A-Za-z.-]*` |
| `UUID` | `[A-Fa-f0-9]{8}-(?:[A-Fa-f0-9]{4}-){3}[A-Fa-f0-9]{12}` |

### Networking

| Grok pattern | OPAL regex | Notes |
|---|---|---|
| `IPV4` | `(?:\d{1,3}\.){3}\d{1,3}` | Simplified; full version validates ranges |
| `IPV6` | `[0-9A-Fa-f:]+(?:%[^\s]+)?` | Simplified; full is 800+ chars |
| `IP` | `(?:[0-9A-Fa-f:]+(?:%[^\s]+)?|(?:\d{1,3}\.){3}\d{1,3})` | v4 or v6 |
| `HOSTNAME` | `[0-9A-Za-z][0-9A-Za-z\-]*(?:\.[0-9A-Za-z][0-9A-Za-z\-]*)*` | |
| `IPORHOST` | `\S+` | IP or hostname — `\S+` is simpler and sufficient |
| `HOSTPORT` | `\S+:\d+` | host:port combo |
| `CISCOMAC` | `[A-Fa-f0-9]{4}\.[A-Fa-f0-9]{4}\.[A-Fa-f0-9]{4}` | |
| `COMMONMAC` | `(?:[A-Fa-f0-9]{2}:){5}[A-Fa-f0-9]{2}` | |
| `MAC` | `(?:[A-Fa-f0-9]{4}\.){2}[A-Fa-f0-9]{4}|(?:[A-Fa-f0-9]{2}[:\-]){5}[A-Fa-f0-9]{2}` | |

### Paths and URIs

| Grok pattern | OPAL regex |
|---|---|
| `UNIXPATH` | `/\S*` |
| `WINPATH` | `(?:[A-Za-z]+:|\\\\)\S+` |
| `PATH` | `/\S*` |
| `URIPROTO` | `[A-Za-z][A-Za-z0-9+\-.]+` |
| `URIHOST` | `[^\s/?#]+` |
| `URIPATH` | `/[^\s?#]*` |
| `URIQUERY` | `[^\s#]*` |
| `URI` | `[A-Za-z][A-Za-z0-9+\-.]+://\S+` |
| `URN` | `urn:[0-9A-Za-z][0-9A-Za-z\-]{0,31}:\S+` |

### Time and dates

| Grok pattern | OPAL regex | Notes |
|---|---|---|
| `HOUR` | `(?:2[0-3]|[01]?\d)` | |
| `MINUTE` | `[0-5]\d` | |
| `SECOND` | `(?:[0-5]?\d|60)(?:[.,]\d+)?` | Includes leap seconds |
| `TIME` | `(?:2[0-3]|[01]?\d):[0-5]\d:[0-5]\d(?:[.,]\d+)?` | HH:MM:SS |
| `YEAR` | `\d{2,4}` | |
| `MONTHNUM` | `(?:0?[1-9]|1[0-2])` | |
| `MONTHNUM2` | `(?:0[1-9]|1[0-2])` | Zero-padded |
| `MONTHDAY` | `(?:0[1-9]|[12]\d|3[01]|\d)` | |
| `MONTH` | `(?:Jan(?:uary)?|Feb(?:ruary)?|Mar(?:ch)?|Apr(?:il)?|May|Jun(?:e)?|Jul(?:y)?|Aug(?:ust)?|Sep(?:tember)?|Oct(?:ober)?|Nov(?:ember)?|Dec(?:ember)?)` | |
| `DAY` | `(?:Mon(?:day)?|Tue(?:sday)?|Wed(?:nesday)?|Thu(?:rsday)?|Fri(?:day)?|Sat(?:urday)?|Sun(?:day)?)` | |
| `TZ` | `(?:[APMCE][SD]T|UTC)` | |
| `ISO8601_TIMEZONE` | `(?:Z|[+-](?:2[0-3]|[01]?\d)(?::?[0-5]\d)?)` | |
| `TIMESTAMP_ISO8601` | `\d{4}-\d{2}-\d{2}[T ]\d{2}:?\d{2}(?::?\d{2}(?:[.,]\d+)?)?(?:Z|[+-]\d{2}:?\d{2})?` | ISO 8601 |
| `HTTPDATE` | `\d{2}/\w{3}/\d{4}:\d{2}:\d{2}:\d{2} [+-]\d{4}` | Apache CLF date |
| `SYSLOGTIMESTAMP` | `\w{3} +\d{1,2} \d{2}:\d{2}:\d{2}` | Syslog MMM DD HH:MM:SS |
| `DATE_US` | `\d{1,2}[/\-]\d{1,2}[/\-]\d{2,4}` | MM/DD/YYYY |
| `DATE_EU` | `\d{1,2}[.\-/]\d{1,2}[.\-/]\d{2,4}` | DD.MM.YYYY |
| `DATESTAMP` | `\S+ \d{2}:\d{2}:\d{2}(?:[.,]\d+)?` | Date + time |
| `DATESTAMP_RFC822` | `\w+ \w+ \d{1,2} \d{4} \d{2}:\d{2}:\d{2} \S+` | |
| `DATESTAMP_RFC2822` | `\w+, \d{1,2} \w+ \d{4} \d{2}:\d{2}:\d{2} \S+` | |
| `HTTPDERROR_DATE` | `\w+ \w+ \d{1,2} \d{2}:\d{2}:\d{2}(?:\.\d+)? \d{4}` | Apache error log date |

### Log-specific

| Grok pattern | OPAL regex | Notes |
|---|---|---|
| `LOGLEVEL` | `(?:[Aa]lert|ALERT|[Tt]race|TRACE|[Dd]ebug|DEBUG|[Nn]otice|NOTICE|[Ii]nfo(?:rmation)?|INFO(?:RMATION)?|[Ww]arn(?:ing)?|WARN(?:ING)?|[Ee]rr(?:or)?|ERR(?:OR)?|[Cc]rit(?:ical)?|CRIT(?:ICAL)?|[Ff]atal|FATAL|[Ss]evere|SEVERE|EMERG(?:ENCY)?)` | Case-insensitive |
| `PROG` | `[\x21-\x5a\x5c\x5e-\x7e]+` | Process name |
| `SYSLOGPROG` | `(?P<process_name>[^\[:\s]+)(?:\[(?P<pid::int64>\d+)\])?` | Expands to two fields |
| `SYSLOGHOST` | `\S+` | |
| `SYSLOGFACILITY` | `<(?P<syslog_facility_code::int64>\d+)\.(?P<syslog_priority::int64>\d+)>` | Expands to two fields |

### Java

| Grok pattern | OPAL regex |
|---|---|
| `JAVACLASS` | `(?:[a-zA-Z$_][a-zA-Z$_0-9]*\.)*[a-zA-Z$_][a-zA-Z$_0-9]*` |
| `JAVAFILE` | `[a-zA-Z$_0-9. -]+` |
| `JAVAMETHOD` | `(?:<(?:cl)?init>|[a-zA-Z$_][a-zA-Z$_0-9]*)` |
| `JAVALOGMESSAGE` | `.*` |
| `JAVATHREAD` | `[A-Z]{2}-Processor\d+` |

---

## Pre-built OPAL translations

Complete `extract_regex` statements for common log formats. Use these directly when the user gives you a top-level composite pattern name.

---

### Apache / Nginx — Common Log Format (`HTTPD_COMMONLOG`)

Grok: `%{HTTPD_COMMONLOG}`

```
// Fields: client_ip, username, ts_str, http_method, url, http_version, status_code, bytes_sent
extract_regex body, /(?P<client_ip>\S+) \S+ (?P<username>\S+) \[(?P<ts_str>[^\]]+)\] "(?P<http_method>\w+) (?P<url>\S+) HTTP\/(?P<http_version>[^"]+)" (?P<status_code::int64>-|\d+) (?P<bytes_sent::int64>-|\d+)/

// HTTPDATE → parse_timestamp (not parse_isotime)
| make_col _ts: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### Apache / Nginx — Combined Log Format (`HTTPD_COMBINEDLOG`)

Grok: `%{HTTPD_COMBINEDLOG}`

```
// Fields: client_ip, username, ts_str, http_method, url, http_version,
//         status_code, bytes_sent, referrer, user_agent
extract_regex body, /(?P<client_ip>\S+) \S+ (?P<username>\S+) \[(?P<ts_str>[^\]]+)\] "(?P<http_method>\w+) (?P<url>\S+) HTTP\/(?P<http_version>[^"]+)" (?P<status_code::int64>-|\d+) (?P<bytes_sent::int64>-|\d+) "(?P<referrer>[^"]*)" "(?P<user_agent>[^"]*)"/

| make_col _ts: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### Apache Error Log 2.4 (`HTTPD24_ERRORLOG`)

```
// Fields: ts_str, log_module, log_level, pid, client_ip, extracted_message
// Apache error date: "Sat Mar 28 10:14:32.059 2026"
extract_regex body, /\[(?P<ts_str>[^\]]+)\] \[(?P<log_module>\w+):(?P<log_level>\w+)\] \[pid (?P<pid::int64>\d+)\] (?P<extracted_message>.*)$/

| make_col _ts: parse_timestamp(ts_str, "DY MON DD HH24:MI:SS.FF3 YYYY")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### Syslog (`SYSLOGLINE`)

Grok: `%{SYSLOGLINE}`

```
// Fields: ts_str, hostname, process_name, pid, extracted_message
// Syslog timestamp has no year and no timezone — supply UTC as default
extract_regex body, /(?P<ts_str>\w{3} +\d{1,2} \d{2}:\d{2}:\d{2}) (?P<hostname>\S+) (?P<process_name>[^\[:\s]+)(?:\[(?P<pid::int64>\d+)\])?: (?P<extracted_message>.*)$/

| make_col _ts: parse_timestamp(ts_str, "MON DD HH24:MI:SS", "UTC")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### Syslog RFC 5424 (`SYSLOG5424LINE`)

```
// Fields: syslog_priority, syslog_version, ts_str, hostname, process_name, pid, extracted_message
// RFC 5424 timestamp is ISO 8601 → parse_isotime
extract_regex body, /<(?P<syslog_priority::int64>\d+)>(?P<syslog_version>\d+) (?P<ts_str>\S+) (?P<hostname>\S+) (?P<process_name>\S+) (?P<pid::int64>-|\d+) \S+ (?P<extracted_message>.*)$/

| make_col _ts: parse_isotime(ts_str)
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### Cron (`CRONLOG`)

```
// Fields: ts_str, hostname, process_name, pid, username, cron_action, extracted_message
extract_regex body, /(?P<ts_str>\w{3} +\d{1,2} \d{2}:\d{2}:\d{2}) (?P<hostname>\S+) (?P<process_name>[^\[:\s]+)(?:\[(?P<pid::int64>\d+)\])?:\s+\((?P<username>[^)]+)\) (?P<cron_action>[A-Z ]+) \((?P<extracted_message>[^)]*)\)/

| make_col _ts: parse_timestamp(ts_str, "MON DD HH24:MI:SS", "UTC")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### Java / Tomcat 8.x (`CATALINA8_LOG` / `TOMCAT8_LOG`)

```
// Fields: ts_str, log_level, thread_name, java_class, java_method, extracted_message
// Tomcat 8 format: "28-Mar-2026 10:14:32.059"
extract_regex body, /(?P<ts_str>\d{2}-\w{3}-\d{4} \d{2}:\d{2}:\d{2}\.\d+) (?P<log_level>\w+) \[(?P<thread_name>[^\]]+)\] (?P<java_class>[a-zA-Z$_][a-zA-Z$_0-9.]*)\.(?P<java_method>\S+?) (?P<extracted_message>.*)$/

| make_col _ts: parse_timestamp(ts_str, "DD-MON-YYYY HH24:MI:SS.FF3")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

### Java / Tomcat 7.x (`CATALINA7_LOG` / `TOMCAT7_LOG`)

```
// Fields: ts_str, java_class, java_method, log_level, extracted_message
// Tomcat 7 format: "Mar 28, 2026 10:14:32 AM"
extract_regex body, /(?P<ts_str>\w+ \d{1,2}, \d{4} \d{1,2}:\d{2}:\d{2} [AP]M) (?P<java_class>[a-zA-Z$_][a-zA-Z$_0-9.]*) (?P<java_method>\w+)\s+(?P<log_level>\w+): (?P<extracted_message>.*)$/

| make_col _ts: parse_timestamp(ts_str, "MON DD, YYYY HH12:MI:SS AM")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

### Java stack trace line (`JAVASTACKTRACEPART`)

```
extract_regex body, /\s+at (?P<java_class>[a-zA-Z$_][a-zA-Z$_0-9.]*)\.(?P<java_method>[a-zA-Z$_][a-zA-Z$_0-9]*)\((?P<source_file>[^:)]+)(?::(?P<source_line::int64>\d+))?\)/
```

---

### MongoDB (`MONGO3_LOG`)

```
// Fields: ts_str, log_level, mongo_component, mongo_context, extracted_message
// MongoDB ISO timestamp with tz offset → parse_timestamp
extract_regex body, /(?P<ts_str>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d+[+-]\d{4}) (?P<log_level>\w) (?P<mongo_component>\S+)\s+\[(?P<mongo_context>[^\]]*)\] (?P<extracted_message>.*)$/

| make_col _ts: parse_timestamp(ts_str, "YYYY-MM-DDTHH24:MI:SS.FF3TZHTZM")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### Redis monitor log (`REDISMONLOG`)

```
// Fields: ts_str, redis_db_id, client_ip, client_port, redis_command, redis_args
// Redis timestamp is a Unix float → timestamp_s()
extract_regex body, /(?P<ts_str>[\d.]+) \[(?P<redis_db_id::int64>\d+) (?P<client_ip>\S+):(?P<client_port::int64>\d+)\] "(?P<redis_command>\w+)"\s?(?P<redis_args>.*)$/

| make_col _ts: timestamp_s(int64(float64(ts_str)))
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### PostgreSQL (`POSTGRESQL`)

```
// Fields: ts_str, event_timezone, username, connection_id, pid
// PostgreSQL: "2026-03-28 10:14:32.059 UTC" — ISO-like, use parse_isotime
extract_regex body, /(?P<ts_str>\S+ \d{2}:\d{2}:\d{2}[.,]\d+) (?P<event_timezone>\S+) (?P<username>\S+) (?P<connection_id>\S+) (?P<pid::int64>\d+)/

| make_col _ts: parse_timestamp(ts_str, "YYYY-MM-DD HH24:MI:SS.FF3", event_timezone)
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### AWS S3 Access Log (`S3_ACCESS_LOG`)

```
// Fields: bucket_owner, bucket, ts_str, client_ip, client_user_id, request_id,
//         operation, key, http_method, url, http_version, status_code, error_code,
//         bytes_sent, object_size, total_time, turn_around_time, referrer, user_agent
// S3 timestamp format: [28/Mar/2026:10:22:15 +0000] → HTTPDATE
extract_regex body, /(?P<bucket_owner>\S+) (?P<bucket>\S+) \[(?P<ts_str>[^\]]+)\] (?P<client_ip>\S+) (?P<client_user_id>\S+) (?P<request_id>\S+) (?P<operation>\S+) (?P<key>\S+) "(?P<http_method>\w+) (?P<url>\S+) HTTP\/(?P<http_version>[^"]+)" (?P<status_code::int64>-|\d+) (?P<error_code>\S+) (?P<bytes_sent::int64>-|\d+) (?P<object_size::int64>-|\d+) (?P<total_time::int64>-|\d+) (?P<turn_around_time::int64>-|\d+) "(?P<referrer>[^"]*)" "(?P<user_agent>[^"]*)"/

| make_col _ts: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### AWS ELB Access Log (`ELB_V1_HTTP_LOG`)

```
// Fields: ts_str, elb_name, source_ip, source_port, backend_ip, backend_port,
//         request_time, backend_time, response_time, status_code, backend_status,
//         request_bytes, response_bytes, http_method, url, http_version, user_agent
// ELB timestamp is ISO 8601 → parse_isotime
extract_regex body, /(?P<ts_str>\S+) (?P<elb_name>\S+) (?P<source_ip>[^:]+):(?P<source_port::int64>\d+) (?P<backend_ip>[^:]+):(?P<backend_port::int64>\d+) (?P<request_time::float64>-1|[\d.]+) (?P<backend_time::float64>-1|[\d.]+) (?P<response_time::float64>-1|[\d.]+) (?P<status_code::int64>\d+) (?P<backend_status::int64>-|\d+) (?P<request_bytes::int64>\d+) (?P<response_bytes::int64>\d+) "(?P<http_method>\w+) (?P<url>\S+) HTTP\/(?P<http_version>[^"]+)"/

| make_col _ts: parse_isotime(ts_str)
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### HAProxy HTTP (`HAPROXYHTTP`)

```
// Fields: client_ip, client_port, ts_str, frontend_name, backend_name, server_name,
//         timings, status_code, bytes_read, http_method, url, http_version
// HAProxy timestamp: [28/Mar/2026:10:22:15.123] → HTTPDATE with ms
extract_regex body, /(?P<client_ip>\S+):(?P<client_port::int64>\d+) \[(?P<ts_str>[^\]]+)\] (?P<frontend_name>\S+) (?P<backend_name>[^/]+)\/(?P<server_name>\S+) (?P<timings>\S+) (?P<status_code::int64>\d+) (?P<bytes_read::int64>\d+) \S+ \S+ \S+ "(?P<http_method>\w+) (?P<url>\S+) HTTP\/(?P<http_version>[^"]+)"/

| make_col _ts: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS.FF3")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

## Conversion examples

### Example 1 — Custom Grok with ECS field names

**Input Grok:**
```
%{TIMESTAMP_ISO8601:[event][created]} %{LOGLEVEL:[log][level]} \[%{DATA:[process][thread][name]}\] %{JAVACLASS:[java][log][origin][class][name]}: %{GREEDYDATA:message}
```

**OPAL output:**
```
// Fields: ts_str (→ set_timestamp), log_level, thread_name, java_class, extracted_message
// TIMESTAMP_ISO8601 → parse_isotime; GREEDYDATA → extracted_message
extract_regex body, /(?P<ts_str>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}[^\s]*) (?P<log_level>[A-Za-z]+) \[(?P<thread_name>[^\]]*)\] (?P<java_class>[a-zA-Z$_][a-zA-Z$_0-9.]*): (?P<extracted_message>.*)$/

| make_col _ts: parse_isotime(ts_str)
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
```

---

### Example 2 — Optional field with `-` placeholder

**Input Grok:**
```
%{IPORHOST:[source][address]} (?:-|%{HTTPDUSER:[user][name]}) \[%{HTTPDATE:timestamp}\]
```

**OPAL output:**
```
// source_address always captured; user_name optional (may be "-")
extract_regex body, /(?P<source_address>\S+) (?:-|(?P<user_name>\S+)) \[(?P<ts_str>[^\]]+)\]/

| make_col _ts: parse_timestamp(ts_str, "DD/MON/YYYY:HH24:MI:SS TZHTZM")
| set_timestamp options(max_time_diff:4h), _ts
| drop_col _ts, ts_str
// then handle optional: make_col user_name: if_null(user_name, "-")
```

---

### Example 3 — Inline raw regex in Grok

**Input Grok:**
```
(?<session_id>[0-9a-f]{8}) %{WORD:action} %{NUMBER:duration_ms:float}
```

**OPAL output:**
```
// Grok (?<name>...) → OPAL (?P<name>...)
// NUMBER regex avoids (?:...) — use character classes instead
extract_regex body, /(?P<session_id>[0-9a-f]{8}) (?P<action>\w+) (?P<duration_ms::float64>[+-]?[0-9]+[.,]?[0-9]*)/
```
