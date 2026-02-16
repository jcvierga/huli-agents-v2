# Multi-Agent Architecture

This document explains how the Huli specialized agents work together to analyze and plan BTU implementations.

## How It Works

### The Problem with Generic Agents

Traditional AI assistants try to do everything: read requirements, analyze code, estimate effort, and plan implementation. This leads to:

- **Context overload**: Too much information to process effectively
- **Generic responses**: No deep expertise in any area
- **Inconsistent output**: Different approaches each time
- **Missed details**: Hidden work goes unnoticed

### The Specialized Agent Solution

We decompose the problem into specialized roles, each with:

- **Single responsibility**: One job, done well
- **Domain expertise**: Deep knowledge of their area
- **Consistent output**: Predictable, structured results
- **Focused context**: Only the information they need

## Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER REQUEST                                  │
│                    "Plan BTU-1666: Patient Homepage V2"              │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      ORCHESTRATOR-HULI                               │
│  • Receives BTU request                                              │
│  • Coordinates all agents                                            │
│  • Consolidates final output                                         │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    ▼                              │
┌───────────────────────────────┐                 │
│        BTU-ESTIMATOR          │                 │
│  • Analyzes requirements       │                 │
│  • INFERS hidden work          │                 │
│  • Outputs tasks + points      │                 │
│  • NO code analysis            │                 │
└───────────────────────────────┘                 │
          │                                        │
          │ Tasks: [db], [ehr], [practice-web]    │
          ▼                                        │
┌─────────────────────────────────────────────────┴───────────────────┐
│                     ARCHITECT DELEGATION                             │
│  Orchestrator routes each task to the appropriate specialist         │
└─────────────────────────────────────────────────────────────────────┘
          │
          ├────────────────┬────────────────┬────────────────┐
          ▼                ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ARCHITECT-    │  │ARCHITECT-GO  │  │ARCHITECT-    │  │ARCHITECT-    │
│MYSQL         │  │              │  │VUE2          │  │VUE2          │
│              │  │              │  │              │  │              │
│Task 1: Index │  │Task 2: API   │  │Task 3: FE    │  │Task 4,5: FE  │
│              │  │endpoint      │  │somatometry   │  │improvements  │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
          │                │                │                │
          │  Schema +      │  Proto +       │  Components +  │  Modifications
          │  Migration     │  Handler       │  Charts        │  + Fixes
          ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    REVIEWER-INTEGRATION                              │
│  • Validates API contracts between repos                             │
│  • Checks data flow end-to-end                                       │
│  • Identifies missing pieces                                         │
│  • Confirms dependency order                                         │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        FINAL OUTPUT                                  │
│  • Consolidated task list with dependencies                          │
│  • Implementation plans per repo                                     │
│  • Integration validation results                                    │
└─────────────────────────────────────────────────────────────────────┘
```

## How Claude Code Executes This

### Behind the Scenes

When you invoke the orchestrator, Claude Code uses the **Task tool** to spawn subagents:

```
Main Conversation (orchestrator-huli)
    │
    ├─► Task(subagent_type: "btu-estimator", prompt: "...")
    │       └─► Spawns new Claude instance with btu-estimator.md context
    │           └─► Returns: Task list with inferred work
    │
    ├─► Task(subagent_type: "architect-mysql", prompt: "...")
    │       └─► Spawns new Claude instance with architect-mysql.md context
    │           └─► Has access to: Read, Glob, Grep, Bash
    │           └─► Returns: Migration plan
    │
    ├─► Task(subagent_type: "architect-go", prompt: "...")
    │       └─► Spawns new Claude instance with architect-go.md context
    │           └─► Analyzes ehr codebase
    │           └─► Returns: Proto + handler plan
    │
    ├─► Task(subagent_type: "architect-vue2", prompt: "...") × 3
    │       └─► Spawns new Claude instances for each FE task
    │           └─► Analyzes practice-web codebase
    │           └─► Returns: Component plans
    │
    └─► Task(subagent_type: "reviewer-integration", prompt: "...")
            └─► Reviews all plans together
            └─► Returns: Validation results
```

### Agent Communication

Agents don't communicate directly with each other. Instead:

1. **Orchestrator collects outputs** from each agent
2. **Orchestrator passes context** to the next agent as needed
3. **Final consolidation** happens in the main conversation

```
btu-estimator output ──┐
                       │
architect-mysql output ├──► orchestrator ──► reviewer-integration
                       │
architect-go output ───┤
                       │
architect-vue2 output ─┘
```

### Context Isolation

Each agent runs in isolation with:

- **Its own system prompt** (the agent .md file)
- **Access to specified tools** only
- **Fresh context** (no conversation history from other agents)
- **Focused task** (single responsibility)

This isolation is **intentional**:
- Prevents context pollution
- Ensures consistent behavior
- Makes each agent predictable
- Enables parallel execution

## Real Example: BTU-1666

### Input

```
BTU-1666: Patient Homepage V2 - Somatometría

Requirements:
- Display vital signs from last 6 consultations
- Show evolutionary charts
- Click navigates to source consultation
- Iterate appointments block
- Various UX improvements
```

### Agent Execution Flow

#### 1. btu-estimator (First)

**Input:** Raw BTU requirements
**Process:**
- Reads requirements
- Identifies 3 main objectives
- **INFERS hidden work**: "Display vital signs from last 6 consultations" → needs BE endpoint
- **INFERS DB work**: Aggregation query → needs index

**Output:**
```
| # | Repo         | Task                                    | Pts |
|---|--------------|----------------------------------------|-----|
| 1 | db           | Add index for vital signs history query | 1   |
| 2 | ehr          | Create vital-signs history endpoint     | 5   |
| 3 | practice-web | Create somatometry section              | 8   |
| 4 | practice-web | Iterate appointments block              | 5   |
| 5 | practice-web | Apply UX V1-V2 improvements             | 5   |
```

**Key insight:** Tasks 1 and 2 were NOT in the original BTU - they were inferred.

#### 2. architect-mysql (Parallel)

**Input:** Task 1 from estimator
**Process:**
- Searches for `patient_checkup` table schema
- Finds existing indexes
- Identifies missing composite index for query pattern

**Output:**
```sql
ALTER TABLE `hh`.`patient_checkup`
ADD INDEX `ix_patient_checkup_patient_history`
(`id_patient_file`, `created_on` DESC);
```

#### 3. architect-go (Parallel)

**Input:** Task 2 from estimator
**Process:**
- Reads ehr CLAUDE.md for patterns
- Finds existing vital signs proto
- Designs new endpoint following handler pattern

**Output:**
- Proto definition for request/response
- Handler structure in `internal/service/handler/patient/vital-signs-history/`
- SQL query with proper indexes

#### 4. architect-vue2 (Parallel, 3 instances)

**Input:** Tasks 3, 4, 5 from estimator
**Process:**
- Reads practice-web CLAUDE.md
- Finds similar components (patient-consents, development-curves-chart)
- Maps DataMixin pattern, Chart.js usage

**Output:**
- New component files with paths
- Modifications to existing components
- i18n additions

#### 5. reviewer-integration (Last)

**Input:** All architect outputs
**Process:**
- Validates API contract: ehr endpoint ↔ practice-web consumer
- Checks field names, types, nullability
- Verifies dependency order

**Output:**
```
| Aspect          | FE Expects | BE Provides | Status |
|-----------------|------------|-------------|--------|
| Field: checkupId | number     | int64       | ✅     |
| Field: pulse     | number     | Int32Value  | ✅     |
| All 8 vitals     | present    | present     | ✅     |

Dependencies validated:
1. [db] → 2. [ehr] → 3. [practice-web]
```

### Final Consolidated Output

```
BTU-1666: Patient Homepage V2 - Somatometría

Total: 24 pts | 5 tasks | 3 repos

Execution Order:
├── Task 1 (db) ──► Task 2 (ehr) ──► Task 3 (practice-web)
├── Task 4 (practice-web) - parallel
└── Task 5 (practice-web) - parallel

Inferred Work (Not in Original BTU):
- Task 1: Index required for query performance
- Task 2: New endpoint required - aggregated data doesn't exist
```

## Why This Approach Works

### 1. Inference Over Transcription

The `btu-estimator` doesn't just transcribe requirements - it **thinks like a tech lead**:

> "If FE needs to display data, where does that data come from?"

This catches hidden work that product managers don't specify.

### 2. Specialization Over Generalization

Each architect knows their domain deeply:

- `architect-mysql` knows index patterns, migration syntax
- `architect-go` knows handler structure, proto conventions
- `architect-vue2` knows DataMixin, Chart.js, BEM naming

### 3. Validation Over Assumption

`reviewer-integration` catches contract mismatches before implementation:

- Field naming inconsistencies (camelCase vs snake_case)
- Missing fields in API responses
- Type mismatches (string vs number)

### 4. Consolidation Over Fragmentation

The orchestrator provides a **single source of truth**:

- One task list with dependencies
- Clear execution order
- All repos accounted for

## Comparison: With vs Without Multi-Agent

| Aspect | Single Agent | Multi-Agent |
|--------|--------------|-------------|
| Tasks identified | 3 (explicit only) | 5 (with inferred) |
| Repos covered | 1 (practice-web) | 3 (db, ehr, practice-web) |
| Hidden work found | None | Index + API endpoint |
| Contract validation | None | Full validation |
| Consistency | Variable | Structured |

## Limitations

### Current Limitations

1. **No direct agent communication** - All routing through orchestrator
2. **Sequential dependency** - Integration review must wait for all architects
3. **Manual invocation** - User must call orchestrator explicitly

### Future Improvements

1. **Parallel architect execution** - Run all architects simultaneously
2. **Iterative refinement** - Architects can request clarification
3. **Automatic triggers** - Invoke on BTU ticket creation
4. **Learning from feedback** - Improve estimates based on actuals
