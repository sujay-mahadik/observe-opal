---
name: elastic-event-to-metric
description: >
  Use this skill whenever the user has an event arriving at Observe via the /elastic endpoint — from Logstash, Filebeat, Metricbeat, Elastic Agent, or any Elastic Stack shipper — and wants to convert it into an Observe metric interface in OPAL. Trigger phrases include: "elastic event to observe metrics", "convert my elastic event", "turn this event into a metric", "ingest elastic data as observe metric", "logstash event to observe", "metricbeat data to observe metric", "filebeat to observe metric", "elastic agent to observe", "beats to observe", "create a metric dataset from elastic", "metric interface for my elastic data", "how do I make a metric from this JSON", "identify which fields are metrics vs tags", "my data comes from logstash", "I'm sending data via the elastic endpoint", or any time the user pastes a raw JSON event and wants to build an OPAL pipeline that outputs a metric dataset in Observe. Always use this skill when the user shows a sample event and wants to track or chart values in Observe.
---

# Elastic Event → Observe Metrics Skill

You are an expert at converting raw events ingested via Observe's `/elastic` endpoint — from Logstash, Filebeat, Metricbeat, Elastic Agent, or any Beats shipper — into correct, efficient OPAL metric interfaces. Your job is to look at a sample event, classify every field, and produce the leanest possible pipeline.

---

## How the metric interface works

Observe's `interface "metric"` requires **exactly two named columns**:

| Column | Type | Meaning |
|---|---|---|
| `metric` | string | The name of the metric (e.g. `"latency_ms"`, `"cpu_percent"`) |
| `value` | float64 | The numeric value |

Optional metadata columns Observe will display in the metric browser:

| Column | Type | Meaning |
|---|---|---|
| `metricType` | string | e.g. `"gauge"`, `"histogram"` |
| `metricUnit` | string | e.g. `"ms"`, `"bytes"`, `"percent"` |
| `metricDescription` | string | Human-readable description |

Declaration:
```
interface "metric"
```
If your columns are already named `metric` and `value`, that's all you need. If they have different names, bind them:
```
interface "metric", metric:my_name_col, value:my_value_col
```

---

## Setting the timestamp — always use `set_timestamp`

Logstash events arrive as **Event-shaped** data (point-in-time observations). The correct verb is:

```
make_col timestamp: parse_isotime(string(FIELDS["@timestamp"]))
set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp
```

> **Why not `set_valid_from`?** `set_valid_from` is for Resource or Interval datasets. Its support for event shape is deprecated. Logstash events are Event-shaped, so always use `set_timestamp`.

`set_timestamp` options:
- `max_time_diff` — rows where the new timestamp differs from the ingest time by more than this are dropped (default: 1h). Set to `4h` or larger for delayed data.
- `frame(back: 2h, ahead: 20m)` — alternative to `max_time_diff` for asymmetric windows.

---

## The three patterns

### Pattern A — Event already carries metric name + value (one metric per event)

```json
{ "@timestamp": "...", "metric_name": "latency_ms", "sample_value": 142.5, "host": "web-01" }
```

```
pick_col FIELDS

make_col metric: string(FIELDS["metric_name"])
make_col value:  float64(FIELDS["sample_value"])

make_col timestamp: parse_isotime(string(FIELDS["@timestamp"]))
set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp

make_col host: string(FIELDS["host"])
drop_col FIELDS

interface "metric"
```

---

### Pattern B — Multiple numeric fields per event → union (explicit, recommended when fields are known)

Most Logstash events look like this — several numeric fields alongside dimension fields:

```json
{
  "@timestamp": "2026-03-28T10:00:00Z",
  "host": { "name": "web-01" }, "app": "checkout", "environment": "production",
  "latency_ms": 142.5, "response_code": 200, "error": false
}
```

Use `union` to create one row per metric, then apply shared timestamp and dimensions:

```
// Filter early if needed
filter string(FIELDS["environment"]) = "production"
pick_col FIELDS

// One branch per metric
union
  (make_col metric: "latency_ms",    value: float64(FIELDS["latency_ms"])),
  (make_col metric: "response_code", value: float64(FIELDS["response_code"])),
  (make_col metric: "is_error",      value: float64(bool(FIELDS["error"])))

// Timestamp — shared across all branches
| make_col timestamp: parse_isotime(string(FIELDS["@timestamp"]))
| set_timestamp options(max_time_diff:4h), timestamp
| drop_col timestamp

// Dimensions — shared across all branches
| make_col host:        string(FIELDS["host"]["name"])
| make_col app:         string(FIELDS["app"])
| make_col environment: string(FIELDS["environment"])

| drop_col FIELDS

| interface "metric"
```

---

### Pattern D — Structured object separation + flatten_leaves (recommended for most cases)

Build explicit `metrics` and `tags` objects from FIELDS, then use `flatten_leaves` on the metrics object. This gives you the cleanest separation: type transforms and metric selection happen upfront at object construction time, and the tags object rides along through the flatten untouched.

**Key advantages:**
- Boolean/special fields are cast *at construction time*, not after flattening
- Metrics and dimensions are explicitly named — no accidental inclusion
- No need to enumerate one union branch per metric (unlike Pattern B)
- `tags` object survives `flatten_leaves` intact so dimensions appear on every metric row

```json
{
  "@timestamp": "2026-03-28T10:00:00Z",
  "@version": "1",
  "message": "request complete",
  "host": { "name": "web-01" }, "app": "checkout", "environment": "production",
  "latency_ms": 142.5, "response_code": 200, "error": false
}
```

```
filter string(FIELDS["environment"]) = "production"
pick_col FIELDS

// 1. Save timestamp before transforming FIELDS
make_col _ts: string(FIELDS["@timestamp"])

// 2. Build the metrics object — apply type transforms now
//    Boolean fields must be cast to float64(bool(...)) here
make_col metrics: make_object(
  "latency_ms",    float64(FIELDS["latency_ms"]),
  "response_code", float64(FIELDS["response_code"]),
  "is_error",      float64(bool(FIELDS["error"]))
)

// 3. Build the tags object — two equivalent approaches:
//
//    a) Explicit selection (best when dimensions are well-known):
make_col tags: make_object(
  "host",        string(FIELDS["host"]["name"]),
  "app",         string(FIELDS["app"]),
  "environment", string(FIELDS["environment"])
)
//
//    b) Drop-unwanted approach (best when internals are few but dimensions are many):
//       drop_fields(obj, key1, key2, ...) removes named keys from an object.
//       Alternatively, select_fields(obj, key1, key2, ...) keeps only named keys.
//    make_col tags: drop_fields(FIELDS, "@timestamp", "@version", "message",
//                                       "latency_ms", "response_code", "error")

drop_col FIELDS

// 4. Flatten metrics → one row per metric
//    After this, each row has: _c_metrics_path (metric name), _c_metrics_value,
//    plus the 'tags' object column still intact on every row
flatten_leaves metrics

// 5. Rename to the required metric interface columns
rename_col metric: _c_metrics_path
make_col   value:  float64(_c_metrics_value)
drop_col   _c_metrics_value

// 6. Keep 'tags' as an object column — it's already on every metric row
//    Observe indexes each key inside 'tags' as a dimension automatically

// 7. Set event timestamp — shared across all metric rows
make_col timestamp: parse_isotime(_ts)
set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp, _ts

interface "metric"
```

> **Why `tags` survives the flatten:** `flatten_leaves metrics` only expands the `metrics` column. All other columns — including the `tags` object — are duplicated onto every output row unchanged, so dimensions are automatically carried across every metric row. There is no need to break `tags` apart into individual columns; Observe will index each key within the object as a dimension.

> **`make_object` signature:** `make_object(key1: string, val1, key2: string, val2, ...) -> object` — alternating string keys and values. Confirmed from OPAL function reference.

> **`drop_fields` / `select_fields`:** `drop_fields(obj, "key1", "key2", ...)` removes named keys. `select_fields(obj, "key1", "key2", ...)` keeps only named keys. Both return an object.

---

### Pattern C — Dynamic fields → flatten_leaves (elegant when fields are unknown or numerous)

When you don't know the metric field names in advance, or there are many of them, use `flatten_leaves` to automatically expand every leaf field into its own row. This avoids enumerating each field in a `union`.

**Key principle**: extract dimensions and timestamp *before* flattening — the flatten removes the `FIELDS` column, so anything you need must be saved first.

```
pick_col FIELDS

// 1. Save dimensions and timestamp before FIELDS is consumed by flatten
make_col _ts:  string(FIELDS["@timestamp"])
make_col host: string(FIELDS["host"]["name"])
make_col app:  string(FIELDS["app"])
make_col env:  string(FIELDS["environment"])

// 2. Flatten FIELDS into one row per leaf — produces _c_FIELDS_path and _c_FIELDS_value
//    Pass true to also get _c_FIELDS_type for type-based filtering
flatten_leaves FIELDS, true

// 3. Keep only the numeric leaf fields (the actual metrics)
//    Discard @timestamp, @version, and any string fields
filter _c_FIELDS_type in ("float64", "int64")

// 4. Name the metric interface columns
rename_col metric: _c_FIELDS_path
make_col value: float64(_c_FIELDS_value)
drop_col _c_FIELDS_type

// 5. Set timestamp from the saved value
make_col timestamp: parse_isotime(_ts)
set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp, _ts

interface "metric"
```

> **Performance note**: Prefer `flatten_leaves` over `flatten` — it skips intermediate object paths and only returns leaf values. `flatten` produces null intermediate rows that you'd need to filter away. Both are accelerable.

**When to use each pattern:**

| Pattern | Use when |
|---|---|
| A | Each event carries a single named metric |
| B | You know the metric fields and want maximum readability (one union branch per metric) |
| C | Fields are fully dynamic or unknown (auto-discovery via type filtering) |
| D | You know the metric fields but want clean object separation + flatten_leaves; especially good when booleans or casts are involved |

---

## Field classification

When the user gives you a sample event, classify every field before writing the pipeline:

| Field type | Examples | Becomes |
|---|---|---|
| Timestamp | `@timestamp` | `set_timestamp` |
| Metric (numeric) | `latency_ms`, `cpu_percent`, `count` | `metric`/`value` rows |
| Dimension (tag) | `host`, `app`, `region`, `service` | Extra string columns on each metric row |
| Logstash internals | `@version`, `tags`, `message` | **Discard** — never useful in metrics |

---

## Performance rules (in priority order)

1. **Filter first** — if only some events matter, filter on raw `FIELDS` before anything else:
   ```
   filter string(FIELDS["environment"]) = "production"
   pick_col FIELDS
   ```

2. **`pick_col FIELDS` immediately** — drops `EXTRA`, `OBSERVATION_KIND`, `BUNDLE_TIMESTAMP` and all other Observe ingest metadata columns you'll never use.

3. **Use `flatten_leaves` not `flatten`** — `flatten` generates null intermediate rows; `flatten_leaves` returns only leaf values. Both accelerable, but `flatten_leaves` is cheaper.

4. **Explicit type casts everywhere** — always use `float64()`, `int64()`, `string()`, `bool()`. Never rely on implicit coercion.

5. **`drop_col FIELDS` before `interface`** — don't carry the raw payload into the metric output.

6. **Watch dimension cardinality** — pod names, user IDs, trace IDs are high-cardinality. They multiply metric storage. Flag these proactively.

---

## Nested field access

Both notations work:
```
FIELDS["kubernetes"]["namespace"]   // bracket notation
FIELDS.kubernetes.namespace         // dot notation — equivalent, shorter
```

---

## Common caveats

**Booleans** must be cast to float64 for the `value` column:
```
make_col value: float64(bool(FIELDS["error"]))
```

**Millisecond timestamps** — if `@timestamp` is epoch-ms not ISO string:
```
make_col timestamp: timestamp_ms(int64(FIELDS["@timestamp"]))
```

**Missing fields** — null check if a field might not exist on all events:
```
make_col value: case(isnull(FIELDS["latency_ms"]), float64(0), true, float64(FIELDS["latency_ms"]))
```

**`set_timestamp` drops rows outside `max_time_diff`** — if you see no data, your source timestamps may be too far from ingest time. Increase `max_time_diff`:
```
set_timestamp options(max_time_diff:24h), timestamp
```

---

## Response style

- Always classify every field in the sample event before writing the pipeline
- Choose the appropriate pattern (A/B/C) and briefly explain why
- Produce a complete, runnable pipeline with inline comments
- Flag high-cardinality dimensions proactively
- Note any type coercion choices that aren't obvious
