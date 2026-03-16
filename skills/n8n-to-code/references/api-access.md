# n8n API Access Reference

How to interact with the n8n REST API to fetch workflows programmatically.

## Authentication

n8n uses API keys for authentication. Users generate keys in: **Settings → API → Create API Key**.

Header format:
```
X-N8N-API-KEY: <api-key>
```

## Endpoints

### List All Workflows

```
GET {baseUrl}/api/v1/workflows
```

**Response:**
```json
{
  "data": [
    {
      "id": "1",
      "name": "My Workflow",
      "active": true,
      "createdAt": "2024-01-15T10:30:00.000Z",
      "updatedAt": "2024-03-10T14:22:00.000Z",
      "tags": [
        { "id": "1", "name": "production" }
      ]
    }
  ],
  "nextCursor": null
}
```

**Pagination:** If `nextCursor` is not null, pass it as `?cursor={value}` to get the next page.

**Useful fields for listing:**
- `id` — needed to fetch the full workflow
- `name` — display to user
- `active` — whether the workflow is currently enabled
- `updatedAt` — last modification date
- `tags` — for grouping/filtering

### Get Specific Workflow

```
GET {baseUrl}/api/v1/workflows/{id}
```

**Response:**
```json
{
  "id": "1",
  "name": "My Workflow",
  "active": true,
  "nodes": [...],
  "connections": {...},
  "settings": {...},
  "staticData": null,
  "tags": [...],
  "createdAt": "2024-01-15T10:30:00.000Z",
  "updatedAt": "2024-03-10T14:22:00.000Z"
}
```

This is the same structure as an exported workflow JSON file — the `nodes` and `connections` fields are identical.

## Workflow JSON Structure

### Top Level

```json
{
  "name": "Workflow Name",
  "nodes": [],
  "connections": {},
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "callerPolicy": "workflowsFromSameOwner"
  },
  "staticData": null
}
```

### Node Structure

```json
{
  "id": "uuid-here",
  "name": "Display Name",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [580, 300],
  "parameters": {
    "method": "POST",
    "url": "https://api.example.com/data",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "Bearer {{ $env.API_KEY }}" }
      ]
    },
    "sendBody": true,
    "bodyParameters": {
      "parameters": [
        { "name": "user_id", "value": "={{ $json.userId }}" }
      ]
    }
  },
  "credentials": {
    "httpBasicAuth": {
      "id": "5",
      "name": "My API Auth"
    }
  }
}
```

### Connections Structure

```json
{
  "connections": {
    "Source Node Name": {
      "main": [
        [
          { "node": "Target Node 1", "type": "main", "index": 0 },
          { "node": "Target Node 2", "type": "main", "index": 0 }
        ],
        [
          { "node": "False Branch Node", "type": "main", "index": 0 }
        ]
      ]
    }
  }
}
```

- Each key in `connections` is a source node name
- `main[0]` = first output (true/default)
- `main[1]` = second output (false for IF nodes)
- Multiple entries in the same array = parallel connections
- `index` on the target = which input of the target node (usually 0)

### Common Node Types Reference

| Type String | Display Name |
|-------------|-------------|
| `n8n-nodes-base.webhook` | Webhook |
| `n8n-nodes-base.httpRequest` | HTTP Request |
| `n8n-nodes-base.code` | Code |
| `n8n-nodes-base.if` | IF |
| `n8n-nodes-base.switch` | Switch |
| `n8n-nodes-base.set` | Edit Fields (Set) |
| `n8n-nodes-base.merge` | Merge |
| `n8n-nodes-base.postgres` | Postgres |
| `n8n-nodes-base.redis` | Redis |
| `n8n-nodes-base.mongoDb` | MongoDB |
| `n8n-nodes-base.slack` | Slack |
| `n8n-nodes-base.gmail` | Gmail |
| `n8n-nodes-base.googleSheets` | Google Sheets |
| `n8n-nodes-base.airtable` | Airtable |
| `n8n-nodes-base.stripe` | Stripe |
| `n8n-nodes-base.github` | GitHub |
| `n8n-nodes-base.jira` | Jira |
| `n8n-nodes-base.notion` | Notion |
| `n8n-nodes-base.executeWorkflow` | Execute Workflow |
| `n8n-nodes-base.scheduleTrigger` | Schedule Trigger |
| `n8n-nodes-base.respondToWebhook` | Respond to Webhook |
| `n8n-nodes-base.splitInBatches` | Split In Batches |

## Error Handling

| Status | Meaning | Action |
|--------|---------|--------|
| 401 | Invalid or missing API key | Ask user to verify key |
| 404 | Workflow not found | Verify ID, list workflows first |
| 403 | Insufficient permissions | Key may not have workflow read access |
| 500 | n8n server error | Suggest JSON export as fallback |
| Timeout | Instance unreachable | Check URL, suggest JSON export |

Always suggest manual JSON export as a fallback if API access fails.
