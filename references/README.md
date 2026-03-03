# streams-mcp-skills

Claude Code skills for building, operating, and debugging Atlas Stream Processing through the [MongoDB MCP Server](https://github.com/jwongmongodb/mongodb-mcp-server/tree/option-4-intent-tools). Pushing change

## What's included

```
SKILL.md                              # Main skill definition
references/
  connection-configs.md               # Connection config schemas by type
  development-workflow.md             # 5-phase lifecycle, debugging decision trees
  output-diagnostics.md               # Processor output classification, diagnostic workflow
  pipeline-patterns.md                # Stage categories, source/sink/window/enrichment patterns
  sizing-and-parallelism.md           # Tier specs, parallelism formula, cost optimization
```

## Setup Guide

### Prerequisites

- **Node.js** v20+ (`node --version`)
- **pnpm** (`npm install -g pnpm`)
- **Claude Code CLI** (`claude` command available)
- A **MongoDB Atlas** account with:
  - An Organization-level **Service Account** (API key pair)
  - The service account granted **Organization Owner** or **Project Owner** on the target project
  - A payment method configured (Atlas Stream Processing has no free tier)

### Step 1: Clone and build the MCP server

```bash
git clone https://github.com/jwongmongodb/mongodb-mcp-server.git
cd mongodb-mcp-server
git checkout option-4-intent-tools
pnpm install
npm run build
```

Verify the build output exists and note the absolute path:

```bash
echo "$(pwd)/dist/esm/index.js"
```

Save this path for Step 4.

### Step 2: Install the streams skill

```bash
git clone https://github.com/jwongmongodb/streams-mcp-skills.git
mkdir -p ~/.claude/skills/streams-mcp-tools
cp -r streams-mcp-skills/SKILL.md streams-mcp-skills/references ~/.claude/skills/streams-mcp-tools/
```

Verify:

```
~/.claude/skills/streams-mcp-tools/
├── SKILL.md
└── references/
    ├── connection-configs.md
    ├── development-workflow.md
    ├── output-diagnostics.md
    ├── pipeline-patterns.md
    └── sizing-and-parallelism.md
```

### Step 3: Create Atlas Service Account (if you don't have one)

1. Go to **Atlas** > **Organization** > **Access Manager** > **Service Accounts**
2. Click **Create Service Account**
3. Name: anything descriptive (e.g., `mcp-server-dev`)
4. Role: **Organization Owner** (or scope to a specific project with **Project Owner**)
5. Copy the **Client ID** (`mdb_sa_id_...`) and **Client Secret** (`mdb_sa_sk_...`) — the secret is shown only once

### Step 4: Configure `~/.claude.json`

Open (or create) `~/.claude.json`. If the file already exists, merge the `mcpServers` block into the top-level object. If creating fresh:

```json
{
  "mcpServers": {
    "mongodb-local": {
      "command": "node",
      "args": [
        "/ABSOLUTE/PATH/TO/mongodb-mcp-server/dist/esm/index.js"
      ],
      "env": {
        "MDB_MCP_API_BASE_URL": "https://cloud.mongodb.com",
        "MDB_MCP_API_CLIENT_ID": "<YOUR_SERVICE_ACCOUNT_CLIENT_ID>",
        "MDB_MCP_API_CLIENT_SECRET": "<YOUR_SERVICE_ACCOUNT_CLIENT_SECRET>",
        "MDB_MCP_PREVIEW_FEATURES": "streams"
      }
    }
  }
}
```

Replace the placeholders:

| Placeholder | Value |
|-------------|-------|
| `/ABSOLUTE/PATH/TO/mongodb-mcp-server/dist/esm/index.js` | The path from Step 1 |
| `<YOUR_SERVICE_ACCOUNT_CLIENT_ID>` | Client ID from Step 3 (`mdb_sa_id_...`) |
| `<YOUR_SERVICE_ACCOUNT_CLIENT_SECRET>` | Client Secret from Step 3 (`mdb_sa_sk_...`) |

> **Note:** If you use `asdf`, `nvm`, or another version manager, replace `"node"` with the full path from `which node`.

### Step 5: Verify

Start Claude Code from any directory:

```bash
claude
```

Check the MCP server is connected:

```
> /mcp
```

You should see `mongodb-local` listed. Then test:

```
> list my Atlas projects
> show my streams workspaces in project <your-project-name>
```

### Step 6: Test operational logs

With a running or recently failed processor:

```
> why is my processor failing in workspace <workspace-name>?
> show me the operational logs for processor <name> in workspace <workspace>
> show me the audit logs for workspace <workspace>
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `MCP server mongodb-local failed to start` | Check the `node` path is correct. Run `node /path/to/dist/esm/index.js` manually to see errors |
| `401 Unauthorized` | Verify Client ID and Secret. Check service account has correct role |
| Streams tools not showing | Ensure `"MDB_MCP_PREVIEW_FEATURES": "streams"` is in env |
| Skill not activating on streams questions | Verify `~/.claude/skills/streams-mcp-tools/SKILL.md` exists |
| `Cannot find module` on build | Run `pnpm install` then `npm run build` again |
