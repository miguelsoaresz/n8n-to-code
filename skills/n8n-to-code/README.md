# n8n-to-code

A Claude Code skill that converts n8n workflows into clean, structured code in any programming language.

## Purpose

n8n is a powerful workflow automation tool, but as projects grow, teams often need to migrate workflows to maintainable code. This skill automates that conversion by analyzing n8n workflow JSON, proposing architecture, and generating production-ready code.

## When to Use

- Converting n8n workflows to a target programming language
- Migrating from n8n to a code-based solution
- Understanding what an n8n workflow does by converting it to readable code
- Batch-converting multiple workflows from an n8n instance

## How It Works

### 1. Collect Input
The skill asks for the workflow source (JSON, file, or n8n API), target language, and architecture preferences.

### 2. Analyze
Parses all nodes, connections, and dependencies. Produces an analysis document with a node inventory, flow map, external dependencies, and proposed architecture. Waits for user approval.

### 3. Generate
After approval, generates code in 3 layers:
- **Structure** — directories, config, types, dependency file
- **Logic** — one function/module per node at 3 fidelity levels
- **Orchestration** — main flow connecting everything in execution order

## Node Support

| Level | Coverage | Example |
|-------|----------|---------|
| Full conversion | Flow and logic nodes | Webhook, IF, Switch, HTTP Request, Code, Set, Merge, Cron, Wait |
| SDK/library | Integration nodes with known libraries | Postgres, Redis, Slack, S3, Stripe, Twilio, Google Sheets, etc. |
| Stub | Unrecognized nodes | Typed stub with TODO, parameters, and docs link |

## Features

- **Any language** — TypeScript, JavaScript, Python, Go, Rust, Java, C#, or anything else
- **n8n API integration** — list and fetch workflows directly from an n8n instance
- **Multi-workflow** — convert multiple workflows at once with cross-dependency detection
- **Expression conversion** — translates n8n `{{ }}` expressions to target language syntax
- **Analysis-first** — always presents a conversion plan before generating code
- **Graceful degradation** — unknown nodes become useful stubs, not errors

## Example

```
User: Convert my n8n workflow to TypeScript

Skill: How would you like to provide the workflow?
       A) Paste JSON or point to a file
       B) Fetch from n8n API

User: B — here's my instance: https://n8n.example.com

Skill: [Lists all workflows]
       Which workflow(s) would you like to convert?

User: 1 and 3

Skill: What architecture style?

User: Express, clean modules

Skill: [Presents analysis document with node inventory, flow map, architecture]
       Does this look right?

User: Yes, go ahead

Skill: [Generates code in 3 layers, presents summary with files, TODOs, next steps]
```
