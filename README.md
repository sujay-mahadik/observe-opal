# Observe OPAL Plugin

A Claude plugin for teams using [Observe](https://www.observeinc.com/). This plugin helps you write, debug, explain, and learn OPAL queries, convert Logstash events arriving via the `/elastic` endpoint into production-ready metric interfaces, and migrate Logstash Grok patterns to OPAL `extract_regex` statements.

---

## Skills

### `opal-query` — OPAL Query Assistant

Helps with everything OPAL-related:

- **Write queries** — describe what you want to observe in plain language and get a working OPAL pipeline
- **Extract fields** — paste a raw log line (CLF, syslog, app logs) and get `extract_regex` statements to parse every field
- **Explain queries** — walk through an existing query clause by clause
- **Debug queries** — diagnose why a query fails or returns unexpected results
- **Onboard** — guided introduction to OPAL for engineers new to Observe

**Trigger phrases:** "write an OPAL query", "explain this query", "debug my OPAL", "extract fields from this log", "how do I filter by pod", "count errors per service", "show me latency by namespace"

### `elastic-event-to-metric` — Elastic Event → Observe Metrics

Converts any raw event arriving at Observe via the `/elastic` endpoint — from Logstash, Filebeat, Metricbeat, Elastic Agent, or any Beats shipper — into a complete, efficient OPAL pipeline that outputs an `interface "metric"` dataset. Handles field classification, metric/tag separation, timestamp parsing, and `flatten_leaves` patterns.

**Trigger phrases:** "elastic event to observe metrics", "convert my elastic event", "turn this event into a metric", "ingest elastic data as observe metric", "logstash event to observe", "metricbeat to observe metric", "filebeat to observe metric", "elastic agent to observe", "beats to observe", "create a metric dataset from elastic", "how do I make a metric from this JSON", "identify which fields are metrics vs tags", "my data comes from logstash", "I'm sending data via the elastic endpoint"

### `grok-to-opal` — Grok Pattern Converter

Converts Logstash Grok patterns and filters into equivalent OPAL `extract_regex` statements. Supports all ecs-v1 patterns, type casting (`:int`, `:float`), ECS nested field name normalization (`[http][response][status_code]` → `http_response_status_code`), and includes pre-built translations for common log formats (Apache, Syslog, Tomcat, AWS, HAProxy, PostgreSQL, and more).

**Trigger phrases:** "convert this grok", "translate grok to OPAL", "migrate grok filter", "grok pattern in OPAL", "what is the OPAL equivalent of this grok", "rewrite this grok as an extract_regex"

---

## What's covered

The `opal-query` skill includes a comprehensive reference (`references/syntax.md`) covering all 74 active OPAL verbs and key functions, including:

- All projection verbs: `filter`, `make_col`, `pick_col`, `drop_col`, `rename_col`, `extract_regex`
- Aggregation: `statsby`, `timechart`, `timestats`, `aggregate`
- Joins: `join`, `leftjoin`, `fulljoin`, `lookup`, `union`, `follow`, `surrounding`
- Data shaping: `flatten_leaves`, `flatten`, `unpivot`, `pivot`, `topk`, `bottomk`, `sort`, `dedup`
- Time: `set_timestamp`, `set_valid_from`, `parse_isotime`, `timestamp_s/ms/us/ns`
- Objects: `make_object`, `drop_fields`, `select_fields`, `parse_json`, `parse_kvp`, `parse_url`
- Strings: `get_regex`, `contains`, `split_string`, `replace_regex`, `upper`, `lower`, and more
- Window functions: `lag`, `lead`, `rank`, `row_number`, `ntile`
- Type casts, conditional functions, numeric and array functions

---

## Usage

### OPAL queries

Just describe your goal in plain language:

> "Write an OPAL query to count 5xx errors per service in the last hour"

> "Extract the client IP, method, path, and status code from this nginx log:
> `10.0.0.9 - - [28/Mar/2026:10:22:15 +0000] "POST /login HTTP/1.1" 401 312`"

> "Debug this query — it's returning no data: `filter status_code > 400 | timechart 5m, count:count()`"

### Elastic event → Observe metric conversion

Paste a sample event and ask to convert it:

> "My data comes from Logstash into Observe using the /elastic endpoint. Here's a sample event — turn it into a metric interface."

> "I'm using Metricbeat to send data to Observe. Can you build a metric dataset from this event?"

The skill will classify every field, choose the right pattern (union, `make_object` + `flatten_leaves`, or single-metric), and output a complete pipeline.

### Grok → OPAL conversion

Paste a Grok pattern and ask to convert it:

> "Convert this grok to OPAL: `%{IPORHOST:client_ip} %{USER:ident} %{USER:username} \[%{HTTPDATE:timestamp}\] \"%{WORD:http_method} %{URIPATH:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:status_code:int} %{NUMBER:bytes_sent:int}`"

The skill expands every `%{PATTERN:field:type}` token, preserves literals, handles optional fields, and emits a complete `extract_regex` statement with a timestamp pipeline when needed.

---

## Datasets supported

- **Kubernetes Logs** — `body`, `pod`, `namespace`, `container`, `cluster`, `attributes`
- **Tracing / Spans** — `start_time`, `end_time`, `duration`, `service_name`, `error`, `attributes`
- **Metrics** — any metric dataset
- **Logstash / Elastic ingest** — events arriving via `/elastic` with top-level `FIELDS` column

---

## Requirements

No external services or environment variables required. The plugin works entirely from the skills' built-in knowledge.
