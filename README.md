# Huli Agents v2

Specialized AI agents for Huli development workflows.

## Installation

```bash
# Add marketplace
/plugin marketplace add hulilabs/huli-agents-v2

# Install plugin
/plugin install huli-agents
```

## Agents

| Agent | Purpose |
|-------|---------|
| `btu-estimator` | PM-level BTU estimation (no code analysis) |
| `architect-vue2` | Vue.js 2.7 implementation planning |
| `architect-go` | Go microservices planning |
| `architect-php` | PHP backend planning |
| `orchestrator-huli` | Workflow coordination |

## Usage

### Estimate a BTU

```
/agents btu-estimator

Prompt: Estimate BTU-1666: Patient Homepage V2 - Somatometria
[paste requirements]
```

### Get Implementation Plan

```
/agents architect-vue2

Prompt: Create implementation plan for:
[practice-web] Create somatometry section with vital sign cards - 8pts
```

### Full Workflow

```
/agents orchestrator-huli

Prompt: Plan BTU-1666: [paste requirements]
```

## Philosophy

- **Specialized > Generic**: Each agent does ONE thing well
- **Consolidate tasks**: Fewer, larger tasks over many small ones
- **PM first, code second**: Estimate without code, then dive in
- **Iterate**: These agents evolve with your workflow

## Contributing

1. Fork the repo
2. Create/modify agent in `agents/`
3. Update `plugin.json`
4. Submit PR

## License

MIT
