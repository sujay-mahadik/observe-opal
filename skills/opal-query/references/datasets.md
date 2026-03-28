# Observe Dataset Reference

Observe workspaces contain several categories of datasets. Understanding which dataset to query is the first step in any investigation.

---

## Kubernetes Logs

Container and pod logs collected from your Kubernetes clusters.

**Columns:**
| Column | Type | Description |
|---|---|---|
| `timestamp` | time | When the log line was emitted |
| `body` | string | The raw log message text |
| `stream` | string | `stdout` or `stderr` |
| `cluster` | string | Kubernetes cluster name |
| `namespace` | string | Kubernetes namespace |
| `container` | string | Container name |
| `pod` | string | Pod name |
| `node` | string | Node name |
| `attributes` | map | Structured key-value pairs from log metadata |
| `resource_attributes` | map | Resource-level attributes (e.g. k8s.node.name) |
| `instrumentation_scope` | string | OTel instrumentation scope |
| `fields` | map | Additional parsed fields |
| `meta` | map | Internal metadata |

**Typical use cases:**
- Searching for error messages or stack traces
- Counting log events per pod/container
- Finding crashes or restarts
- Correlating log output with deployment events

**Example queries:**
```
# Find all error logs in the payments namespace
filter namespace = "payments" and body ~ /error|exception/i
| pick_col timestamp, body, pod, container

# Count errors per pod
filter body ~ /error/i
| statsby count:count(1), group_by(pod)
```

---

## Tracing / Spans

Distributed trace spans collected via OpenTelemetry.

**Columns:**
| Column | Type | Description |
|---|---|---|
| `start_time` | time | Span start (**required in pick_col**) |
| `end_time` | time | Span end (**required in pick_col**) |
| `duration` | number | Span duration in nanoseconds |
| `service_name` | string | Service that emitted the span |
| `span_name` | string | Name of the operation |
| `error` | boolean | Whether the span represents an error |
| `response_status` | string | HTTP/gRPC response status |
| `status_code` | string | OTel status code |
| `status_message` | string | Human-readable status |
| `span_type` | string | `client`, `server`, `internal`, etc. |
| `attributes` | map | Span-level attributes |
| `resource_attributes` | map | Resource-level attributes |
| `parent_span_id` | string | Parent span for trace reconstruction |
| `span_id` | string | Unique span identifier |
| `trace_id` | string | Trace identifier |

> **Critical**: When using `pick_col` on span datasets, you MUST always include `start_time` AND `end_time`. Omitting either causes an error:
> - `need to pick 'valid from' column "start_time"`
> - `need to pick 'valid to' column "end_time"`

**Typical use cases:**
- Latency analysis by service or endpoint
- Error rate tracking
- Trace reconstruction
- Service dependency mapping
- SLI/SLO investigation

**Example queries:**
```
# P99 latency per service
statsby p99_ms:percentile(duration/1000000, 99), group_by(service_name)

# Error rate over time
filter error = true
| timechart 5m, count:count(1), group_by(service_name)

# Slow spans with custom attributes
filter duration > 5000000000   # > 5 seconds in nanoseconds
| make_col loyalty: string(attributes["app.loyalty.level"])
| pick_col start_time, end_time, service_name, span_name, duration, loyalty
```

---

## Metrics

Time-series metric data. Companion datasets named `metric-sma-for-*` provide smoothed moving averages.

**Typical use cases:**
- CPU/memory utilization
- Request rate and error rate
- SLI/SLO tracking
- Capacity planning

The column schema varies by metric. Use `pick_col *` or browse the dataset schema to discover available columns.

---

## Reference Tables

Static or slowly-changing enrichment data (e.g. product catalog, customer records, service ownership registry).

**Typical use cases:**
- Join trace/log data with human-readable names or metadata
- Enrich spans with product prices, categories, or owners
- Look up user/account info from trace attributes

Reference tables are regular Observe datasets. Join them using a `join` clause or extract the key from `attributes` with `make_col` and then look up the value.

**Example: enriching spans with product info**
```
make_col product_id: string(attributes["app.product.id"])
| join Products on product_id = product_id
| pick_col start_time, end_time, service_name, product_id, product_name, category
```
