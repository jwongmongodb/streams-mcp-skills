# Connection Configuration Reference

**Official examples repo**: https://github.com/mongodb/ASP_example

## Important Notes
- HTTPS connections are for `$https` enrichment ONLY — they are NOT valid as `$source` data sources
- Store API authentication in connection settings, never hardcode in processor pipelines
- AWS connections (S3, Kinesis, Lambda) require IAM role ARN registered via Atlas Cloud Provider Access first
- Supported `connectionType` values: `Kafka`, `Cluster`, `S3`, `Https`, `AWSKinesisDataStreams`, `AWSLambda`, `SchemaRegistry`, `Sample`

## MCP Tool Behaviors for Connections

**Elicitation:** When required fields are missing, the build tool auto-prompts for them via an interactive form (MCP elicitation protocol). Do NOT manually ask the user for passwords or bootstrap servers — let the tool collect them.

**Auto-normalization:**
- `bootstrapServers` passed as array → auto-converted to comma-separated string
- `schemaRegistryUrls` passed as string → auto-wrapped in array
- Cluster `dbRoleToExecute` → auto-defaults to `{role: "readWriteAnyDatabase", type: "BUILT_IN"}` if omitted

## connectionConfig by type

### Kafka
```json
{
  "bootstrapServers": "broker1:9092,broker2:9092",
  "authentication": {
    "mechanism": "SCRAM-256",
    "username": "my-user",
    "password": "my-password"
  },
  "security": {
    "protocol": "SASL_SSL"
  }
}
```
**Important:** `bootstrapServers` is a **comma-separated string**, not an array.

All fields above are required. The tool will prompt the user for username/password via elicitation if not provided.

Authentication mechanisms: `PLAIN`, `SCRAM-256`, `SCRAM-512`, `OAUTHBEARER`
Security protocols: `SASL_SSL`, `SASL_PLAINTEXT`, `SSL`

For Confluent Cloud, use `mechanism: "PLAIN"` with your API key as `username` and API secret as `password`.

Kafka supports PrivateLink networking with Confluent Cloud on AWS (see Terraform examples in the ASP_example repo: `terraform/privatelinkConfluentAWS.tf`).

**Important: Networking cannot be modified after connection creation.** To add or change PrivateLink on an existing Kafka connection, you must delete it and recreate it with the networking config. The `networking.access` field format is:
```json
"networking": {"access": {"type": "PRIVATE_LINK", "connectionId": "<Atlas PrivateLink ID>"}}
```
The `connectionId` is the Atlas PrivateLink `_id` (not the AWS service endpoint ID). Use `atlas-streams-discover` → `get-networking` to list available PrivateLink endpoints.

### Cluster (Atlas)
```json
{
  "clusterName": "my-atlas-cluster",
  "dbRoleToExecute": {
    "role": "readWriteAnyDatabase",
    "type": "BUILT_IN"
  }
}
```
`clusterName` is **required** — must be a cluster in the same project (use `atlas-list-clusters` to verify).

`dbRoleToExecute` defaults to `{role: "readWriteAnyDatabase", type: "BUILT_IN"}` if not provided.

Optional: `clusterGroupId` (if cluster is in a different project — requires cross-project access to be enabled at the org level).

### S3
```json
{
  "aws": {
    "roleArn": "arn:aws:iam::123456789:role/streams-s3-role",
    "testBucket": "my-test-bucket"
  }
}
```
**Prerequisite:** The IAM role ARN must be registered in the Atlas project via Cloud Provider Access before creating the connection.

Required IAM policy permissions: `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`.

### Https
```json
{
  "url": "https://api.example.com/webhook",
  "headers": {
    "Authorization": "Bearer token123"
  }
}
```
**IMPORTANT:** HTTPS connections are for `$https` enrichment stages ONLY. They are NOT valid data sources — do not use them in `$source`.

Store all API authentication in the connection config headers, not in the processor pipeline.

#### HTTPS Auth Patterns

**API Key:**
```json
{"url": "https://api.example.com", "headers": {"X-API-Key": "your-api-key"}}
```

**Bearer Token:**
```json
{"url": "https://api.example.com", "headers": {"Authorization": "Bearer your-token"}}
```

**Basic Auth:**
```json
{"url": "https://api.example.com", "headers": {"Authorization": "Basic base64-encoded-credentials"}}
```

**OAuth 2.0 (pre-obtained token):**
```json
{"url": "https://api.example.com", "headers": {"Authorization": "Bearer oauth-access-token"}}
```

### AWSKinesisDataStreams
```json
{
  "aws": {
    "roleArn": "arn:aws:iam::123456789:role/streams-kinesis-role"
  }
}
```
**Prerequisite:** The IAM role ARN must be registered in the Atlas project via Cloud Provider Access before creating the connection.

Required IAM policy permissions: `kinesis:ListShards`, `kinesis:SubscribeToShard`, `kinesis:PutRecords`, `kinesis:DescribeStreamSummary`.

### AWSLambda
```json
{
  "aws": {
    "roleArn": "arn:aws:iam::123456789:role/streams-lambda-role"
  }
}
```
**Prerequisite:** The IAM role ARN must be registered in the Atlas project via Cloud Provider Access before creating the connection.

### SchemaRegistry
```json
{
  "connectionType": "SchemaRegistry",
  "connectionConfig": {
    "schemaRegistryUrls": ["https://schema-registry.example.com"],
    "schemaRegistryAuthentication": {
      "type": "USER_INFO",
      "username": "...",
      "password": "..."
    }
  }
}
```
- `connectionType` MUST be `"SchemaRegistry"` (not `"Kafka"` or `"Https"`)
- `schemaRegistryUrls` is an **array** (not a string). The tool auto-wraps a string into an array if needed.
- `schemaRegistryAuthentication.type`: `"USER_INFO"` (explicit credentials) or `"SASL_INHERIT"` (inherit from Kafka connection)
- Tool elicitation will collect sensitive fields (password) — don't ask the user for these directly

### Sample
No connectionConfig required. Provides built-in test data. Useful for development and testing without external infrastructure.

Available sample formats: `sample_stream_solar` (default, auto-created when `includeSampleData: true` on workspace), `samplestock`, `sampleweather`, `sampleiot`, `samplelog`, `samplecommerce`.
