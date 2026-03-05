---
name: streams-mcp-tools
description: Build, operate, and debug Atlas Stream Processing through the MongoDB MCP Server. Use when user says "set up a Kafka pipeline", "deploy a stream processor", "add a connection to my workspace", "why is my processor failing", "stop my processor", "delete my workspace", "show me my Streams workspaces", or any task involving Atlas Stream Processing workspaces, connections, or processors. Do NOT use for general MongoDB queries, Atlas cluster management, or non-Streams Atlas operations.
version: 3.4.0
user-invocable: true
---

# Atlas Stream Processing — MCP Tool Skill

## Prerequisites

This skill requires the **MongoDB MCP Server** connected with:
- Atlas API credentials (`apiClientId` and `apiClientSecret`)
- `previewFeatures: ["streams"]` enabled in the MCP server config

The 4 tools: `atlas-streams-discover`, `atlas-streams-build`, `atlas-streams-manage`, `atlas-streams-teardown`.

## CRITICAL: Consult Official Examples Before Creating Processors

**IMPORTANT: Before creating any processor, consult the official MongoDB ASP examples repo for best practices and valid patterns:**

**https://github.com/mongodb/ASP_example**

This repo is continuously updated with high-value customer-driven examples and contains 33+ end-to-end processors, 6 quickstarts, and 15 code snippets maintained by MongoDB engineering. Use it as the authoritative source for:
- Valid pipeline stage combinations and ordering
- Correct `$source` and `$merge`/`$emit` syntax per connection type
- Windowing patterns (tumbling, hopping, session)
- Advanced patterns (joins, `$https` enrichment, `$externalFunction` for Lambda, `$validate` schema validation, `$function` JavaScript UDFs, dynamic routing, chained processors)

Key quickstart references:
| Quickstart | Pattern |
|-----------|---------|
| `00_hello_world.json` | Inline `$source.documents` with `$match` (zero infra, ephemeral) |
| `01_changestream_basic.json` | Change stream → tumbling window → `$merge` to Atlas |
| `02_changestream_to_kafka.json` | Change stream → `$emit` to Kafka topic |
| `03_kafka_to_mongo.json` | Kafka source → tumbling window rollup → `$merge` to Atlas |
| `04_mongo_to_mongo.json` | Change stream → transform → `$merge` (archive pattern) |
| `05_kafka_tail.json` | Kafka source with no sink (ephemeral `tail -f`) |

## Pipeline Warnings — Invalid Constructs

These MongoDB aggregation features are **NOT valid** in streaming pipelines:
- **`$$NOW`** — not available in stream processing context
- **`$$ROOT`** — not available in stream processing context
- **`$$CURRENT`** — not available in stream processing context
- **HTTPS connections as `$source`** — HTTPS connections are for `$https` enrichment only, not as data sources
- **Pipelines without a sink** — `$merge` or `$emit` is required for persistent (deployed) processors. Sinkless pipelines only work ephemerally via `sp.process()` in mongosh
- **Kafka `$source` without `topic`** — Kafka sources MUST include a `topic` field
- **Lambda connections as `$emit` target** — Lambda uses `$externalFunction` (mid-pipeline stage), not `$emit`

## Instructions

You are helping a user interact with Atlas Stream Processing through the MongoDB MCP Server. This skill teaches you which tools to call, what fields to fill, and how to sequence multi-step workflows.

### Step 1: Select the right tool

| Tool | When to use |
|------|-------------|
| `atlas-streams-discover` | **See, inspect, or diagnose** — "list workspaces", "show processor stats", "why is it failing?" |
| `atlas-streams-build` | **Create** — "set up a workspace", "add a Kafka connection", "deploy a processor" |
| `atlas-streams-manage` | **Change state or config** — "start/stop processor", "change pipeline", "update credentials" |
| `atlas-streams-teardown` | **Delete** — "delete workspace", "remove connection", "clean up" |

Intent mapping:
- **"What do I have?" / "Show me" / "List" / "Status" / "Why failing?"** → `atlas-streams-discover`
- **"Create" / "Set up" / "Add" / "Deploy" / "Connect"** → `atlas-streams-build`
- **"Start" / "Stop" / "Restart" / "Change" / "Modify" / "Update"** → `atlas-streams-manage`
- **"Delete" / "Remove" / "Tear down" / "Clean up"** → `atlas-streams-teardown`

When in doubt, call `atlas-streams-discover` first to understand current state.

Do NOT use these tools for general MongoDB queries (`find`/`aggregate`), Atlas cluster management (`atlas-list-clusters`), or non-Streams operations.

### Step 2: Fill the right fields

**Every tool call requires `projectId`.** If unknown, call `atlas-list-projects` first.

#### atlas-streams-build field mapping

CRITICAL: This tool uses a `resource` enum. **Only fill fields for the selected resource type.**

**resource = "workspace":**
Fill: `projectId`, `workspaceName`, `cloudProvider`, `region`, `tier`, `includeSampleData`
Leave empty: all connection and processor fields

**CRITICAL — Region naming:** The `region` field uses Atlas-specific names that differ by cloud provider. Using the wrong format returns a cryptic `dataProcessRegion` error. Reference table:

| Provider | Cloud Region | Atlas `region` Value |
|----------|-------------|---------------------|
| **AWS** | us-east-1 | `VIRGINIA_USA` |
| **AWS** | us-east-2 | `US_EAST_2` |
| **AWS** | us-west-2 | `OREGON_USA` |
| **AWS** | ca-central-1 | `CA_CENTRAL_1` |
| **AWS** | sa-east-1 | `SA_EAST_1` |
| **AWS** | eu-west-1 | `IRELAND_IRL` |
| **GCP** | us-central1 | `US_CENTRAL1` |
| **GCP** | europe-west1 | `WESTERN_EUROPE` |
| **Azure** | eastus | `US_EAST_1` |
| **Azure** | eastus2 | `US_EAST_2` |
| **Azure** | westus | `US_WEST` |
| **Azure** | westeurope | `EUROPE_WEST` |

**If the region format is unknown:** Inspect an existing workspace with `atlas-streams-discover` → `action: "inspect-workspace"` and check the `dataProcessRegion.region` field for the correct format.

**resource = "connection":**
Fill: `projectId`, `workspaceName`, `connectionName`, `connectionType`, `connectionConfig`
Leave empty: all workspace and processor fields
(See [references/connection-configs.md](references/connection-configs.md) for type-specific schemas)

**Connection Capabilities — Source/Sink Reference:**

Know what each connection type can do before creating pipelines:

| Connection Type | As Source ($source) | As Sink ($merge / $emit) | Mid-Pipeline | Notes |
|-----------------|---------------------|--------------------------|--------------|-------|
| **Cluster** | ✅ Change streams | ✅ $merge to collections | ✅ $lookup | Change streams monitor insert/update/delete/replace operations |
| **Kafka** | ✅ Topic consumer | ✅ $emit to topics | ❌ | Source MUST include `topic` field |
| **Sample Stream** | ✅ Sample data | ❌ Not valid | ❌ | Testing/demo only |
| **S3** | ❌ Not valid | ✅ $emit to buckets | ❌ | Sink only - use `path`, `format`, `compression` |
| **Https** | ❌ Not valid | ✅ $https as sink | ✅ $https enrichment | Can be used mid-pipeline for enrichment OR as final sink stage |
| **AWSLambda** | ❌ Not valid | ✅ $externalFunction (async only) | ✅ $externalFunction (sync or async) | **Sink:** `execution: "async"` required. **Mid-pipeline:** `execution: "sync"` or `"async"` |
| **AWS Kinesis** | ✅ Stream consumer | ✅ $emit to streams | ❌ | Similar to Kafka pattern |
| **SchemaRegistry** | ❌ Not valid | ❌ Not valid | ✅ Schema resolution | **Metadata only** - used by Kafka connections for Avro schemas |

**Common connection usage mistakes to avoid:**
- ❌ Using HTTPS connections as `$source` → HTTPS is for enrichment or sink only
- ❌ Using `$externalFunction` as sink with `execution: "sync"` → Must use `execution: "async"` for sink stage
- ❌ Forgetting change streams exist → Atlas Cluster is a powerful source, not just a sink
- ❌ Using `$merge` with Kafka → Use `$emit` for Kafka sinks

**$externalFunction execution modes:**
- **Mid-pipeline:** Can use `execution: "sync"` (blocks until Lambda returns) or `execution: "async"` (non-blocking)
- **Final sink stage:** MUST use `execution: "async"` only

**resource = "processor":**
Fill: `projectId`, `workspaceName`, `processorName`, `pipeline`, `dlq` (recommended), `autoStart` (optional)
Leave empty: all workspace and connection fields
(See [references/pipeline-patterns.md](references/pipeline-patterns.md) for pipeline examples)

**Before creating a processor — REQUIRED validation steps:**

You MUST call `search-knowledge` before composing any processor pipeline. This is not optional — the reference files in this skill cover common patterns, but `search-knowledge` is the authoritative source for exact field schemas and catches field name errors that skill references alone may not prevent.

1. **`search-knowledge` (REQUIRED)** — call with a query describing the sink/source type, e.g. "Atlas Stream Processing $emit S3 fields" or "Atlas Stream Processing Kafka $source configuration". Do this even if you think you know the answer from skill references. This call validates field names and catches errors like using `prefix` instead of `path` for S3 `$emit`.
2. **ASP_example repo (recommended for complex pipelines)** — for end-to-end pipeline patterns, fetch the relevant quickstart or example from `https://raw.githubusercontent.com/mongodb/ASP_example/main/` using `WebFetch`. Key files: `quickstarts/` (6 quickstarts), `example_processors/` (15+ patterns), `code_snippets/` (reusable stages).

The skill's reference files provide a starting point, but always cross-check with `search-knowledge` before calling `atlas-streams-build` with `resource: "processor"`.

**resource = "privatelink":**
Fill: `projectId`, `workspaceName`, `privateLinkProvider`, `privateLinkConfig`
Leave empty: all connection and processor fields

#### atlas-streams-discover notes

- `action: "list-workspaces"` — list all workspaces in a project
- `action: "inspect-workspace"` — details on a specific workspace
- `action: "list-connections"` / `"inspect-connection"` — connections in a workspace
- `action: "list-processors"` / `"inspect-processor"` — processors in a workspace
- `action: "diagnose-processor"` — combined health report (state, stats, connection health, errors, actionable recommendations)
- `action: "get-logs"` — operational logs (default) or audit logs. Use `logType: "operational"` for runtime errors (Kafka failures, schema issues, OOM). Use `logType: "audit"` for lifecycle events (start/stop). Optionally filter by `resourceName` (processor name).
- `action: "get-networking"` — PrivateLink/VPC peering. Optionally provide `cloudProvider` and `region` for account details.

**Pagination** (all list actions): `limit` (1-100, default 20), `pageNum` (default 1).
**Response format**: `responseFormat` — `"concise"` (default for list actions: names/states only) or `"detailed"` (default for inspect/diagnose: full config).

#### atlas-streams-manage field mapping

Always fill: `projectId`, `workspaceName`. Then by action:

- `"start-processor"` → `resourceName`. Optional: `tier`, `resumeFromCheckpoint`, `startAtOperationTime`
- `"stop-processor"` → `resourceName`
- `"modify-processor"` → `resourceName`. At least one of: `pipeline`, `dlq`, `newName`
- `"update-workspace"` → `newRegion` or `newTier`
- `"update-connection"` → `resourceName`, `connectionConfig`. Works for updating credentials, bootstrap servers, and other config. **Exception: networking config (e.g., PrivateLink) cannot be modified after creation** — to change networking, delete and recreate the connection.
- `"accept-peering"` → `peeringId`, `requesterAccountId`, `requesterVpcId`
- `"reject-peering"` → `peeringId`

#### atlas-streams-teardown field mapping

Always fill: `projectId`, `resource`. Then:

- `resource: "workspace"` → `workspaceName`
- `resource: "connection"` or `"processor"` → `workspaceName`, `resourceName`
- `resource: "privatelink"` or `"peering"` → `resourceName` (the ID)

### Step 3: Sequence multi-step workflows

**Setup from scratch:**
1. `atlas-streams-build` → `resource: "workspace"` (cloud, region, tier)
2. `atlas-streams-build` → `resource: "connection"` (one call per connection)
3. `atlas-streams-build` → `resource: "processor"` (reference connections by name in pipeline)
4. Set `autoStart: true` in step 3, or call `atlas-streams-manage` → `action: "start-processor"`

**Incremental pipeline development (recommended):**
See [references/development-workflow.md](references/development-workflow.md) for the full 5-phase lifecycle.
1. Start with basic `$source` → `$merge` pipeline (validate connectivity)
2. Add `$match` stages (validate filtering)
3. Add `$addFields` / `$project` transforms (validate reshaping)
4. Add windowing or enrichment (validate aggregation logic)
5. Add error handling / DLQ configuration

**Modify a processor pipeline:**
1. `atlas-streams-manage` → `action: "stop-processor"` — **processor MUST be stopped first**
2. `atlas-streams-manage` → `action: "modify-processor"` — provide new pipeline
3. `atlas-streams-manage` → `action: "start-processor"` — restart

**Debug a failing processor:**
See [references/output-diagnostics.md](references/output-diagnostics.md) for the full decision framework.
1. `atlas-streams-discover` → `action: "diagnose-processor"` — one-shot health report. Always call this first.
2. `atlas-streams-discover` → `action: "get-logs"` (defaults to `logType: "operational"`) — runtime errors, Kafka failures, schema issues, OOM messages. Filter by `resourceName` for a specific processor. Always call this second.
3. **Commit to a specific root cause.** After reviewing the diagnose output and logs, identify THE primary issue — do not present a list of hypothetical scenarios. The diagnostic data will contain specific error codes, state transitions, and stats that point to one root cause. Common patterns:
   - **Error 419 + "no partitions found"** → Kafka topic doesn't exist or is misspelled
   - **State: FAILED + multiple restarts** → connection-level error (bypasses DLQ), check logs for the repeated error
   - **State: STARTED + zero output + windowed pipeline** → likely idle Kafka partitions blocking window closure; check for missing `partitionIdleTimeout`
   - **State: STARTED + zero output + non-windowed** → check if source has data; inspect Kafka offset lag
   - **High memoryUsageBytes approaching tier limit** → OOM risk; recommend higher tier
   - **DLQ count increasing** → per-document processing errors; use MongoDB `find` on DLQ collection
4. Classify processor type before interpreting output volume:
   - **Alert/anomaly processors**: low or zero output is NORMAL and healthy
   - **Data transformation processors**: low output is a RED FLAG
   - **Filter processors**: variable output depending on data match rate
5. Provide concrete, ordered fix steps specific to the diagnosed root cause (e.g., "stop → modify pipeline to add partitionIdleTimeout → restart with resumeFromCheckpoint: false").
6. If lifecycle event history needed → `atlas-streams-discover` → `action: "get-logs"`, `logType: "audit"` — shows start/stop events

**Tear down:**
Delete workspace directly (removes all contained resources), or individually: delete processors (auto-stops if running) → delete connections (fails if referenced by running processors) → delete workspace.

**Chained processors:**
Multiple processors can be chained: processor A writes to an Atlas collection via `$merge`, processor B reads from that collection via change stream `$source`. This enables multi-stage processing pipelines.

## MCP Tool Behaviors

These are built-in behaviors of the MCP tools — do not duplicate this logic manually.

**Connection creation — elicitation:** When creating a connection, the build tool auto-collects missing sensitive fields (passwords, bootstrap servers, usernames) via an interactive form using the MCP elicitation protocol. Do NOT ask the user for these fields yourself — let the tool elicit them.

**Connection creation — auto-normalization:**
- `bootstrapServers` array → auto-converted to comma-separated string
- `schemaRegistryUrls` string → auto-wrapped in array
- `dbRoleToExecute` → auto-defaults to `{role: "readWriteAnyDatabase", type: "BUILT_IN"}` for Cluster connections

**Workspace creation — sample data:** `includeSampleData` defaults to `true`, which auto-creates the `sample_stream_solar` connection via a special API endpoint.

**State pre-checks (manage tool):**
- `start-processor` → errors if processor is already STARTED
- `stop-processor` → no-ops if already STOPPED or CREATED (not an error)
- `modify-processor` → errors if processor is STARTED (must stop first)

**Teardown safety checks:**
- **Processor deletion** → auto-stops the processor before deleting (no need to stop manually first)
- **Connection deletion** → scans all processor pipelines for references; **blocks deletion** if any running processor uses the connection. Stop/delete referencing processors first.
- **Workspace deletion** → counts connections and processors, reports the full impact before deleting

## Pre-Deploy Quality Checklist

Before creating a processor, verify:
- [ ] `search-knowledge` was called to validate sink/source field names
- [ ] Pipeline starts with `$source` and ends with `$merge` or `$emit`
- [ ] No `$$NOW`, `$$ROOT`, or `$$CURRENT` in the pipeline
- [ ] Kafka `$source` includes a `topic` field
- [ ] Kafka `$source` with windowed pipeline includes `partitionIdleTimeout` (prevents windows from stalling on idle partitions)
- [ ] HTTPS connections are only used in `$https` enrichment stages, not in `$source`
- [ ] All `connectionName` references match actual connections in the workspace (use `discover` → `list-connections` to verify)
- [ ] DLQ is configured (recommended for production)
- [ ] `$https` stages use `onError: "dlq"` (not `"fail"`)
- [ ] `$externalFunction` stages use `onError: "dlq"` and `execution` is explicitly set
- [ ] API auth is stored in connection settings, not hardcoded in the pipeline

## Post-Deploy Verification Workflow

After creating and starting a processor:
1. `atlas-streams-discover` → `action: "inspect-processor"` — confirm state is STARTED
2. `atlas-streams-discover` → `action: "diagnose-processor"` — check for errors in the health report
3. Use MongoDB `count` tool on the DLQ collection — verify no errors accumulating
4. Use MongoDB `find` tool on the output collection — verify documents are arriving
5. If output is low/zero, classify processor type before assuming a problem (see Debug section)

## Tier Sizing & Performance

See [references/sizing-and-parallelism.md](references/sizing-and-parallelism.md) for the complete guide including complexity scoring, worked examples, and cost optimization.

### Tier Reference

| Tier | vCPU | RAM | Bandwidth | Max Parallelism | Kafka Partitions | Use case |
|------|------|-----|-----------|-----------------|------------------|----------|
| SP2  | 0.25 | 512MB | 50 Mbps | 1 | 32 | Minimal filtering, testing |
| SP5  | 0.5 | 1GB | 125 Mbps | 2 | 64 | Simple filtering and routing |
| SP10 | 1 | 2GB | 200 Mbps | 8 | Unlimited | Moderate workloads, joins, grouping |
| SP30 | 2 | 8GB | 750 Mbps | 16 | Unlimited | Windows, JavaScript UDFs, production |
| SP50 | 8 | 32GB | 2500 Mbps | 64 | Unlimited | High throughput, large window state |

### Sizing Rules
- Stream Processing reserves **20% memory for overhead** — user processes are limited to 80%
- Monitor `memoryUsageBytes` via processor stats to determine proper tier
- If memory usage exceeds 80% of tier capacity, processor fails with OOM
- Use `parallelism` setting on `$merge`, `$lookup`, `$https` for concurrent I/O operations

**Parallelism formula:** `minimum tier = sum of (parallelism - 1) for all stages where parallelism > 1`. Example: a pipeline with `$lookup` at parallelism 3 and `$merge` at parallelism 4 needs `(3-1) + (4-1) = 5` excess parallelism → requires SP10 (max 8).

### Performance Best Practices
- Place `$match` stages as early as possible to reduce downstream volume
- Place `$https` enrichment calls downstream of window stages to batch and reduce API call frequency
- Use `partitionIdleTimeout` in Kafka `$source` to unblock windows when partitions go idle
- Use descriptive processor names indicating their function (e.g., `celsius-converter`, `fraud-detector`)

## Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| 404 on workspace | Doesn't exist or misspelled | `discover` → `list-workspaces` |
| 409 on create | Name already exists | Inspect existing resource or pick new name |
| 402 on create | No billing configured | Add payment method in Atlas → Billing. Use `sp.process()` in mongosh as free alternative |
| "processor must be stopped" | Tried to modify running processor | `manage` → `stop-processor` first |
| Processor FAILED | Pipeline error, connection failure, or OOM | `discover` → `diagnose-processor` |
| bootstrapServers format | Passed as array instead of string | Use comma-separated string: `"broker1:9092,broker2:9092"` |
| "must choose at least one role" | Cluster connection without `dbRoleToExecute` | Defaults to `readWriteAnyDatabase` — or specify custom role |
| "No cluster named X" | Cluster doesn't exist in project | `atlas-list-clusters` to verify |
| IAM role ARN not found | ARN not registered in project | Register via Atlas → Cloud Provider Access |
| dataProcessRegion format | Wrong region format | See region reference table in Step 2 workspace section. AWS uses names like `VIRGINIA_USA`, GCP uses `US_CENTRAL1`, Azure uses `US_EAST_1`. If unsure, inspect an existing workspace to see the correct format. |
| Low/zero processor output | May be normal for alert-type processors | Classify processor type before assuming a problem |
| Windowed processor "stuck" | Idle Kafka partitions blocking window closure | Add `partitionIdleTimeout` to Kafka `$source` (e.g., `{"size": 30, "unit": "second"}`) |
| Processor PROVISIONING for minutes | Restart cycle with exponential backoff | Wait for FAILED state, or stop → restart. Check logs for repeated error |
| `$$NOW` / `$$ROOT` / `$$CURRENT` in pipeline | Invalid in streaming context | Remove these system variables; use alternative approaches |

## Billing & Cost Awareness

**Atlas Stream Processing has no free tier.** All deployed processors incur charges while running. You MUST surface this proactively — do not silently start a processor without the user understanding cost implications.

### Before creating or starting a processor

1. **Confirm billing is set up.** Ask the user if they have a payment method on their Atlas account. If unsure, recommend they verify in Atlas → Organization → Billing before proceeding.
2. **Warn about ongoing costs.** A running processor bills continuously, calculated per-second. `start-processor` begins billing, `stop-processor` stops it. Suggest stopping processors when not actively needed.
3. **If no billing or user wants to avoid charges:** Recommend `sp.process()` in mongosh as an ephemeral alternative. This runs a pipeline ad-hoc without deploying a named processor — no billing method required, no persistent cost. Ideal for prototyping and validating pipelines before committing to a deployed processor.

### If you receive a 402 error

Do NOT retry. Instead:
1. Explain that Atlas Stream Processing requires an active payment method
2. Direct the user to Atlas → Organization → Billing to add a credit card
3. Offer `sp.process()` in mongosh as a no-cost way to test their pipeline in the meantime

## Safety Rules

- `atlas-streams-teardown` and `atlas-streams-manage` require user confirmation — do not bypass
- Deleting a workspace removes ALL connections and processors permanently
- Processors must be STOPPED before modifying their pipeline
- After stopping, state is preserved 45 days — then checkpoints are discarded
- `resumeFromCheckpoint: false` drops all window state — warn user first
- Moving processors between workspaces is not supported (must recreate)
- Dry-run / simulation is not supported — explain what you would do and ask for confirmation
- Always warn users about billing before starting processors
- Store API authentication credentials in connection settings, never hardcode in processor pipelines

## Additional References

### Internal Reference Files
- [references/pipeline-patterns.md](references/pipeline-patterns.md) — Stage categories, source/sink/window/enrichment patterns, full pipeline examples
- [references/connection-configs.md](references/connection-configs.md) — Connection config schemas by type, auth patterns, elicitation behavior
- [references/development-workflow.md](references/development-workflow.md) — 5-phase lifecycle, debugging decision trees, monitoring cadence
- [references/output-diagnostics.md](references/output-diagnostics.md) — Processor output classification, red/green flag framework, diagnostic workflow
- [references/sizing-and-parallelism.md](references/sizing-and-parallelism.md) — Tier hardware specs, parallelism formula, complexity scoring, cost optimization

### External Resources
- Official ASP examples (33+ processors, continuously updated): https://github.com/mongodb/ASP_example
- ASP Claude plugin (tier sizing, development workflows, CI/CD): https://github.com/kgorman/asp_claude
- Atlas Stream Processing billing: https://www.mongodb.com/docs/atlas/billing/stream-processing-costs/
