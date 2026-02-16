# Multi-Agent Architecture

This document explains how the Huli specialized agents work together, the technical implementation behind Claude Code's multi-agent system, and a real-world example of how agents collaborated on BTU-1666.

---

## Table of Contents

1. [The Problem with Generic Agents](#the-problem-with-generic-agents)
2. [How Claude Code Implements Multi-Agent](#how-claude-code-implements-multi-agent)
3. [Technical Deep Dive: The Task Tool](#technical-deep-dive-the-task-tool)
4. [Agent Communication Model](#agent-communication-model)
5. [Huli Agents Architecture](#huli-agents-architecture)
6. [Real Example: BTU-1666](#real-example-btu-1666)
7. [Observations: Theory vs Practice](#observations-theory-vs-practice)

---

## The Problem with Generic Agents

Traditional AI assistants try to do everything: read requirements, analyze code, estimate effort, and plan implementation. This leads to:

| Problem | Impact |
|---------|--------|
| **Context overload** | Too much information to process effectively |
| **Generic responses** | No deep expertise in any area |
| **Inconsistent output** | Different approaches each time |
| **Missed details** | Hidden work goes unnoticed |
| **Token bloat** | Large outputs consume context window |

### The Specialized Agent Solution

We decompose the problem into specialized roles, each with:

- **Single responsibility**: One job, done well
- **Domain expertise**: Deep knowledge of their area
- **Consistent output**: Predictable, structured results
- **Focused context**: Only the information they need
- **Isolated token usage**: Verbose output stays contained

---

## How Claude Code Implements Multi-Agent

Claude Code provides two mechanisms for multi-agent coordination:

### 1. Subagents (What We Use)

Subagents work **within a single session**. They:

- Run in isolated context windows
- Report only to the parent agent
- Cannot communicate directly with each other
- Cannot spawn other subagents (no nesting)
- Return summarized results to parent

```
┌─────────────────────────────────────────────┐
│              MAIN CONVERSATION              │
│           (orchestrator-huli)               │
└─────────────────────────────────────────────┘
         │           │           │
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │Subagent │ │Subagent │ │Subagent │
    │    A    │ │    B    │ │    C    │
    └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │
         └───────────┴───────────┘
                     │
              Returns summaries
                     │
                     ▼
         ┌─────────────────────┐
         │   MAIN RECEIVES     │
         │   CONSOLIDATED      │
         │   RESULTS           │
         └─────────────────────┘
```

**Key characteristic:** Subagents report **only to parent**, not to each other.

### 2. Agent Teams (Alternative)

Agent Teams work **across separate sessions**. They:

- Have fully independent context windows
- Can message each other directly
- Coordinate via shared task lists
- Each teammate is a separate Claude instance

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Teammate A │◄───►│  Teammate B │◄───►│  Teammate C │
└─────────────┘     └─────────────┘     └─────────────┘
       ▲                   ▲                   ▲
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                  ┌────────┴────────┐
                  │  SHARED TASK    │
                  │     LIST        │
                  └─────────────────┘
```

**Key characteristic:** Teammates can communicate **directly with each other**.

### Comparison

| Aspect | Subagents (Our Choice) | Agent Teams |
|--------|------------------------|-------------|
| Context | Fresh, isolated | Fresh, fully independent |
| Communication | Reports only to parent | Direct messaging |
| Coordination | Parent manages all work | Shared task list |
| Session Scope | Single session | Multiple sessions |
| Token Cost | Lower (summarized results) | Higher (separate instances) |
| Best For | Focused tasks, verbose output | Complex peer discussion |

**Why we chose Subagents:**
- Lower token cost
- Orchestrator maintains control
- Simpler coordination model
- Sufficient for our workflow

---

## Technical Deep Dive: The Task Tool

### How Spawning Works

When an agent invokes the Task tool, Claude Code:

1. **Creates a new context window** - Fresh, isolated from parent
2. **Loads the agent definition** - The `.md` file becomes the system prompt
3. **Grants specified tools** - Only tools listed in the agent's `tools` field
4. **Executes the task** - Agent works independently
5. **Returns summary** - Only the summary enters parent context

```javascript
// Conceptual flow
Task(subagent_type: "architect-vue2", prompt: "Plan: [task]")
    │
    ├── 1. Create new context window
    │
    ├── 2. Load agents/architect-vue2.md as system prompt
    │
    ├── 3. Grant tools: Read, Glob, Grep, Bash
    │
    ├── 4. Execute with prompt: "Plan: [task]"
    │
    ├── 5. Agent produces output (may be large)
    │
    └── 6. Return summary to parent (compressed)
```

### Context Isolation

Each subagent runs in **complete isolation**:

```
┌─────────────────────────────────────────────────────────────┐
│ PARENT CONTEXT                                              │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Conversation history                                     │ │
│ │ User messages                                            │ │
│ │ Tool calls and results                                   │ │
│ │                                                          │ │
│ │ Task(architect-vue2) ─────────────┐                      │ │
│ │                                   │                      │ │
│ │ ◄── Returns: "Plan created..." ◄──┘                      │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ SUBAGENT CONTEXT (architect-vue2)         [ISOLATED]        │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ System prompt: agents/architect-vue2.md                  │ │
│ │ Task: "Plan: [task]"                                     │ │
│ │                                                          │ │
│ │ Read(CLAUDE.md) → 5000 tokens                            │ │
│ │ Grep(patterns) → 2000 tokens                             │ │
│ │ Read(component.vue) → 3000 tokens                        │ │
│ │                                                          │ │
│ │ Total: 10000+ tokens (STAYS HERE)                        │ │
│ │                                                          │ │
│ │ Output: Summarized plan (500 tokens) ─── Returns to ───► │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Benefit:** The 10,000 tokens of file reads stay in the subagent context. Only the 500-token summary returns to the parent.

### Tool Restrictions

Agents can only use tools specified in their definition:

```yaml
---
name: architect-vue2
tools: Read, Glob, Grep, Bash    # Only these 4 tools available
---
```

To allow an agent to spawn subagents, include `Task`:

```yaml
tools: Task, Read, Glob, Grep, Bash
```

To restrict which subagents can be spawned:

```yaml
tools: Task(worker, researcher), Read, Bash   # Only worker and researcher
```

### Transcript Storage

Subagent transcripts are stored separately from the main conversation:

```
~/.claude/projects/{project}/{sessionId}/
├── transcript.jsonl           # Main conversation
└── subagents/
    ├── agent-abc123.jsonl     # Subagent 1 transcript
    ├── agent-def456.jsonl     # Subagent 2 transcript
    └── agent-ghi789.jsonl     # Subagent 3 transcript
```

**Important:** When the main conversation compacts, subagent transcripts are **unaffected**.

---

## Agent Communication Model

### What Subagents CAN Do

1. **Receive task from parent** - Via the Task tool prompt
2. **Use their allowed tools** - Read files, search, execute commands
3. **Return results to parent** - Summary of their work
4. **Be resumed later** - Using their agent ID

### What Subagents CANNOT Do

1. **Communicate with siblings** - No direct agent-to-agent messaging
2. **Spawn other subagents** - No nesting allowed
3. **Access parent's context** - Fresh context only
4. **Share state** - Each invocation starts clean

### Communication Flow Diagram

```
                    USER REQUEST
                         │
                         ▼
              ┌─────────────────────┐
              │  orchestrator-huli  │
              │                     │
              │  "Plan BTU-1666"    │
              └─────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
    ┌───────────┐  ┌───────────┐  ┌───────────┐
    │   btu-    │  │ architect │  │ architect │
    │ estimator │  │   -vue2   │  │   -go     │
    └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
          │              │              │
          │    ┌─────────┴─────────┐    │
          │    │                   │    │
          │    ▼                   ▼    │
          │  ┌───┐               ┌───┐  │
          │  │vue│               │vue│  │
          │  │ 2 │               │ 2 │  │
          │  └─┬─┘               └─┬─┘  │
          │    │                   │    │
          │    └─────────┬─────────┘    │
          │              │              │
          ▼              ▼              ▼
    ┌─────────────────────────────────────────┐
    │         ORCHESTRATOR RECEIVES           │
    │         ALL SUMMARIES                   │
    │                                         │
    │  • Task list from estimator             │
    │  • Vue plans from architect-vue2        │
    │  • Go plan from architect-go            │
    └─────────────────────────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │ reviewer-integration│
              │                     │
              │ Validates all plans │
              └─────────────────────┘
                         │
                         ▼
                  FINAL OUTPUT
```

**Key insight:** All arrows go **through the orchestrator**. Subagents never communicate directly.

---

## Huli Agents Architecture

### Agent Definitions

Each agent is defined in a markdown file with YAML frontmatter:

```yaml
---
name: architect-vue2
description: Vue.js 2.7 architecture specialist
model: opus                          # or sonnet for simpler tasks
tools: Read, Glob, Grep, Bash        # allowed tools
---

# Agent Instructions

[Markdown content becomes the system prompt]
```

### Agent Catalog

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ORCHESTRATOR-HULI                           │
│                     (Entry point, coordinates all)                   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                  │
                 ▼                  ▼                  ▼
         ┌──────────────┐   ┌─────────────────────────────────┐
         │BTU-ESTIMATOR │   │         ARCHITECTS              │
         │              │   │                                 │
         │• Analyzes    │   │ ┌─────────┐ ┌─────────┐        │
         │  requirements│   │ │architect│ │architect│        │
         │• Infers      │   │ │ -vue2   │ │  -go    │        │
         │  hidden work │   │ └─────────┘ └─────────┘        │
         │• Estimates   │   │ ┌─────────┐ ┌─────────┐        │
         │  points      │   │ │architect│ │architect│        │
         │              │   │ │  -php   │ │  -bff   │        │
         └──────────────┘   │ └─────────┘ └─────────┘        │
                            │ ┌─────────┐ ┌─────────┐        │
                            │ │architect│ │architect│        │
                            │ │  -iris  │ │ -mysql  │        │
                            │ └─────────┘ └─────────┘        │
                            └─────────────────────────────────┘
                                           │
                                           ▼
                            ┌─────────────────────────┐
                            │  REVIEWER-INTEGRATION   │
                            │                         │
                            │  • Validates contracts  │
                            │  • Checks data flow     │
                            │  • Confirms deps        │
                            └─────────────────────────┘
```

### Workflow Sequence

```
Step 1: ESTIMATION
┌────────────────────────────────────────────────────────────────┐
│ orchestrator-huli                                              │
│                                                                │
│   Task(btu-estimator, "Analyze BTU-1666 requirements")         │
│         │                                                      │
│         └──► btu-estimator                                     │
│                  │                                             │
│                  ├── Reads requirements                        │
│                  ├── Infers hidden work (BE, DB)               │
│                  └── Returns: Task list with 5 tasks           │
│                         │                                      │
│         ◄───────────────┘                                      │
└────────────────────────────────────────────────────────────────┘

Step 2: ARCHITECTURE (Parallel)
┌────────────────────────────────────────────────────────────────┐
│ orchestrator-huli                                              │
│                                                                │
│   For each task, delegate to appropriate architect:            │
│                                                                │
│   Task(architect-mysql, "Task 1: Add index")                   │
│   Task(architect-go, "Task 2: Create endpoint")                │
│   Task(architect-vue2, "Task 3: Create somatometry")           │
│   Task(architect-vue2, "Task 4: Iterate appointments")         │
│   Task(architect-vue2, "Task 5: UX improvements")              │
│         │                                                      │
│         └──► Each architect works in isolation                 │
│                  │                                             │
│                  ├── Reads codebase (CLAUDE.md, patterns)      │
│                  ├── Designs implementation                    │
│                  └── Returns: Detailed plan                    │
│                         │                                      │
│         ◄───────────────┘                                      │
└────────────────────────────────────────────────────────────────┘

Step 3: INTEGRATION REVIEW
┌────────────────────────────────────────────────────────────────┐
│ orchestrator-huli                                              │
│                                                                │
│   Task(reviewer-integration, "Review all plans")               │
│         │                                                      │
│         └──► reviewer-integration                              │
│                  │                                             │
│                  ├── Validates API contracts                   │
│                  ├── Checks field naming consistency           │
│                  ├── Verifies dependency order                 │
│                  └── Returns: Validation report                │
│                         │                                      │
│         ◄───────────────┘                                      │
└────────────────────────────────────────────────────────────────┘

Step 4: CONSOLIDATION
┌────────────────────────────────────────────────────────────────┐
│ orchestrator-huli                                              │
│                                                                │
│   Combines all outputs into final plan:                        │
│   • Task list with dependencies                                │
│   • Implementation details per repo                            │
│   • Validation results                                         │
│   • Execution order                                            │
│                                                                │
│   Returns: Complete BTU plan                                   │
└────────────────────────────────────────────────────────────────┘
```

---

## Real Example: BTU-1666

### Input

```
BTU-1666: Patient Homepage V2 - Somatometría

Requirements:
- Display vital signs from last 6 consultations
- Evolutionary chart visualization
- Click navigates to source consultation
- Iterate appointments block (empty state, 3 types)
- Various UX V1→V2 improvements
```

### Agent Execution Trace

#### 1. btu-estimator (First)

**Context:** Fresh, no conversation history
**Tools used:** Read (BTU requirements), WebFetch (none needed)
**Token usage:** ~2,000 tokens

**Process:**
```
1. Read requirements
2. Identify 3 explicit objectives:
   - Somatometry section
   - Appointments iteration
   - UX improvements

3. INFER hidden work:
   ├── "Display vital signs from last 6 consultations"
   │    └── Question: Where does this data come from?
   │    └── Answer: Need BE endpoint to aggregate
   │
   ├── "Aggregate query on patient_checkup"
   │    └── Question: Will this be performant?
   │    └── Answer: Need index for ORDER BY created_at DESC LIMIT 6
   │
   └── Result: 2 additional tasks inferred

4. Output task list with 5 tasks
```

**Output returned to orchestrator:**
```
| # | Repo         | Task                              | Pts |
|---|--------------|-----------------------------------|-----|
| 1 | db           | Add index for history query       | 1   |
| 2 | ehr          | Create vital-signs history API    | 5   |
| 3 | practice-web | Create somatometry section        | 8   |
| 4 | practice-web | Iterate appointments block        | 5   |
| 5 | practice-web | Apply UX V1-V2 improvements       | 5   |

Total: 24 pts | Inferred: Tasks 1 & 2
```

#### 2. architect-mysql

**Context:** Fresh, only sees Task 1
**Tools used:** Grep (search for schema), Read (check existing indexes)
**Token usage:** ~3,000 tokens

**Process:**
```
1. Search for patient_checkup table:
   Grep("patient_checkup", mysql/sql/)

2. Find existing indexes:
   - fk_patient_checkup_patient_file_idx (id_patient_file)
   - ix_patient_checkup_record_created_on_idx (created_on)

3. Identify gap:
   - No composite index for (id_patient_file, created_on DESC)
   - This query pattern needs covering index

4. Design migration:
   - Non-blocking ALTER TABLE
   - Include rollback script
```

**Output returned to orchestrator:**
```sql
ALTER TABLE `hh`.`patient_checkup`
ADD INDEX `ix_patient_checkup_patient_history`
(`id_patient_file`, `created_on` DESC);
```

#### 3. architect-go

**Context:** Fresh, only sees Task 2
**Tools used:** Glob, Grep, Read
**Token usage:** ~8,000 tokens (read CLAUDE.md, existing handlers, proto files)

**Process:**
```
1. Read ehr/CLAUDE.md for patterns
2. Find similar handlers in internal/service/handler/
3. Check existing proto definitions for vital signs
4. Design new endpoint following patterns:
   - Proto message definition
   - Handler structure
   - Repository query
```

**Output returned to orchestrator:**
```
Proto: GetVitalSignsHistoryRequest/Response
Handler: internal/service/handler/patient/vital-signs-history/
Query: SELECT ... FROM patient_checkup WHERE ... ORDER BY ... LIMIT
```

#### 4. architect-vue2 (x3 instances)

**Context:** Fresh for each task
**Tools used:** Glob, Grep, Read
**Token usage:** ~6,000 tokens each

**Process (Task 3 - Somatometry):**
```
1. Read practice-web/CLAUDE.md
2. Find similar components:
   - patient-consents_component.vue (DataMixin pattern)
   - development-curves-chart_component.vue (Chart.js)
3. Design new components:
   - patient-somatometry_component.vue
   - vital-sign-card_component.vue
```

**Process (Task 4 - Appointments):**
```
1. Read patient-next-appointment_component.vue
2. Identify changes:
   - Add empty state with action button
   - Refactor for 3 appointment types
   - Modify data loading
```

**Process (Task 5 - UX):**
```
1. Read each affected component
2. Map specific changes:
   - Emergency contacts: v-for instead of single
   - Files: window.open() instead of router
   - Consents: Add tooltips
```

#### 5. reviewer-integration (Last)

**Context:** Fresh, receives all architect outputs via orchestrator prompt
**Tools used:** Read (verify file paths exist)
**Token usage:** ~4,000 tokens

**Process:**
```
1. Validate API contract ehr ↔ practice-web:
   - Field: checkupId - BE has it? ✓
   - Field: recordedAt - Format matches? ✓
   - All 8 vital signs present? ✓

2. Check naming consistency:
   - Proto: snake_case
   - JS: camelCase
   - Issue: Minor - handle in FE transformation

3. Verify dependencies:
   - Task 1 → Task 2 → Task 3 ✓
   - Tasks 4, 5 independent ✓
```

**Output returned to orchestrator:**
```
✅ API contract validated
✅ All fields present
✅ Dependencies correct
⚠️ Minor: snake_case → camelCase transformation needed
```

### Final Consolidated Output

The orchestrator combined all outputs:

```
BTU-1666: Patient Homepage V2 - Somatometría

Total: 24 pts | 5 tasks | 3 repos

| # | Repo         | Task                              | Pts | Deps |
|---|--------------|-----------------------------------|-----|------|
| 1 | db           | Add index for history query       | 1   | -    |
| 2 | ehr          | Create vital-signs history API    | 5   | 1    |
| 3 | practice-web | Create somatometry section        | 8   | 2    |
| 4 | practice-web | Iterate appointments block        | 5   | -    |
| 5 | practice-web | Apply UX V1-V2 improvements       | 5   | -    |

Inferred Work: Tasks 1 & 2 (not in original BTU)
Integration: Validated ✅
```

---

## Observations: Theory vs Practice

### Did Agents Follow the Documentation?

| Documentation Says | What Actually Happened | Verdict |
|-------------------|------------------------|---------|
| Fresh context per subagent | Each agent started clean, no shared history | ✅ Confirmed |
| Only reports to parent | All outputs returned to orchestrator only | ✅ Confirmed |
| Cannot spawn other subagents | No agent tried to spawn children | ✅ Confirmed |
| Tools restricted to definition | Each agent only used its allowed tools | ✅ Confirmed |
| Summary returned, verbose stays | Large file reads stayed in subagent context | ✅ Confirmed |

### Did Agents Iterate With Each Other?

**No.** According to documentation and observed behavior:

- Agents **did not communicate directly** with each other
- Agents **did not iterate** or refine each other's work
- All coordination happened **through the orchestrator**

This is by design. The subagent model is:
```
Parent → Subagent → Parent (receive results) → Next Subagent → Parent
```

Not:
```
Subagent A ←→ Subagent B (direct communication)
```

### How Context Was Managed

| Agent | Tokens Used | Tokens Returned to Parent |
|-------|-------------|---------------------------|
| btu-estimator | ~2,000 | ~500 (task list) |
| architect-mysql | ~3,000 | ~300 (migration script) |
| architect-go | ~8,000 | ~800 (plan summary) |
| architect-vue2 (x3) | ~18,000 total | ~1,500 total |
| reviewer-integration | ~4,000 | ~400 (validation) |
| **Total subagent** | ~35,000 | ~3,500 returned |

**Benefit:** 35,000 tokens of detailed analysis, but only 3,500 tokens entered the main conversation context. This is the key advantage of the subagent model.

### What Would Improve With Agent Teams?

If we used Agent Teams instead of Subagents:

| Scenario | Subagents (Current) | Agent Teams |
|----------|---------------------|-------------|
| architect-go needs to ask architect-mysql about schema | Must go through orchestrator | Direct message possible |
| reviewer finds issue, needs architect to fix | New orchestrator task | Direct collaboration |
| Complex multi-repo refactor | Sequential coordination | Parallel peer discussion |

**Trade-off:** Agent Teams would enable richer collaboration but at higher token cost (each teammate is a full Claude instance).

### Conclusion

The subagent model worked well for our use case:

1. **Clear delegation** - Each specialist handled their domain
2. **Context isolation** - Large outputs stayed contained
3. **Orchestrator control** - Single point of coordination
4. **Predictable flow** - Sequential steps with parallel potential

For more complex scenarios requiring agent-to-agent iteration, Agent Teams would be the better choice.
