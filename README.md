# Huli Agents v2

Specialized AI agents for Huli development workflows.

## Philosophy

**Specialized over generic.** Each agent has ONE job and does it well. Together, they form a multi-agent system that can analyze requirements, infer hidden work, and create comprehensive implementation plans across all repositories.

## Installation

```bash
# Add marketplace
/plugin marketplace add jcvierga/huli-agents-v2

# Install plugin
/plugin install huli-agents
```

## Agents

### Estimation & Orchestration

| Agent | Purpose |
|-------|---------|
| `btu-estimator` | Analyze requirements, **infer hidden work** (BE, DB, queues), estimate tasks |
| `orchestrator-huli` | Coordinate full BTU workflow, delegate to specialists |

### Architects (by repo/technology)

| Agent | Repo/Tech |
|-------|-----------|
| `architect-vue2` | practice-web (Vue.js 2.7) |
| `architect-go` | ehr, Go microservices |
| `architect-php` | practice-api (PHP) |
| `architect-bff` | hulipractice-api (BFF aggregation) |
| `architect-iris` | iris (queues, async processing) |
| `architect-mysql` | Database migrations |

### Reviewers

| Agent | Purpose |
|-------|---------|
| `reviewer-integration` | Validate cross-repo contracts and data flow |

## Usage

### Full BTU Planning (Recommended)

```
/agents orchestrator-huli

Prompt: Plan BTU-1666: [paste requirements]
```

The orchestrator will:
1. Delegate to `btu-estimator` to analyze and infer all work
2. Route tasks to appropriate architects
3. Run `reviewer-integration` to validate contracts
4. Return consolidated plan

### Individual Agents

```bash
# Just estimate tasks
/agents btu-estimator
Prompt: Estimate: [requirements]

# Just plan Vue.js implementation
/agents architect-vue2
Prompt: Plan: [practice-web] Create somatometry section - 8pts
```

## Example: BTU-1666

### Input

```
BTU-1666: Patient Homepage V2 - Somatometría

Requirements:
- Display vital signs from last 6 consultations with charts
- Click navigates to source consultation
- Iterate appointments block (empty state, 3 types)
- Various UX V1→V2 improvements
```

### Output

```
BTU-1666: Patient Homepage V2 - Somatometría

Total: 24 pts | 5 tasks | 3 repos

| # | Repo         | Task                                              | Pts |
|---|--------------|---------------------------------------------------|-----|
| 1 | db           | Add index for vital signs history query           | 1   |
| 2 | ehr          | Create vital-signs history endpoint               | 5   |
| 3 | practice-web | Create somatometry section with charts            | 8   |
| 4 | practice-web | Iterate appointments block (empty state, 3 types) | 5   |
| 5 | practice-web | Apply UX V1-V2 improvements                       | 5   |

Inferred Work (Not in Original BTU):
- Task 1: Index required for query performance
- Task 2: New endpoint required - FE needs aggregated data

Execution Order:
├── Task 1 (db) ──► Task 2 (ehr) ──► Task 3 (practice-web)
├── Task 4 (practice-web) - parallel
└── Task 5 (practice-web) - parallel

Integration Validated:
✅ API contract: ehr ↔ practice-web
✅ All 8 vital sign fields present
✅ checkupId included for navigation
```

### What Happened Behind the Scenes

```
orchestrator-huli
       │
       ├─► btu-estimator
       │       └─► Inferred: Tasks 1 & 2 (hidden BE work)
       │
       ├─► architect-mysql ──► Migration script for index
       ├─► architect-go ────► Proto + handler for endpoint
       ├─► architect-vue2 ──► 3 component plans
       │
       └─► reviewer-integration
               └─► Validated contracts between repos
```

See [docs/MULTI-AGENT.md](docs/MULTI-AGENT.md) for detailed explanation of how agents work together.

## Key Principles

1. **Infer hidden work** - FE data needs = BE endpoint needed
2. **Specialize by repo** - Each architect knows their codebase patterns
3. **Review integration** - Validate contracts before implementation
4. **Consolidate tasks** - Fewer, larger tasks over many small ones

## Documentation

- [MULTI-AGENT.md](docs/MULTI-AGENT.md) - How multi-agent architecture works
- [CLAUDE.md](CLAUDE.md) - Agent catalog and workflow

## Contributing

1. Fork the repo
2. Create/modify agent in `agents/`
3. Update `plugin.json`
4. Submit PR

## License

MIT
