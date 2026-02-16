# Huli Agents v2

Specialized AI agents for Huli development workflows.

## Installation

### Prerequisites

- [Claude Code CLI](https://claude.ai/code) installed
- Claude Pro or Team subscription

### Step 1: Add the Marketplace

```bash
/plugin marketplace add jcvierga/huli-agents-v2
```

### Step 2: Install the Plugin

```bash
/plugin install huli-agents
```

### Step 3: Verify Installation

```bash
/agents
```

You should see the Huli agents listed:
- `btu-estimator`
- `orchestrator-huli`
- `architect-vue2`
- `architect-go`
- `architect-php`
- `architect-bff`
- `architect-iris`
- `architect-mysql`
- `reviewer-integration`

### Step 4: Restart Claude Code

Restart Claude Code to load the new agents.

## Configuration

### For Teams (Recommended)

Add to your repository's `.claude/settings.json` for automatic team access:

```json
{
  "extraKnownMarketplaces": {
    "huli-agents": {
      "source": {
        "source": "github",
        "repo": "jcvierga/huli-agents-v2"
      }
    }
  }
}
```

### Local Development

For local development, symlink the repo:

```bash
ln -sf /path/to/huli-agents-v2 ~/.claude/plugins/marketplaces/huli-agents-v2
```

## Agents

### Estimation & Orchestration

| Agent | Purpose |
|-------|---------|
| `btu-estimator` | Analyze requirements, infer hidden work (BE, DB, queues), estimate tasks |
| `orchestrator-huli` | Coordinate full BTU workflow, delegate to specialists |

### Architects

| Agent | Repo/Technology |
|-------|-----------------|
| `architect-vue2` | practice-web (Vue.js 2.7) |
| `architect-go` | ehr, Go microservices |
| `architect-php` | practice-api (PHP) |
| `architect-bff` | hulipractice-api (BFF) |
| `architect-iris` | iris (queues, async) |
| `architect-mysql` | Database migrations |

### Reviewers

| Agent | Purpose |
|-------|---------|
| `reviewer-integration` | Validate cross-repo contracts and data flow |

## Usage

### Invoke an Agent

```bash
/agents orchestrator-huli
```

Then provide your prompt:

```
Plan BTU-1666: [paste requirements]
```

### Use via Task Tool

In your prompts, agents can be invoked via the Task tool:

```
Task(subagent_type: "architect-vue2", prompt: "Plan: [task description]")
```

## Documentation

- [MULTI-AGENT.md](docs/MULTI-AGENT.md) - How the multi-agent architecture works
- [CLAUDE.md](CLAUDE.md) - Agent catalog and conventions

## Updating

```bash
/plugin marketplace update
/plugin
# Navigate to Installed → huli-agents → Update
```

## Uninstalling

```bash
/plugin
# Navigate to Installed → huli-agents → Uninstall
```

## Contributing

1. Fork the repo
2. Create/modify agent in `agents/`
3. Update `plugin.json`
4. Submit PR

## License

MIT
