# CLAUDE.md

Specialized AI agents for Huli development workflows.

## Philosophy

**Specialized over generic.** Each agent has ONE job and does it well.

## Agent Catalog

### Estimation
| Agent | Job | Model |
|-------|-----|-------|
| `btu-estimator` | Analyze requirements, infer hidden work, estimate tasks | opus |

### Orchestration
| Agent | Job | Model |
|-------|-----|-------|
| `orchestrator-huli` | Coordinate full BTU workflow, delegate to specialists | opus |

### Architects (by repo/tech)
| Agent | Repo/Tech | Model |
|-------|-----------|-------|
| `architect-vue2` | practice-web (Vue.js 2.7) | opus |
| `architect-go` | ehr, Go microservices | opus |
| `architect-php` | practice-api (PHP) | opus |
| `architect-bff` | hulipractice-api (BFF) | opus |
| `architect-iris` | iris (queues, async) | opus |
| `architect-mysql` | Database migrations | opus |

### Reviewers
| Agent | Job | Model |
|-------|-----|-------|
| `reviewer-integration` | Validate cross-repo contracts and data flow | opus |

## Workflow

```
User
  │
  ▼
orchestrator-huli
  │
  ├──► btu-estimator (infer all work)
  │         │
  │         ▼
  │    Tasks across repos
  │
  ├──► architect-vue2 ────┐
  ├──► architect-go ──────┤
  ├──► architect-php ─────┼──► Plans per repo
  ├──► architect-bff ─────┤
  ├──► architect-iris ────┤
  ├──► architect-mysql ───┘
  │
  └──► reviewer-integration (validate all)
              │
              ▼
         Final Plan
```

## Key Principles

1. **Infer hidden work** - FE data needs = BE endpoint needed
2. **Specialize by repo** - Each architect knows their codebase
3. **Review integration** - Validate contracts before implementation
4. **Consolidate tasks** - Fewer, larger tasks over many small ones

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

3. Update orchestrator if agent is part of main workflow

## Naming Convention

- `btu-*` - BTU workflow agents
- `architect-*` - Technology/repo-specific architects
- `orchestrator-*` - Workflow coordinators
- `reviewer-*` - Review specialists
