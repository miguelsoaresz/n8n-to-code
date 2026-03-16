# n8n-to-code

Convert n8n workflows into clean, structured code in any programming language.

A skill plugin for [Claude Code](https://claude.ai/claude-code).

## Why migrate away from n8n?

n8n is great for prototyping and small automations. But as your project grows, the visual workflow model starts working against you:

**Version control is painful.** Workflow JSON diffs are unreadable. Good luck reviewing a PR that changes 3 nodes in a 200-node workflow — you're staring at thousands of lines of shuffled JSON with no meaningful context.

**Testing doesn't exist.** There's no unit testing, no mocking, no CI pipeline for workflows. You either test manually by clicking "Execute Workflow" or you don't test at all. Most teams don't test at all.

**Refactoring is manual labor.** Renaming a field that 15 nodes reference? Extracting shared logic into a reusable piece? In code, that's a find-and-replace. In n8n, you're clicking through nodes one by one, hoping you didn't miss one.

**Debugging is guesswork.** When a workflow fails at 3am, you get a node name and an error message. No stack traces, no structured logging, no breakpoints. Complex workflows with branches and sub-workflows turn debugging into archaeology.

**Collaboration doesn't scale.** Two people can't work on the same workflow simultaneously. There's no branching, no merging, no code review process. The workflow is a single monolithic blob owned by whoever touched it last.

**Performance hits a ceiling.** n8n processes items synchronously within a workflow. When you need connection pooling, caching, rate limiting, or parallel processing with backpressure, you're fighting the platform instead of building features.

**Vendor lock-in is real.** Your business logic lives inside a proprietary execution engine. If n8n changes pricing, drops a feature, or goes away, your workflows aren't portable. Code is.

At some point, the speed advantage of drag-and-drop disappears under the weight of these limitations. This plugin helps you make the transition without starting from scratch.

## Installation

```bash
claude plugin add miguelsoaresz/n8n-to-code
```

## How it works

The plugin analyzes your n8n workflow and converts it to code through a 3-step process:

1. **You provide the workflow** — paste JSON, point to a file, or connect directly to your n8n instance API to list and fetch workflows
2. **Review the analysis** — the plugin maps every node, traces the execution flow, flags external dependencies, and proposes a code architecture. Nothing is generated until you approve
3. **Get your code** — generated in 3 layers (project structure, node logic, orchestration) in your language of choice

### What you choose

- **Target language** — TypeScript, Python, Go, Rust, Java, C#, or anything else
- **Architecture** — describe your preferred patterns and framework, or let the plugin suggest based on workflow complexity
- **Output location** — current project or a new directory

### Node coverage

Every n8n node is handled, with 3 levels of conversion fidelity:

| Level | What happens | Examples |
|-------|-------------|---------|
| Full | Working code, ready to run | Webhook, HTTP Request, IF, Switch, Code, Set, Merge, Cron, Wait |
| SDK/lib | Code using the appropriate library | Postgres, Redis, Slack, S3, Stripe, Google Sheets, Twilio |
| Stub | Typed function with TODO and docs link | Any node the plugin doesn't recognize |

The plugin is typeVersion-aware — it handles schema differences between node versions (e.g., HTTP Request v1 vs v4 have completely different parameter structures) so the generated code actually matches what your workflow does.

### Multi-workflow support

Select multiple workflows from your n8n instance and convert them together. The plugin detects cross-workflow dependencies (Execute Workflow nodes), extracts shared config, and generates a unified codebase with proper module boundaries.

## Usage

Mention n8n conversion in any Claude Code conversation:

```
Convert my n8n workflow to TypeScript
```
```
Migrate the workflow in ./checkout-flow.json to Python with FastAPI
```
```
Fetch all workflows from my n8n instance and convert the active ones to Go
```

## Built from experience

This plugin was built by a team that migrated a production system with 20 interconnected n8n workflows (14 ERP integrations, 5 AI agents, 15 tools) into 4,700 lines of typed, tested, deployable code. The conversion patterns, gotchas, and node-specific knowledge baked into this plugin come from that migration.

## License

MIT
