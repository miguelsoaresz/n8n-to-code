# Node Mapping Reference

Expert knowledge for extracting and converting n8n nodes to code. Focuses on gotchas, typeVersion differences, and non-obvious behavior — not which library to use (Claude already knows that).

## typeVersion Differences — The Silent Killer

The same node type changes its parameter schema across versions. If you parse v2 params with v1 logic, you get wrong or missing data.

### HTTP Request

| Version | Parameter Structure |
|---------|-------------------|
| v1 | `url`, `method`, `body`, `headers` at root level |
| v2 | Same as v1 but adds `authentication` field (replaces credentials) |
| v3 | Restructured: `sendBody`, `bodyParameters.parameters[]`, `sendHeaders`, `headerParameters.parameters[]` |
| v4 (current) | Same as v3 but adds `sendQuery`, `queryParameters`, and `options.redirect`, `options.timeout` |

**Gotcha:** v1-v2 headers are a flat object `{ "Authorization": "Bearer X" }`. v3+ headers are an array `[{ "name": "Authorization", "value": "Bearer X" }]`. Parse wrong structure = silent data loss.

### Postgres / MySQL

| Version | Parameter Structure |
|---------|-------------------|
| v1 | `query` field directly at parameters root. Single operation only. |
| v2 (current) | `operation` field (`executeQuery`, `insert`, `update`, `delete`) + conditional params per operation. `query` only exists when `operation: "executeQuery"`. Insert uses `columns` + `schema` + `table`. |

**Gotcha:** v2 `insert` doesn't have a `query` — it constructs the INSERT from `columns` and `table`. If you look for `parameters.query` on a v2 insert node, you get `undefined` and generate broken code.

### Code Node

| Version | Behavior |
|---------|----------|
| v1 (`function` type) | JS only. Receives `items` array. Must return `items` array. Uses `$item()` helper. |
| v2 (`code` type) | JS or Python. Two modes: "Run Once for All Items" (`$input.all()`) vs "Run Once for Each Item" (`$input.item`). `parameters.mode` determines which. |

**Gotcha:** v2 "Run Once for All Items" means the code processes the entire array — wrap in a single function call. "Run Once for Each Item" means it runs per-item — wrap in a map/loop. Getting this wrong means N function calls instead of 1, or 1 instead of N.

### IF Node

| Version | Condition Structure |
|---------|-------------------|
| v1 | `conditions.string[]`, `conditions.number[]`, `conditions.boolean[]` — separate arrays by type. `combineOperation: "all" | "any"`. |
| v2 (current) | `conditions.options.conditions[]` — unified array with `leftValue`, `rightValue`, `operator` per condition. `options.combineOperation`. Also adds `looseTypeValidation`. |

**Gotcha:** v1 splits conditions by type, v2 unifies them. Parsing v2 with v1 logic misses all conditions. Also, `looseTypeValidation: true` means `"1" == 1` is true — in code you need type coercion.

### Set / Edit Fields

| Version | Behavior |
|---------|----------|
| v1 | `values.string[]`, `values.number[]`, `values.boolean[]` — assigns by type |
| v2 | `fields.values[]` with `name` + `type` + `stringValue`/`numberValue`/etc |
| v3 (current) | `assignments.assignments[]` with `id`, `name`, `value`, `type`. Also has `mode: "manual" | "raw"` — raw mode uses a JSON expression |

**Gotcha:** v3 raw mode means the entire output is a single expression, not field-by-field assignments. Check `parameters.mode` first.

## Merge Node — 5 Completely Different Operations

The Merge node's `parameters.mode` determines its behavior. Each needs different code:

| Mode | joinMode | Code Equivalent | Notes |
|------|----------|----------------|-------|
| `append` | — | `[...input1, ...input2]` | Simple array concat |
| `combine` | `mergeByFields` | SQL-like JOIN on `mergeByFields.values[]` | Check `options.clashHandling`: preferInput1, preferInput2, addSuffix |
| `combine` | `multiplex` | Cartesian product | Every item from input1 paired with every item from input2 |
| `combine` | `chooseBranch` | `output = chosenInput` | `parameters.output` = "input1" or "input2" — just picks one |
| `keepKeyMatches` | — | Filter input1 where key exists in input2 | Like SQL WHERE EXISTS |

**Gotcha:** Most people assume Merge = join. But `chooseBranch` is used as a conditional passthrough (like a ternary), and `multiplex` generates N*M items which can explode data volume.

## continueOnFail — Changes Error Flow Per Node

Any node can have `"continueOnFail": true` in `onError` settings. This means:
- On failure, the node outputs `{ error: { message, description } }` instead of throwing
- Downstream nodes receive the error as data, not as an exception

**In code:** nodes with `continueOnFail` need individual try/catch that returns an error object. Nodes without it should propagate errors normally. Check `node.onError` or `node.settings?.continueOnFail`.

## SplitInBatches — Loop Pattern

SplitInBatches creates a loop in n8n by connecting its "done" output back to itself or to the next processing node. The connections structure shows this as a cycle.

**In code:** This becomes a `for` loop with chunked array:
```
chunks = splitArray(items, batchSize)
for (chunk of chunks) {
  result = await processChunk(chunk)
  allResults.push(...result)
}
```

**Gotcha:** The nodes BETWEEN SplitInBatches' two outputs form the loop body. Trace the connections from output[0] (loop body) back to SplitInBatches input to find all nodes in the loop. Output[1] ("done") is the exit.

## Wait Node — Not Just Sleep

| Resume Type | Code Equivalent |
|-------------|----------------|
| `afterTimeInterval` | `await sleep(ms)` — simple |
| `specificTime` | Scheduled job at specific datetime |
| `webhook` | Pause execution, create a temporary webhook endpoint, resume when called. Returns `$execution.resumeUrl` to upstream nodes. |

**Gotcha:** `webhook` resume type means the workflow pauses mid-execution and waits for an external HTTP call. In code, this needs a queue/job system — not just async/await. The `$execution.resumeUrl` expression referenced in upstream nodes is a URL that resumes this specific execution.

## Connection Semantics

```json
{
  "Node A": {
    "main": [
      [{ "node": "B", "type": "main", "index": 0 }],
      [{ "node": "C", "type": "main", "index": 0 }]
    ]
  }
}
```

- `main[0]` = first output. For IF: true branch. For all others: default/success output.
- `main[1]` = second output. For IF: false branch. For Switch: second case. Doesn't exist for most nodes.
- Multiple entries in SAME output array (e.g., `main[0]` has B and C) = both run in parallel with the same input.
- `index` on target = which input slot of the target. Only relevant for Merge nodes (input 0 vs input 1).

**Gotcha:** A node with NO outgoing connections in `connections` is a terminal node — its output is discarded. Don't generate a return value for it unless it's a Respond to Webhook node.

## Credential Type Detection

The `credentials` object in a node maps credential type to reference:

```json
"credentials": {
  "postgresApi": { "id": "1", "name": "My DB" },
  "oAuth2Api": { "id": "2", "name": "Google Auth" }
}
```

| Credential Type Pattern | Auth Strategy in Code |
|------------------------|----------------------|
| `*Api` (e.g., `postgresApi`, `slackApi`) | Connection string or API key → env var |
| `*OAuth2Api` (e.g., `googleOAuth2Api`) | OAuth2 flow → needs token refresh logic |
| `httpBasicAuth` | Basic auth header → username:password env vars |
| `httpHeaderAuth` | Custom header → env var for header value |
| `httpQueryAuth` | Query parameter auth → env var for param value |
| `httpDigestAuth` | Digest auth → library-specific implementation |

**Gotcha:** OAuth2 credentials cannot be mapped to a single env var. They need `client_id`, `client_secret`, `access_token`, `refresh_token`, and a refresh mechanism. Flag these in the analysis as requiring additional setup.
