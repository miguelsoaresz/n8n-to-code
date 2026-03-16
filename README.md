# n8n-to-code

A Claude Code plugin that converts n8n workflows into clean, structured code in any programming language.

## Installation

```bash
claude plugin add intellectco/n8n-to-code
```

## What It Does

Takes an n8n workflow (exported JSON or fetched via n8n API) and converts it into production-ready code in your language of choice.

### Flow

1. **Input** — Provide workflow JSON, point to a file, or connect to your n8n instance API
2. **Analyze** — The skill parses all nodes, maps the execution flow, detects dependencies, and proposes an architecture
3. **Approve** — Review the analysis document before any code is generated
4. **Generate** — Code is created in 3 layers: structure, node logic, orchestration

### Supported Nodes

| Level | What Happens | Examples |
|-------|-------------|---------|
| Full conversion | Generates working code | Webhook, HTTP Request, IF, Switch, Code, Set, Merge, Cron, Wait |
| SDK/library | Generates code using the right library | Postgres, Redis, Slack, S3, Stripe, Google Sheets, Twilio |
| Stub | Generates typed placeholder with TODO | Any unrecognized node |

### Features

- Any target language (TypeScript, Python, Go, Rust, Java, C#, etc.)
- n8n API integration — list and fetch workflows from your instance
- Multi-workflow support with cross-dependency detection
- Expression conversion (`{{ }}` syntax to target language)
- typeVersion-aware parsing (handles schema differences between node versions)
- Analysis-first approach — always review before generating

## Usage

Trigger the skill by mentioning n8n conversion in any Claude Code conversation:

```
> Convert my n8n workflow to TypeScript
> Migrate this n8n JSON to Python
> Fetch workflows from my n8n instance and convert them
```

Or reference a workflow file:

```
> Convert the workflow in ./my-workflow.json to Go
```

## License

MIT
