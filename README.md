# Buildkite Linear Issue Pipeline Example

[![Build status](https://badge.buildkite.com/FIXME.svg?branch=main)](https://buildkite.com/buildkite/linear-issue-pipeline-example)
[![Add to Buildkite](/.buildkite/add.svg)](https://buildkite.com/new)

<!-- docs:start -->

This example demonstrates an **AI-powered issue analysis pipeline** built with Buildkite [dynamic pipelines](https://buildkite.com/docs/pipelines/configure/dynamic-pipelines) and [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview). When a Linear issue gets a specific label, a Buildkite pipeline kicks off an AI agent to analyse the issue and start working on it.

👉 See this example in action: [buildkite/linear-issue-pipeline-example](https://buildkite.com/buildkite/linear-issue-pipeline-example)

## How it works

1. A Linear issue is created or updated with the `buildkite-analyze` label
2. Linear sends a webhook to Buildkite, which starts a build
3. The first step evaluates the webhook payload — if it's not a create/update action or the label doesn't match, the build exits early
4. A TypeScript handler reads the Linear webhook payload, extracts the issue ID, title, and description
5. The handler uses the [Buildkite SDK](https://github.com/buildkite/buildkite-sdk) to **dynamically generate an analysis step** and uploads it with `buildkite-agent pipeline upload`
6. That step launches Claude Code in a Docker container with access to the codebase, Linear (via the Linear CLI), and GitHub (via `gh` CLI) to analyse and work on the issue

### The handler pattern

The core of the handler is short — read the webhook payload from build metadata, evaluate whether to act, and generate a step with the Buildkite SDK:

```typescript
// 1. Read the webhook payload that Buildkite stored as build metadata
const payload = JSON.parse(
  execSync("buildkite-agent meta-data get buildkite:webhook").toString(),
);

// 2. Evaluate the condition — create/update action with a matching label?
const labels = (payload.data.labels ?? []).map((l) => l.name);
if (!["create", "update"].includes(payload.action) || !labels.includes(process.env.TRIGGER_ON_LABEL)) {
  process.exit(0);
}

// 3. Generate a step with the Buildkite SDK and pipe it into `pipeline upload`
const pipeline = new Pipeline();
pipeline.addStep({ label: ":linear: Analyse the issue", command: "scripts/claude.sh" });
execSync("buildkite-agent pipeline upload", { input: pipeline.toYAML() });
```

The real handler extracts the issue ID, title, and description between steps 2 and 3 so Claude has them on `process.env` — see [`scripts/handler.ts`](./scripts/handler.ts).

The key Buildkite features at play:

- **`buildkite-agent pipeline upload`** — adding steps to a running build based on runtime conditions
- **`buildkite-agent meta-data`** — reading webhook payloads stored as build metadata
- **`@buildkite/buildkite-sdk`** — programmatically generating pipeline YAML in TypeScript
- **Buildkite webhooks** — triggering builds from external events (Linear, not just GitHub)
- **Buildkite Hosted Models** — proxying LLM requests through Buildkite's model provider endpoint

## What's interesting about this?

This example shows that dynamic pipelines aren't limited to GitHub workflows. Any webhook source — Linear, Jira, PagerDuty, your own services — can trigger a Buildkite build that evaluates the payload and dynamically decides what to do. The handler pattern (read metadata → evaluate conditions → generate and upload steps) works the same regardless of where the webhook comes from.

## Setup

To run this yourself, you'll need:

- A [Buildkite account](https://buildkite.com/signup)
- A [Linear](https://linear.app) workspace with webhook support
- A [Linear API token](https://linear.app/settings/api)
- An [Anthropic API key](https://console.anthropic.com/settings/keys) (or use [Buildkite Hosted Models](https://buildkite.com/docs/pipelines/hosted-models))
- Docker installed on your Buildkite agent

1. Fork this repo
2. Create a Buildkite pipeline pointing to your fork with webhook support enabled
3. In Linear, create a [webhook](https://linear.app/developers/webhooks) pointing to your Buildkite pipeline's webhook URL, sending `Issue` events
4. Set up the required secrets: `GITHUB_TOKEN`, `BUILDKITE_API_TOKEN`, and `LINEAR_API_TOKEN`
5. Add the `buildkite-analyze` label to any Linear issue

<!-- docs:end -->

## Credits

Originally built by [Grant Colegate](https://github.com/grantc) and [Christian Nunciato](https://github.com/cnunciato) as a demo for AWS re:Invent.

## License

See [LICENSE](LICENSE) (MIT)
