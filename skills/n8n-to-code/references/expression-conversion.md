# Expression Conversion Reference

Expert knowledge for converting n8n expressions to code. Focuses on edge cases and runtime context — not basic syntax conversion (Claude already knows `toUpperCase()` → `.upper()`).

## Expression vs Template — Two Different Syntaxes

n8n JSON uses TWO syntaxes that look similar but behave differently:

| In JSON | Meaning | Example |
|---------|---------|---------|
| `"={{ $json.field }}"` | Expression (starts with `=`) — evaluated by n8n's expression engine | `"={{ $json.price * 1.1 }}"` |
| `"{{ $json.field }}"` | Template string — interpolated into text | `"Hello {{ $json.name }}"` |
| `"static value"` | Plain string — no evaluation | `"https://api.example.com"` |

**Gotcha:** In the workflow JSON, the `=` prefix distinguishes a computed expression from a template. `"={{ $json.items.length }}"` returns a number. `"{{ $json.items.length }}"` returns a string containing the number. Check for the `=` prefix to determine whether the result should be typed or always a string.

## Runtime Context Variables — Don't Exist in Code

These n8n globals have no direct equivalent outside n8n. Each needs explicit handling:

### $input — Current Node's Input

| Expression | What It Means | Code Equivalent |
|-----------|---------------|-----------------|
| `$input.item` | Current item in per-item mode | Loop variable / function parameter |
| `$input.all()` | All items as array | The full input array |
| `$input.first()` | First item only | `input[0]` |
| `$input.last()` | Last item only | `input[input.length - 1]` |
| `$input.item.json` | Current item's data | Loop variable's data property |
| `$input.item.binary` | Current item's binary data | Separate binary handling (file buffer) |

**Gotcha:** `$input.all()` in a Code node means the code processes ALL items at once. `$input.item` means it runs per-item. The node's `parameters.mode` ("runOnceForAllItems" vs "runOnceForEachItem") determines which is valid. Using `$input.all()` in per-item mode still works in n8n (returns all items) but is semantically confusing — in code, pass the full array explicitly.

### $node — Cross-Node References

| Expression | What It Means | Code Equivalent |
|-----------|---------------|-----------------|
| `$('Node Name').item.json.field` | Output of named node (new syntax) | Variable from that node's return value |
| `$node["Node Name"].json.field` | Same (legacy syntax) | Same |
| `$('Node Name').all()` | All output items from that node | Variable holding the full array |
| `$('Node Name').first()` | First output item | `nodeOutput[0]` |
| `$('Node Name').params.field` | Node's parameter value | Hardcoded config value |

**Gotcha:** `$('Node Name').item` only works in per-item mode and refers to the item at the SAME INDEX. If node A outputs 5 items, and node B processes item #3, `$('A').item` refers to A's item #3. In code, this means you need to pass the index through: `processB(inputItem, aOutput[index])`.

### $prevNode — Dynamic Reference

| Expression | What It Means |
|-----------|---------------|
| `$prevNode.name` | Name of the node that triggered this one |
| `$prevNode.outputIndex` | Which output slot was used (0 = true, 1 = false for IF) |
| `$prevNode.runIndex` | Which run (relevant in loops) |

**In code:** `$prevNode` is used for dynamic routing — "do X if I was called from node A, do Y if from node B." Convert to an explicit parameter: `function process(input, source: string)`. The `outputIndex` is useful for knowing which branch of an IF led here.

### $execution — Execution Metadata

| Expression | What It Means | Code Equivalent |
|-----------|---------------|-----------------|
| `$execution.id` | Unique execution ID | Generate UUID at flow start, pass through |
| `$execution.mode` | "manual" or "production" | Config flag or env var |
| `$execution.resumeUrl` | URL to resume a paused execution (Wait node) | Only exists with Wait webhook — needs job queue URL |

**Gotcha:** `$execution.resumeUrl` is the hardest to convert. In n8n, it's a magic URL that resumes the workflow at the Wait node. In code, you need: (1) a job queue that can pause/resume, (2) a webhook endpoint that triggers resume, (3) the URL generated before the pause is sent to upstream nodes. Flag any workflow using `$execution.resumeUrl` as requiring significant architectural decisions.

### $runIndex and $itemIndex — Loop Context

| Expression | What It Means | Code Equivalent |
|-----------|---------------|-----------------|
| `$runIndex` | Current iteration of SplitInBatches loop | Outer loop counter |
| `$itemIndex` | Current item's index within the batch | Inner loop counter |

**Gotcha:** `$runIndex` resets to 0 each execution. `$itemIndex` resets per batch. In a SplitInBatches with batchSize=10 processing 25 items: run 0 has itemIndex 0-9, run 1 has 0-9, run 2 has 0-4. If the code uses `$runIndex * batchSize + $itemIndex` to get absolute position, you need to replicate this math.

### $env — Environment Variables

`$env.VAR_NAME` → straightforward env var access. But note:
- In n8n, `$env` only exposes vars explicitly allowed in n8n's settings (`N8N_BLOCK_ENV_ACCESS_IN_NODE`)
- Some workflows use `$env` for secrets (API keys) and some for config (base URLs)
- In the analysis, list ALL `$env` references and ask the user to categorize: secret vs config

## Nested Expressions in additionalFields

Many nodes have `additionalFields` or `options` objects that contain expressions at arbitrary depth:

```json
"parameters": {
  "operation": "executeQuery",
  "query": "SELECT * FROM users",
  "additionalFields": {
    "queryTimeout": "={{ $json.timeoutMs }}",
    "schema": "public"
  },
  "options": {
    "batching": {
      "batch": {
        "batchSize": "={{ $json.batchConfig.size }}"
      }
    }
  }
}
```

**Extraction strategy:** Recursively walk ALL values in `parameters`. Any string starting with `=` or containing `{{ }}` is an expression. Don't just check top-level fields — expressions hide in nested objects and arrays.

## Multi-Expression Strings

n8n allows mixing static text and multiple expressions:

```
"Hello {{ $json.firstName }} {{ $json.lastName }}, your order #{{ $json.orderId }} totals {{ $json.total }}"
```

**Conversion strategy:**
1. Split on `{{ }}` boundaries
2. Each `{{ }}` block becomes an interpolated variable
3. Static text stays as-is
4. Use the target language's string interpolation (template literals, f-strings, fmt.Sprintf, etc.)

**Gotcha:** Expressions inside templates can contain complex JS: `{{ $json.items.filter(i => i.active).length }}`. This inline JS needs to be extracted into a helper variable — don't try to inline a `.filter().length` into a Python f-string.

## The $json Shorthand

`$json` is shorthand for `$input.item.json` — the current item's data. It's the most common expression.

**Gotcha in Merge nodes:** After a Merge node, `$json` refers to the MERGED output, not either input. The merged object's shape depends on the merge mode (see node-mapping.md). If the workflow uses `$json.field` after a Merge, you need to know which input contributed that field.
