# Observe /elastic Endpoint — Event Structure Reference

## How Logstash data arrives in Observe

When Logstash sends events to Observe's `/elastic` endpoint (using the Elasticsearch output plugin), each event becomes a row in a raw dataset with these top-level columns:

| Column | Type | Description |
|---|---|---|
| `OBSERVATION_KIND` | string | Always `"event"` for raw ingest rows |
| `FIELDS` | object | The full Logstash JSON payload (everything Logstash sent) |
| `EXTRA` | object | Observe ingest metadata (ingest timestamp, pipeline info, etc.) |
| `BUNDLE_TIMESTAMP` | timestamp | When Observe received the batch |

**Important**: Your Logstash fields are NOT top-level columns — they live inside `FIELDS`. You must use `make_col field_name: type(FIELDS["field_name"])` to promote them.

## Standard Logstash fields inside FIELDS

Every Logstash event includes these by default:

| Field | Type | Notes |
|---|---|---|
| `@timestamp` | ISO 8601 string | Event timestamp — always use this for `set_valid_from` |
| `@version` | string | Always `"1"` — discard |
| `host` | object | `{"name": "hostname"}` — extract with `FIELDS["host"]["name"]` |
| `message` | string | Original log message — optional to keep |
| `tags` | array | Logstash tags — rarely useful in metrics, discard |

## Configuring the Logstash output

Logstash `elasticsearch` output pointing to Observe:
```
output {
  elasticsearch {
    hosts => ["https://your-workspace.observeinc.com/v1/http/elastic"]
    user  => "unused"
    password => "<your-ingest-token>"
    index => "your-dataset-name"
  }
}
```

## Type mapping

Logstash field types map to OPAL cast functions as follows:

| Logstash type | OPAL cast |
|---|---|
| Float/double | `float64(FIELDS["field"])` |
| Integer | `int64(FIELDS["field"])` |
| String | `string(FIELDS["field"])` |
| Boolean | `bool(FIELDS["field"])` |
| ISO timestamp string | `timestamp_s(string(FIELDS["@timestamp"]))` |
| Epoch ms integer | `timestamp_ms(int64(FIELDS["field"]))` |
| Nested object | `string(FIELDS["outer"]["inner"])` |
