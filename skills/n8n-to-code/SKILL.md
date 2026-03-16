---
name: n8n-to-code
description: Converts n8n workflows (exported JSON or via n8n API) into clean, structured code in the user's language of choice. Analyzes nodes, connections, and dependencies, proposes architecture, and generates code in layers after user approval. Use when users mention "convert n8n", "migrate n8n", "n8n to code", "n8n workflow", or paste/reference n8n workflow JSON.
---

# n8n-to-code

Converts n8n workflows into clean, maintainable code. Supports any target language, all n8n nodes (with 3 fidelity levels), and multiple workflows with cross-dependency detection.

## When to Use This Skill

Invoke when users:

- Ask to "convert", "migrate", or "translate" an n8n workflow to code
- Paste or reference n8n workflow JSON (contains `"nodes"`, `"connections"`, `"settings"`)
- Point to a `.json` file exported from n8n
- Want to fetch workflows from an n8n instance API
- Say "n8n to code", "rewrite n8n in [language]", or similar

## Bundled Resources

- **references/node-mapping.md** — typeVersion schema diffs, extraction gotchas per node, Merge mode logic, continueOnFail semantics. **LOAD during Step 2** (analysis phase) when parsing nodes. Do NOT load during Step 1 or Step 3.
- **references/expression-conversion.md** — Runtime expression edge cases ($prevNode, $runIndex, $execution), nested additionalFields parsing, expression vs template syntax. **LOAD during Step 3 Layer 3** (orchestration). Do NOT load during analysis.
- **references/api-access.md** — n8n API endpoints, pagination, workflow JSON structure. **LOAD only when user chooses API access** in Step 1.1. Do NOT load for JSON/file input.

## Workflow Overview

```
Input (JSON / file / n8n API)
  → Collect preferences (language, architecture, output location)
  → Analyze workflow (nodes, flow, dependencies)
  → Present analysis document for approval
  → Generate code in 3 layers (structure → logic → orchestration)
  → Summary with TODOs and next steps
```

## Step 1: Collect Input

Ask the user three questions, one at a time:

### 1.1 — Workflow Source

> How would you like to provide the n8n workflow?
>
> A) Paste the JSON directly or point to a local `.json` file
> B) Fetch from an n8n instance API

**If the user picks B (API access):**

Ask for:
- n8n instance URL (e.g., `https://n8n.example.com`)
- API Key (generated in n8n Settings → API)

Then offer two modes:
- **List all workflows** — `GET {url}/api/v1/workflows` with header `X-N8N-API-KEY: {key}`. Present a numbered list (name, ID, active/inactive, last updated). The user picks one or more (e.g., "1, 3, 5" or "all").
- **Fetch specific workflow** — `GET {url}/api/v1/workflows/{id}` if the user already has the ID.

**Security:** Never save the API key to any file. Use only in-memory for the HTTP call. If the call fails, suggest fallback to manual JSON export.

### 1.2 — Target Language

> What language should the generated code use?
>
> Examples: TypeScript, JavaScript, Python, Go, Rust, Java, C#...

Accept any language. Adapt all generated code, imports, package managers, and idioms accordingly.

### 1.3 — Architecture Preferences

> How should the code be structured? Describe your preferred patterns, framework, or style. Or say "suggest for me" and I'll propose based on the workflow complexity.
>
> Examples:
> - "Express with TypeScript, repository pattern"
> - "FastAPI with clean architecture"
> - "Vanilla Node.js, keep it simple"
> - "Suggest for me"

If the user says "suggest for me", analyze the workflow complexity and propose an appropriate architecture (simple script for small workflows, modular structure for complex ones).

## Before You Start Converting — Ask Yourself

Before touching any node, think through these questions:

1. **What is the workflow's execution model?** Check `settings.executionOrder` — v0 runs nodes breadth-first (all nodes at same depth run before going deeper), v1 runs depth-first (follows one branch to completion). This fundamentally changes how you structure the orchestration code.
2. **Are there hidden state dependencies?** n8n workflows often rely on execution-time state: `$execution.id` for deduplication, `$runIndex` inside SplitInBatches loops, `$prevNode` for dynamic routing. These must become explicit parameters in code — they don't exist outside n8n.
3. **What's the error strategy?** Check which nodes have `continueOnFail: true` and whether there's an Error Trigger workflow. This determines try/catch placement — it's not uniform.
4. **Are credentials simple or complex?** API key credentials → env vars. OAuth2 credentials → need token refresh logic. HTTP Header Auth → may need middleware. Check the credential `type` in the JSON before mapping to env vars.

## Step 2: Analyze the Workflow

Parse the workflow JSON and produce an **Analysis Document**. This is the core deliverable before any code is written.

### Analysis Document Structure

```markdown
# Conversion Analysis — [Workflow Name]

## Summary
- Total nodes: X (Y full conversion, Z SDK/lib, W stubs)
- Triggers: [list of entry points]
- Estimated complexity: low / medium / high
- Target: [language] + [architecture]

## Node Inventory
| # | Node Name | n8n Type | Fidelity | Notes |
|---|-----------|----------|----------|-------|
| 1 | Webhook Entry | n8n-nodes-base.webhook | Full | Entry point, POST /webhook |
| 2 | Get Customer | n8n-nodes-base.postgres | SDK/lib | SELECT query, pg library |
| 3 | Check Status | n8n-nodes-base.if | Full | Condition: status == "active" |
| 4 | Send Alert | n8n-nodes-base.slack | SDK/lib | Slack Web API, #alerts channel |
| ... | ... | ... | ... | ... |

## Flow Map
Webhook Entry
  → Get Customer (Postgres)
    → [IF] Check Status
      ├─ true  → Process Order → Save Result (Postgres)
      └─ false → Send Alert (Slack) → Return Error

## External Dependencies
- PostgreSQL (nodes #2, #7) → [library for target language]
- Slack API (node #4) → [library for target language]
- Redis (node #5) → [library for target language]

## Proposed Architecture
[Based on user preferences or auto-suggestion]

- Entry point: [e.g., src/index.ts]
- Directory structure:
  src/
    ├── config/        → env, database connections
    ├── nodes/         → one module per node (or grouped)
    ├── flow/          → orchestration logic
    └── types/         → shared interfaces/types

## Decisions for Validation
1. Node "X" uses credentials "Y" — env variable, config file, or hardcoded?
2. Node "Z" is unrecognized — will be generated as a stub
3. Workflow has 3 parallel branches — use [parallel construct] or sequential?
4. [Any other decisions that need user input]
```

Present this document and wait for user approval. Adjust if the user requests changes.

## Step 3: Generate Code in Layers

After the user approves the analysis, generate code in 3 sequential layers:

### Layer 1 — Structure

- Create directory structure as proposed
- Entry point file with basic setup
- Config/env file with all detected environment variables
- Base types/interfaces extracted from workflow data shapes
- Dependency file (`package.json`, `requirements.txt`, `go.mod`, etc.) with all detected libraries

### Layer 2 — Node Logic

Generate one function/module per node (or grouped when trivial). Three fidelity levels:

**Full Conversion (flow/logic nodes):**
- Webhook → HTTP route/endpoint
- HTTP Request → HTTP client call (fetch, axios, requests, etc.)
- Code/Function → extracted and adapted inline code
- IF → conditional with converted expression
- Switch → switch/match with mapped branches
- Set → variable assignment / data transformation
- Merge → parallel join (Promise.all, asyncio.gather, etc.)
- Schedule Trigger / Cron → cron job or native scheduler
- Wait → sleep/delay or queue mechanism
- No Operation → passthrough comment

**SDK/Library Conversion (integration nodes):**
- Databases: Postgres, MySQL, MongoDB, Redis, SQLite → native library for the target language
- Messaging: Slack, Discord, Telegram, Email/SMTP, SendGrid → official SDK or HTTP call
- Cloud: AWS S3, Google Sheets, Google Drive, Airtable → official SDK
- APIs: Stripe, Twilio, GitHub, Jira, Notion → official SDK
- Files: FTP, SSH, CSV, XML, RSS → standard library or common package
- All other known nodes → best available library

For SDK/lib nodes: extract operation type, parameters, and credentials from the n8n JSON and generate the equivalent library call.

**Stub (unrecognized nodes):**
```
// TODO: Implement [NodeName] ([n8nType])
// Input: { field1: type, field2: type }
// Output: { result: type }
// Parameters from workflow: { ...extracted params }
// n8n docs: https://docs.n8n.io/integrations/builtin/...
//
// function nodeName(input: InputType): OutputType {
//   throw new Error('Not implemented');
// }
```

### Layer 3 — Orchestration

- Main function that connects all nodes in workflow execution order
- Branches (IF/Switch) become conditionals
- Merge nodes become parallel joins
- Error handling at integration points
- Trigger setup (webhook → route registration, cron → scheduler config)
- Converts n8n expressions throughout:
  - `{{ $json.field }}` → `data.field` (or equivalent)
  - `{{ $node["Name"].json.x }}` → `nameOutput.x`
  - `{{ $env.VAR }}` → `process.env.VAR` / `os.environ["VAR"]` / etc.

### Output Location

Ask the user:

> Where should I save the generated code?
>
> A) Current project directory
> B) New directory (e.g., `n8n-converted/` or custom name)

### Final Summary

After generation, present:

```markdown
## Conversion Complete

### Files Created
- src/index.ts (entry point)
- src/nodes/getCustomer.ts
- src/nodes/checkStatus.ts
- ...

### Dependencies to Install
[install command for the target language]

### TODOs (X remaining)
- [ ] Implement stub: [NodeName] (src/nodes/nodeName.ts:15)
- [ ] Set environment variable: DATABASE_URL
- [ ] Set environment variable: SLACK_TOKEN

### Next Steps
1. Install dependencies
2. Set environment variables
3. Implement remaining stubs
4. Test the flow end-to-end
```

## Multiple Workflows

When the user selects multiple workflows:

1. Analyze each workflow individually
2. Present a **consolidated analysis** in a single document
3. Detect cross-workflow dependencies:
   - `Execute Workflow` / `Execute Sub-workflow` nodes → become function calls to the other workflow's module
   - Shared credentials → shared config
4. Propose unified structure:
   - Shared modules: config, types, utils
   - One module per workflow
   - Shared entry point or separate entry points (ask user)
5. Generate with the same 3-layer approach, plus a shared layer for common code

## NEVER Do

- **NEVER ignore `typeVersion`** — n8n nodes change parameter schema between versions. A Postgres node v1 uses `query` field directly, v2 uses `operation` + `query` with different nesting. An HTTP Request v1 has `url` at root, v4 nests it under `options`. Always check `typeVersion` and adapt extraction logic. If you don't recognize the version, flag it in the analysis as a risk.
- **NEVER treat Code nodes as simple functions** — Code nodes can access `$input.all()` (all items, not just current), `$input.first()`, `$execution.id`, `$env`, and other runtime globals. These must become explicit function parameters. Silently dropping them produces code that compiles but behaves differently.
- **NEVER assume IF nodes are simple if/else** — check for `continueOnFail` (changes error flow), `looseTypeValidation` (changes comparison semantics), and multi-condition rules with `combineOperation: "any"` vs `"all"`. Some IF nodes are complex AND/OR chains, not simple comparisons.
- **NEVER flatten all credentials to env vars** — OAuth2 credentials have token refresh logic that needs a refresh flow in code. API key credentials are simple env vars. HTTP Header Auth may need request middleware. Check the credential `type` in the JSON before deciding how to handle auth.
- **NEVER convert Execute Workflow as a plain function call without checking data passing** — the parent passes its current `$json` to the sub-workflow's trigger, but the sub-workflow may also access `$execution` metadata or have its own static data. Verify what actually flows between them by checking the Execute Workflow node's `parameters.workflowValues`.
- **NEVER skip the `settings` object** — it contains `executionOrder` (v0 breadth-first vs v1 depth-first — fundamentally changes orchestration), `timezone` (affects all date operations), `saveManualExecutions`, and `errorWorkflow` (ID of error handler workflow).
- **NEVER treat Merge node as a simple join** — Merge has 5 modes: `append` (concatenate arrays), `combine` → `mergeByFields` (SQL-like join on key), `combine` → `multiplex` (cartesian product), `combine` → `chooseBranch` (pick one input), and `keepKeyMatches`. Each needs completely different code. Check `parameters.mode` and `parameters.joinMode`.
- **NEVER generate code that ignores item-level processing** — n8n processes arrays of items. A node that receives 10 items may process each independently (most nodes) or all at once (Code node with `$input.all()`). Check whether the node is per-item or batch. Getting this wrong means generating a function that processes one record when it should process many.

## Key Principles

- **Never generate code before analysis approval** — the analysis document is the contract
- **Degrade gracefully** — unknown nodes become useful stubs, not errors
- **Respect user choices** — language, architecture, output location are all user decisions
- **Preserve semantics** — the generated code must do what the workflow did, not just look similar
- **Keep it practical** — include install commands, env var lists, and actionable TODOs
