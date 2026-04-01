---
name: json-event-to-metric
description: >
  Use this skill whenever the user has a JSON event — from any source such as Logstash, Filebeat, Metricbeat, Elastic Agent, Beats, Prometheus remote_write, or any other JSON shipper — and wants to convert it into an Observe metric interface in OPAL. Trigger phrases include: "json event to observe metrics", "elastic event to observe metrics", "convert my event", "convert my elastic event", "turn this event into a metric", "ingest data as observe metric", "ingest elastic data as observe metric", "logstash event to observe", "metricbeat data to observe metric", "filebeat to observe metric", "elastic agent to observe", "beats to observe", "prometheus to observe", "prometheus remote_write to observe", "create a metric dataset", "metric interface for my data", "how do I make a metric from this JSON", "identify which fields are metrics vs tags", "my data comes from logstash", "I'm sending data via the elastic endpoint", "I'm sending prometheus metrics", "elastic agent prometheus", "prometheus collector dataset", "prometheus remote_write dataset", or any time the user pastes a raw JSON event (including Prometheus-style events with __name__ and label fields, or Elastic Agent-transformed events with a nested prometheus object) and wants to build an OPAL pipeline that outputs a metric dataset in Observe. Always use this skill when the user shows a sample event and wants to track or chart values in Observe.
---

# JSON Event → Observe Metrics Skill

You are an expert at converting raw JSON events — from Logstash, Filebeat, Metricbeat, Elastic Agent, any Beats shipper, or any other JSON source — into correct, efficient OPAL metric interfaces. Your job is to look at a sample event, classify every field, and produce the leanest possible pipeline.

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
pick_col FIELDS, BUNDLE_TIMESTAMP

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
pick_col FIELDS, BUNDLE_TIMESTAMP

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
pick_col FIELDS, BUNDLE_TIMESTAMP

// 1. Save timestamp before transforming FIELDS
make_col timestamp: parse_isotime(string(FIELDS["@timestamp"]))

// 2. Build the tags object first using drop_fields — drop internals and metric fields
//    drop_fields removes named keys; what remains are the dimension fields
make_col tags: drop_fields(FIELDS, "@timestamp", "@version", "message",
                                   "latency_ms", "response_code", "error")

// 3. Build the metrics object — apply type transforms now
//    Boolean fields must be cast to float64(bool(...)) here
//    make_object syntax: alternating name: value pairs
make_col metrics: make_object(
  "latency_ms":    float64(FIELDS["latency_ms"]),
  "response_code": float64(FIELDS["response_code"]),
  "is_error":      float64(bool(FIELDS["error"]))
)

//    Alternatively, use select_fields to keep only specific dimension keys in tags:
//    make_col tags: select_fields(FIELDS, "host", "app", "environment")

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
set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp

interface "metric"
```

> **Why `tags` survives the flatten:** `flatten_leaves metrics` only expands the `metrics` column. All other columns — including the `tags` object — are duplicated onto every output row unchanged, so dimensions are automatically carried across every metric row. There is no need to break `tags` apart into individual columns; Observe will index each key within the object as a dimension.

> **`make_object` signature:** `make_object(key1: val1, key2: val2, ...) -> object` — use `name: value` pairs, not `name, value`. Confirmed from OPAL function reference.

> **`drop_fields` / `select_fields`:** `drop_fields(obj, "key1", "key2", ...)` removes named keys. `select_fields(obj, "key1", "key2", ...)` keeps only named keys. Both return an object.

---

### Pattern C — Dynamic fields → flatten_leaves (elegant when fields are unknown or numerous)

When you don't know the metric field names in advance, or there are many of them, use `flatten_leaves` to automatically expand every leaf field into its own row. This avoids enumerating each field in a `union`.

**Key principle**: extract dimensions and timestamp *before* flattening — the flatten removes the `FIELDS` column, so anything you need must be saved first.

```
pick_col FIELDS, BUNDLE_TIMESTAMP

// 1. Save dimensions and timestamp before FIELDS is consumed by flatten
make_col timestamp: parse_isotime(string(FIELDS["@timestamp"]))
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
set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp

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
| E | Data arrives directly from Prometheus remote_write (raw) — `__name__` is the metric name, labels are dimensions |
| F | Data arrives from Prometheus via **Elastic Agent** — metric name is an object key under `prometheus`, labels are under `prometheus.labels` |

---

### Pattern E — Prometheus remote_write / scrape (raw, without Elastic Agent)

Prometheus sends one sample per event. `__name__` is the metric name, `value` is the numeric sample, and every other field is a label (dimension). Timestamps arrive as epoch **milliseconds**.

```json
{
  "__name__": "http_requests_total",
  "job":       "api-server",
  "instance":  "localhost:8080",
  "method":    "GET",
  "status":    "200",
  "value":     1234.0,
  "timestamp": 1711619200000
}
```

```
pick_col FIELDS, BUNDLE_TIMESTAMP

// Metric name is carried in __name__
make_col metric: string(FIELDS["__name__"])
make_col value:  float64(FIELDS["value"])

// Prometheus timestamps are epoch-milliseconds
make_col timestamp: timestamp_ms(int64(FIELDS["timestamp"]))
set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp

// All remaining labels become dimensions — drop the fields we've already extracted
make_col tags: drop_fields(FIELDS, "__name__", "value", "timestamp")
drop_col FIELDS

interface "metric"
```

#### Histogram metrics

Prometheus histograms produce three series per observed metric: `_bucket` (one row per `le` boundary), `_sum`, and `_count`. They need no special handling — each arrives as a separate event and maps naturally to the pattern above. The `le` label on bucket events is preserved in `tags` automatically.

```json
{ "__name__": "http_request_duration_seconds_bucket", "le": "0.1", "job": "api", "value": 42.0, "timestamp": 1711619200000 }
{ "__name__": "http_request_duration_seconds_sum",    "job": "api", "value": 9.3, "timestamp": 1711619200000 }
{ "__name__": "http_request_duration_seconds_count",  "job": "api", "value": 87.0, "timestamp": 1711619200000 }
```

All three pass through the same pipeline unchanged. If you want to isolate only buckets, filter early:

```
filter string(FIELDS["__name__"]) ~ "_bucket$"
pick_col FIELDS, BUNDLE_TIMESTAMP
...
```

#### Counter vs gauge — setting `metricType`

Prometheus counters have names ending in `_total`; gauges do not. You can annotate automatically:

```
make_col metricType: if(metric ~ "_total$", "counter", "gauge")
```

Place this after `make_col metric` and before `interface "metric"`.

> **ISO timestamp variant** — if your Prometheus source sends an ISO string instead of epoch-ms, replace the timestamp line with:
> ```
> make_col timestamp: parse_isotime(string(FIELDS["timestamp"]))
> ```

---

### Pattern F — Prometheus via Elastic Agent (collector or remote_write dataset)

When Elastic Agent scrapes or receives Prometheus metrics it **transforms the data before forwarding to Observe**. The metric name is no longer a field value — it becomes an **object key** inside a `prometheus` object. Labels are grouped under `prometheus.labels`. The `@timestamp` is ECS-standard ISO.

**Event shape (use_types: true, the default):**

```json
{
  "@timestamp": "2024-01-15T10:30:45.123Z",
  "prometheus": {
    "labels": {
      "instance": "localhost:9090",
      "job":      "prometheus"
    },
    "labels_fingerprint": "abc123",
    "go_threads":               { "value":   10.0 },
    "http_requests_total":      { "counter": 1234.0, "rate": 0.5 },
    "http_request_duration_seconds": {
      "histogram": { "values": [0.1, 0.5, 1.0], "counts": [10, 5, 2] }
    }
  },
  "host":        { "name": "my-host" },
  "data_stream": { "dataset": "prometheus.collector", "namespace": "default", "type": "metrics" }
}
```

Because the metric name is an object key, use `flatten_leaves` on the metrics sub-object to recover one row per metric:

```
pick_col FIELDS, BUNDLE_TIMESTAMP

// 1. Timestamp — ECS standard ISO string
make_col timestamp: parse_isotime(string(FIELDS["@timestamp"]))

// 2. Tags — drop internal Prometheus labels (e.g. 'le' used only for histogram bucket boundaries)
//    For most metrics all labels are useful; drop selectively as needed
make_col tags: drop_fields(FIELDS["prometheus"]["labels"], "le")

//    If all labels are valid dimensions, use the labels object directly:
//    make_col tags: FIELDS["prometheus"]["labels"]

// 3. Metrics object — drop labels and fingerprint, leaving one key per metric name
make_col metrics_obj: drop_fields(FIELDS["prometheus"], "labels", "labels_fingerprint")

drop_col FIELDS

// 4. Flatten to one row per leaf
//    Gauge paths:   "go_threads.value"
//    Counter paths: "http_requests_total.counter", "http_requests_total.rate"
//    Histogram paths produce array elements — filtered out in step 5
flatten_leaves metrics_obj, true

// 5. Keep only numeric leaves — discards histogram array values and any strings
filter _c_metrics_obj_type in ("float64", "int64")

// 6. Optional: drop rate rows if you only want the raw counter (avoid double-counting)
// filter not (_c_metrics_obj_path ~ "\\.rate$")

// 7. Metric name is the full leaf path, e.g. "http_requests_total.counter"
rename_col metric: _c_metrics_obj_path
make_col   value:  float64(_c_metrics_obj_value)
drop_col   _c_metrics_obj_type, _c_metrics_obj_value

set_timestamp options(max_time_diff:4h), timestamp
drop_col timestamp

interface "metric"
```

> **`use_types: false` variant** — if `use_types` is disabled in the Elastic Agent policy, metric values are plain scalars directly under the metric key rather than sub-objects with `.value`/`.counter`. The pipeline is identical; `flatten_leaves` still recovers the leaf values, and the path will simply be `"go_threads"` instead of `"go_threads.value"`.

> **Histograms** — with `use_types: true`, histogram metrics are stored as `{ "histogram": { "values": [...], "counts": [...] } }`. These produce non-numeric leaf types and are automatically dropped by the `filter _c_metrics_obj_type in ("float64", "int64")` step. Handle them separately if needed.

> **Extra metadata** — `host`, `agent`, `data_stream` fields arrive alongside `prometheus`. They are dropped with `drop_col FIELDS` after the metrics object is extracted. Add them to `tags` before dropping FIELDS if you need them as dimensions:
> ```
> make_col tags: make_object(
>   "instance":   string(FIELDS["prometheus"]["labels"]["instance"]),
>   "job":        string(FIELDS["prometheus"]["labels"]["job"]),
>   "host":       string(FIELDS["host"]["name"]),
>   "dataset":    string(FIELDS["data_stream"]["dataset"])
> )
> ```

---

## Field classification

When the user gives you a sample event, classify every field before writing the pipeline:

| Field type | Examples | Becomes |
|---|---|---|
| Timestamp | `@timestamp`, `timestamp` (epoch-ms) | `set_timestamp` |
| Metric (numeric) | `latency_ms`, `cpu_percent`, `count` | `metric`/`value` rows |
| Metric name | `__name__` (Prometheus) | `metric` column |
| Dimension (tag) | `host`, `app`, `region`, `service`, Prometheus labels | Extra string columns / `tags` object on each metric row |
| Logstash internals | `@version`, `tags`, `message` | **Discard** — never useful in metrics |
| Prometheus internals (raw) | `__name__`, `value`, `timestamp` | Extracted — drop from `tags` via `drop_fields` |
| Prometheus via Elastic Agent | `prometheus.<name>.value`, `prometheus.<name>.counter` | Recovered via `flatten_leaves` on the `prometheus` object |
| Elastic Agent metadata | `prometheus.labels_fingerprint`, `host`, `agent`, `data_stream` | **Discard** or selectively add to `tags` |

---

## Performance rules (in priority order)

1. **Filter first** — if only some events matter, filter on raw `FIELDS` before anything else:
   ```
   filter string(FIELDS["environment"]) = "production"
   pick_col FIELDS, BUNDLE_TIMESTAMP
   ```

2. **`pick_col FIELDS, BUNDLE_TIMESTAMP` immediately** — drops `EXTRA`, `OBSERVATION_KIND`, and all other Observe ingest metadata columns you'll never use. Always retain `BUNDLE_TIMESTAMP` as the ingest-time reference for `set_timestamp`.

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
