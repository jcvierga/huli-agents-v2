# CLAUDE.md

This repository contains specialized AI agents for Huli development workflows.

## Philosophy

**Specialized over generic.** Each agent has ONE job and does it well.

| Agent | Job | Reads Code? |
|-------|-----|-------------|
| `btu-estimator` | PM-level task estimation | No |
| `architect-vue2` | Vue.js 2.7 implementation plans | Yes |
| `architect-go` | Go microservices plans | Yes |
| `architect-php` | PHP backend plans | Yes |
| `orchestrator-huli` | Coordinate the workflow | Delegates |

## Workflow

```
User → orchestrator-huli → btu-estimator → [architect-*] → Implementation Plan
```

## Adding New Agents

1. Create `agents/agent-name.md` with frontmatter:
   ```yaml
   ---
   name: agent-name
   description: What it does
   model: sonnet|opus
   tools: Read, Glob, Grep, Bash
   ---
   ```

2. Add to `plugin.json` agents array

3. Document the agent's:
   - Input (what it receives)
   - Output (what it produces)
   - Rules (constraints)

## Naming Convention

- `btu-*` - BTU workflow agents
- `architect-*` - Technology-specific architects
- `orchestrator-*` - Workflow coordinators
- `reviewer-*` - Code review specialists

## Testing Agents

Use Claude Code to invoke:
```
/agents architect-vue2
```

Or via Task tool:
```javascript
Task(subagent_type: "architect-vue2", prompt: "...")
```
