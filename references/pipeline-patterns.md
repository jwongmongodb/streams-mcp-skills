# Pipeline Patterns Reference

**Official examples repo**: https://github.com/mongodb/ASP_example
Always consult the official repo for the latest validated patterns before creating processors.

## Pipeline Stage Categories

Stages must follow this ordering. Understanding categories helps compose valid pipelines:

| Category | Stages | Rules |
|----------|--------|-------|
| **Source** (1, required) | `$source` | Must be first. One per pipeline. |
| **Stateless Processing** | `$match`, `$project`, `$addFields`, `$unset`, `$unwind`, `$replaceRoot`, `$redact` | No state or memory overhead. Place `$match` first to reduce volume. |
| **Enrichment** | `$lookup`, `$https` | I/O-bound. Use `parallelism` for throughput. Place `$https` after windows. |
| **Stateful/Window** | `$tumblingWindow`, `$hoppingWindow`, `$sessionWindow` | Accumulates state in memory. Monitor `memoryUsageBytes`. |
| **Custom Code** | `$function` | JavaScript UDFs. Requires SP30+. |
| **Output** (1+, required) | `$merge`, `$emit` | Must be last. Required for deployed processors. |

## Invalid Constructs

Do NOT use these in streaming pipelines:
- `$$NOW`, `$$ROOT`, `$$CURRENT` — not available in stream processing
- HTTPS connections as `$source` — HTTPS is for `$https` enrichment only
- Kafka `$source` without `topic` — topic field is required
- Pipelines without a sink — `$merge`/`$emit` required for deployed processors (sinkless only works via `sp.process()`)

## Source Patterns

### MongoDB Change Stream
```json
{"$source": {"connectionName": "my-cluster"}}
```

With full document and pushdown pipeline:
```json
{"$source": {
  "connectionName": "my-cluster",
  "db": "mydb", "coll": "mycoll",
  "fullDocument": "updateLookup",
  "fullDocumentBeforeChange": "whenAvailable",
  "pipeline": [{"$match": {"operationType": "insert"}}]
}}
```

### Kafka (topic is REQUIRED)
```json
{"$source": {
  "connectionName": "my-kafka",
  "topic": "my-topic",
  "auto_offset_reset": "earliest",
  "partitionIdleTimeout": {"size": 30, "unit": "second"}
}}
```

### Kinesis
```json
{"$source": {
  "connectionName": "my-kinesis",
  "streamName": "my-stream",
  "consumerARN": "arn:aws:kinesis:us-east-1:123456789:stream/my-stream/consumer/my-consumer:123"
}}
```

### Inline Documents (ephemeral testing only)
```json
{"$source": {"documents": [{"device_id": "sensor-1", "temp": 72.5}]}}
```

## Sink Patterns

### $merge to Atlas
```json
{"$merge": {"into": {"connectionName": "my-atlas", "db": "mydb", "coll": "mycoll"}}}
```

With match behavior and parallelism:
```json
{"$merge": {
  "into": {"connectionName": "my-atlas", "db": "mydb", "coll": "mycoll"},
  "on": "_id", "whenMatched": "replace", "whenNotMatched": "insert",
  "parallelism": 4
}}
```

`whenMatched`: `replace`, `merge`, `delete` (via `$cond`). `whenNotMatched`: `insert`.

Additive merge (append to arrays):
```json
{"$merge": {
  "into": {"connectionName": "my-atlas", "db": "mydb", "coll": "mycoll"},
  "on": "device_id",
  "whenMatched": [{"$addFields": {"readings": {"$concatArrays": ["$readings", "$$new.readings"]}}}],
  "whenNotMatched": "insert"
}}
```

Dynamic routing:
```json
{"$merge": {"into": {
  "connectionName": "my-atlas", "db": "mydb",
  "coll": {"$cond": {"if": {"$eq": ["$priority", "high"]}, "then": "alerts", "else": "events"}}
}}}
```

### $emit to Kafka
```json
{"$emit": {
  "connectionName": "my-kafka", "topic": "output-topic",
  "key": {"field": "device_id", "format": "string"}
}}
```

Key formats: `string`, `json`, `int`, `long`, `binData`. Tombstone support: `"tombstoneWhen": {"$expr": {"$eq": ["$status", "deleted"]}}`.

### $emit to Kinesis
```json
{"$emit": {"connectionName": "my-kinesis", "streamName": "out", "partitionKey": "$device_id"}}
```

### $emit to S3
```json
{"$emit": {
  "connectionName": "my-s3", "bucket": "my-bucket",
  "prefix": {"$concat": ["data/", {"$dateToString": {"format": "%Y/%m/%d", "date": "$timestamp"}}]},
  "writeOptions": {"count": 1000}
}}
```

## Window Patterns

### Tumbling (Ref: `quickstarts/01_changestream_basic.json`)
```json
{"$tumblingWindow": {
  "interval": {"size": 5, "unit": "minute"},
  "pipeline": [{"$group": {"_id": "$deviceId", "avg": {"$avg": "$temp"}, "count": {"$sum": 1}}}]
}}
```

### Hopping (with allowedLateness)
```json
{"$hoppingWindow": {
  "interval": {"size": 5, "unit": "minute"},
  "hopSize": {"size": 1, "unit": "minute"},
  "allowedLateness": {"size": 15, "unit": "second"},
  "pipeline": [{"$group": {"_id": "$region", "total": {"$sum": "$amount"}}}]
}}
```

### Session (Ref: `example_processors/sessionWindow/`)
```json
{"$sessionWindow": {
  "gap": {"size": 5, "unit": "minute"}, "key": "$userId",
  "pipeline": [{"$group": {"_id": "$userId", "actions": {"$push": "$action"}, "count": {"$sum": 1}}}]
}}
```

### Late data (Ref: `example_processors/lateData/`)
```json
{"$tumblingWindow": {
  "interval": {"size": 1, "unit": "minute"},
  "allowedLateness": {"size": 30, "unit": "second"},
  "boundaryType": "eventTime",
  "pipeline": [{"$group": {"_id": "$sensorId", "max": {"$max": "$value"}}}]
}}
```

`boundaryType`: `eventTime` (document timestamp) or `processTime` (wall clock, default).

## Enrichment Patterns

### $https (Ref: `example_processors/http_operator/`)
```json
{"$https": {
  "connectionName": "my-api",
  "path": {"$concat": ["/users/", "$userId"]},
  "method": "GET", "as": "userInfo", "onError": "dlq"
}}
```

`onError`: `dlq` (recommended), `discard`, `fail`. Store auth in connection settings, not pipeline. Place `$https` after windows to batch requests.

### $lookup
```json
{"$lookup": {
  "connectionName": "my-atlas",
  "from": {"db": "mydb", "coll": "users"},
  "localField": "userId", "foreignField": "_id", "as": "user",
  "parallelism": 2
}}
```

## Real-Time Alerting with Severity Routing
```json
[
  {"$source": {"connectionName": "my-kafka", "topic": "metrics"}},
  {"$addFields": {
    "alert_type": {"$switch": {
      "branches": [
        {"case": {"$gte": ["$value", 100]}, "then": "critical"},
        {"case": {"$gte": ["$value", 75]}, "then": "warning"},
        {"case": {"$gte": ["$value", 50]}, "then": "info"}
      ],
      "default": "none"
    }}
  }},
  {"$match": {"alert_type": {"$ne": "none"}}},
  {"$merge": {"into": {
    "connectionName": "my-atlas", "db": "monitoring",
    "coll": {"$cond": {"if": {"$eq": ["$alert_type", "critical"]}, "then": "critical_alerts", "else": "alerts"}}
  }}}
]
```

## Full API Enrichment Pipeline (Ref: `example_processors/http_operator/`)
```json
[
  {"$source": {"connectionName": "my-kafka", "topic": "orders"}},
  {"$match": {"status": "pending"}},
  {"$https": {
    "connectionName": "my-api",
    "path": {"$concat": ["/customers/", "$customerId"]},
    "method": "GET",
    "as": "customerInfo",
    "onError": "dlq"
  }},
  {"$addFields": {
    "customerName": {"$ifNull": ["$customerInfo.name", "unknown"]},
    "customerTier": {"$ifNull": ["$customerInfo.tier", "standard"]},
    "enrichmentSucceeded": {"$ne": [{"$type": "$customerInfo"}, "missing"]}
  }},
  {"$unset": "customerInfo"},
  {"$merge": {"into": {"connectionName": "my-atlas", "db": "orders", "coll": "enriched"}}}
]
```

## Time-based Aggregation with Statistics
```json
{"$tumblingWindow": {
  "interval": {"size": 5, "unit": "minute"},
  "pipeline": [
    {"$group": {
      "_id": "$sensorId",
      "avgValue": {"$avg": "$reading"},
      "maxValue": {"$max": "$reading"},
      "minValue": {"$min": "$reading"},
      "stdDev": {"$stdDevPop": "$reading"},
      "sampleCount": {"$sum": 1},
      "windowStart": {"$first": "$_stream_meta.window.start"},
      "windowEnd": {"$first": "$_stream_meta.window.end"}
    }}
  ]
}}
```

## Pipeline Optimization: Selective $project

After filtering, project only needed fields to reduce document size for downstream stages:
```json
[
  {"$source": {"connectionName": "my-kafka", "topic": "events"}},
  {"$match": {"status": "active", "region": "us-east"}},
  {"$project": {"userId": 1, "amount": 1, "timestamp": 1, "region": 1}},
  {"$lookup": {"connectionName": "my-atlas", "from": {"db": "mydb", "coll": "users"},
    "localField": "userId", "foreignField": "_id", "as": "user", "parallelism": 2}},
  {"$merge": {"into": {"connectionName": "my-atlas", "db": "mydb", "coll": "enriched"}}}
]
```

## Window Metadata

Inside window pipelines, `_stream_meta.window.start` and `_stream_meta.window.end` provide the window boundary timestamps:
```json
{"$tumblingWindow": {
  "interval": {"size": 5, "unit": "minute"},
  "pipeline": [
    {"$group": {
      "_id": "$deviceId",
      "windowStart": {"$first": "$_stream_meta.window.start"},
      "windowEnd": {"$first": "$_stream_meta.window.end"},
      "avg": {"$avg": "$temp"}
    }}
  ]
}}
```

## Complex Event Processing (Ref: `example_processors/` fraud detection patterns)
```json
[
  {"$source": {"connectionName": "my-kafka", "topic": "transactions"}},
  {"$tumblingWindow": {
    "interval": {"size": 5, "unit": "minute"},
    "pipeline": [
      {"$group": {
        "_id": "$userId",
        "txnCount": {"$sum": 1},
        "totalAmount": {"$sum": "$amount"},
        "uniqueLocations": {"$addToSet": "$location"},
        "transactions": {"$push": {"amount": "$amount", "merchant": "$merchant", "ts": "$timestamp"}}
      }},
      {"$addFields": {
        "suspiciousLocations": {"$gt": [{"$size": "$uniqueLocations"}, 3]},
        "highVelocity": {"$gt": ["$txnCount", 10]},
        "largeTotal": {"$gt": ["$totalAmount", 5000]}
      }},
      {"$match": {"$or": [
        {"suspiciousLocations": true},
        {"highVelocity": true},
        {"largeTotal": true}
      ]}}
    ]
  }},
  {"$merge": {"into": {"connectionName": "my-atlas", "db": "fraud", "coll": "alerts"}}}
]
```

## Data Quality & Normalization Pattern
```json
{"$addFields": {
  "normalized_email": {"$toLower": {"$trim": {"input": "$email"}}},
  "data_quality_score": {"$sum": [
    {"$cond": [{"$ne": [{"$type": "$email"}, "missing"]}, 25, 0]},
    {"$cond": [{"$ne": [{"$type": "$name"}, "missing"]}, 25, 0]},
    {"$cond": [{"$ne": [{"$type": "$phone"}, "missing"]}, 25, 0]},
    {"$cond": [{"$ne": [{"$type": "$address"}, "missing"]}, 25, 0]}
  ]}
}}
```

## Graceful Degradation with $ifNull

When enrichment fields may be missing (e.g., `$https` or `$lookup` returns incomplete data):
```json
{"$addFields": {
  "userName": {"$ifNull": ["$userInfo.name", "unknown"]},
  "userTier": {"$ifNull": ["$userInfo.tier", "standard"]},
  "enrichmentSucceeded": {"$ne": [{"$type": "$userInfo"}, "missing"]}
}}
```

## Sample Stream Formats

When `includeSampleData: true` on workspace creation (default), the `sample_stream_solar` connection is auto-created. Additional built-in sample formats available via Sample connections:

| Format | Data type |
|--------|-----------|
| `sample_stream_solar` | Solar panel IoT readings (default) |
| `samplestock` | Stock market tick data |
| `sampleweather` | Weather station readings |
| `sampleiot` | Generic IoT sensor data |
| `samplelog` | Application log events |
| `samplecommerce` | E-commerce transaction data |

## Windowing Rules
- Windows require `$group` inside the window pipeline
- Idle Kafka partitions block windows — use `partitionIdleTimeout`
- `allowedLateness` lets late docs update closed windows

## Checkpoint Resume Constraints
With `resumeFromCheckpoint: true` (default), you CANNOT change: window type, interval, remove windows, or modify `$source`. Set `false` to make these changes (restarts from beginning).

## DLQ Configuration
```json
{"dlq": {"connectionName": "my-atlas", "db": "streams_dlq", "coll": "failed_documents"}}
```
DLQ documents include: original document, error message, stage info, timestamp.

## Tier Sizing & Parallelism

See [sizing-and-parallelism.md](sizing-and-parallelism.md) for the full tier table, parallelism formula, complexity scoring, and cost optimization strategies.
