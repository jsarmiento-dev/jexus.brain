---
name: subagent-driven-development
description: "Execute plans via delegate_task subagents (2-stage review)."
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [delegation, subagent, implementation, workflow, parallel]
    related_skills: [writing-plans, requesting-code-review, test-driven-development]
---

# Subagent-Driven Development

## Overview

Execute implementation plans by dispatching fresh subagents per task with systematic two-stage review.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration.

## When to Use

Use this skill when:
- You have an implementation plan (from writing-plans skill or user requirements)
- Tasks are mostly independent
- Quality and spec compliance are important
- You want automated review between tasks

**vs. manual execution:**
- Fresh context per task (no confusion from accumulated state)
- Automated review process catches issues early
- Consistent quality checks across all tasks
- Subagents can ask questions before starting work

## The Process

### 1. Read and Parse Plan

Read the plan file. Extract ALL tasks with their full text and context upfront. Create a todo list:

```python
# Read the plan
read_file("docs/plans/feature-plan.md")

# Create todo list with all tasks
todo([
    {"id": "task-1", "content": "Create User model with email field", "status": "pending"},
    {"id": "task-2", "content": "Add password hashing utility", "status": "pending"},
    {"id": "task-3", "content": "Create login endpoint", "status": "pending"},
])
```

**Key:** Read the plan ONCE. Extract everything. Don't make subagents read the plan file — provide the full task text directly in context.

### 2. Per-Task Workflow

For EACH task in the plan:

#### Step 1: Dispatch Implementer Subagent

Use `delegate_task` with complete context:

```python
delegate_task(
    goal="Implement Task 1: Create User model with email and password_hash fields",
    context="""
    TASK FROM PLAN:
    - Create: src/models/user.py
    - Add User class with email (str) and password_hash (str) fields
    - Use bcrypt for password hashing
    - Include __repr__ for debugging

    FOLLOW TDD:
    1. Write failing test in tests/models/test_user.py
    2. Run: pytest tests/models/test_user.py -v (verify FAIL)
    3. Write minimal implementation
    4. Run: pytest tests/models/test_user.py -v (verify PASS)
    5. Run: pytest tests/ -q (verify no regressions)
    6. Commit: git add -A && git commit -m "feat: add User model with password hashing"

    PROJECT CONTEXT:
    - Python 3.11, Flask app in src/app.py
    - Existing models in src/models/
    - Tests use pytest, run from project root
    - bcrypt already in requirements.txt
    """,
    toolsets=['terminal', 'file']
)
```

#### Step 2: Dispatch Spec Compliance Reviewer

After the implementer completes, verify against the original spec:

```python
delegate_task(
    goal="Review if implementation matches the spec from the plan",
    context="""
    ORIGINAL TASK SPEC:
    - Create src/models/user.py with User class
    - Fields: email (str), password_hash (str)
    - Use bcrypt for password hashing
    - Include __repr__

    CHECK:
    - [ ] All requirements from spec implemented?
    - [ ] File paths match spec?
    - [ ] Function signatures match spec?
    - [ ] Behavior matches expected?
    - [ ] Nothing extra added (no scope creep)?

    OUTPUT: PASS or list of specific spec gaps to fix.
    """,
    toolsets=['file']
)
```

**If spec issues found:** Fix gaps, then re-run spec review. Continue only when spec-compliant.

#### Step 3: Dispatch Code Quality Reviewer

After spec compliance passes:

```python
delegate_task(
    goal="Review code quality for Task 1 implementation",
    context="""
    FILES TO REVIEW:
    - src/models/user.py
    - tests/models/test_user.py

    CHECK:
    - [ ] Follows project conventions and style?
    - [ ] Proper error handling?
    - [ ] Clear variable/function names?
    - [ ] Adequate test coverage?
    - [ ] No obvious bugs or missed edge cases?
    - [ ] No security issues?

    OUTPUT FORMAT:
    - Critical Issues: [must fix before proceeding]
    - Important Issues: [should fix]
    - Minor Issues: [optional]
    - Verdict: APPROVED or REQUEST_CHANGES
    """,
    toolsets=['file']
)
```

**If quality issues found:** Fix issues, re-review. Continue only when approved.

#### Step 4: Mark Complete

```python
todo([{"id": "task-1", "content": "Create User model with email field", "status": "completed"}], merge=True)
```

### 3. Final Review

After ALL tasks are complete, dispatch a final integration reviewer:

```python
delegate_task(
    goal="Review the entire implementation for consistency and integration issues",
    context="""
    All tasks from the plan are complete. Review the full implementation:
    - Do all components work together?
    - Any inconsistencies between tasks?
    - All tests passing?
    - Ready for merge?
    """,
    toolsets=['terminal', 'file']
)
```

### 4. Verify and Commit

```bash
# Run full test suite
pytest tests/ -q

# Review all changes
git diff --stat

# Final commit if needed
git add -A && git commit -m "feat: complete [feature name] implementation"
```

## Task Granularity

**Each task = 2-5 minutes of focused work.**

**Too big:**
- "Implement user authentication system"

**Right size:**
- "Create User model with email and password fields"
- "Add password hashing function"
- "Create login endpoint"
- "Add JWT token generation"
- "Create registration endpoint"

## Red Flags — Never Do These

- Start implementation without a plan
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed critical/important issues
- Dispatch multiple implementation subagents for tasks that touch the same files
- Make subagent read the plan file (provide full text in context instead)
- Skip scene-setting context (subagent needs to understand where the task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance
- Skip review loops (reviewer found issues → implementer fixes → review again)
- Let implementer self-review replace actual review (both are needed)
- **Start code quality review before spec compliance is PASS** (wrong order)
- Move to next task while either review has open issues

## Handling Issues

### If Subagent Asks Questions

- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

### If Reviewer Finds Issues

- Implementer subagent (or a new one) fixes them
- Reviewer reviews again
- Repeat until approved
- Don't skip the re-review

### If Subagent Fails a Task

- Dispatch a new fix subagent with specific instructions about what went wrong
- Don't try to fix manually in the controller session (context pollution)

## Efficiency Notes

**Why fresh subagent per task:**
- Prevents context pollution from accumulated state
- Each subagent gets clean, focused context
- No confusion from prior tasks' code or reasoning

**Why two-stage review:**
- Spec review catches under/over-building early
- Quality review ensures the implementation is well-built
- Catches issues before they compound across tasks

**Cost trade-off:**
- More subagent invocations (implementer + 2 reviewers per task)
- But catches issues early (cheaper than debugging compounded problems later)

## Alternative: Background Cursor Delegation

For projects that follow the **SDD lifecycle** (spec-first, multi-phase, autonomous execution), prefer the pattern in `sdd-goal-orchestration` over this skill's per-task `delegate_task()` approach:

| Aspect | This Skill | sdd-goal-orchestration |
|--------|-----------|----------------------|
| Mechanism | `delegate_task()` | `terminal(background=true)` to Cursor CLI |
| Scope | Single implementation task | Full project lifecycle (R&D → Design → Implement → Deploy) |
| Review | 2-stage per task | Phase-gated (verify at phase boundaries) |
| Autonomy | Hermes decides task order | User sets goal, Hermes decides everything |
| Parallelism | Subagents per task | Full project delegated to Cursor |

Use `sdd-goal-orchestration` when: building a new project from scratch, the user provides only a goal, or the project has multiple phases (research, design, implementation, deployment).

Use this skill (subagent-driven-development) when: executing a well-defined, bounded implementation plan with precise tasks.

## Integration with Other Skills

### With writing-plans

This skill EXECUTES plans created by the writing-plans skill:
1. User requirements → writing-plans → implementation plan
2. Implementation plan → subagent-driven-development → working code

### With test-driven-development

Implementer subagents should follow TDD:
1. Write failing test first
2. Implement minimal code
3. Verify test passes
4. Commit

Include TDD instructions in every implementer context.

### With requesting-code-review

The two-stage review process IS the code review. For final integration review, use the requesting-code-review skill's review dimensions.

### With systematic-debugging

If a subagent encounters bugs during implementation:
1. Follow systematic-debugging process
2. Find root cause before fixing
3. Write regression test
4. Resume implementation

## Analysis-First Workflow (Token Saver)

Before creating any implementation plan, **delegate analysis to a subagent first** when the task involves understanding existing code or architecture. This is the user's explicit preference to save tokens.

### When to use analysis-first
- You need to understand an existing codebase before planning changes
- Migrating monolithic HTML to React (extract components, API endpoints, data types)
- Auditing code quality across multiple modules
- Any task where "understand what exists" is a prerequisite to "build the new thing"

### The pattern

```python
# Step 1: Delegate analysis (uses cheap model, saves Hermes tokens)
analysis_result = delegate_task(
    goal="Analyze [X] and produce structured report covering [specific topics]",
    context="""
    Include in your analysis:
    1. Component inventory from existing HTML
    2. API endpoints referenced
    3. Data types inferred
    4. CSS variables and design tokens
    5. UI states handled (loading, empty, error, edge cases)
    6. Proposed component tree
    7. Architecture recommendations
    
    Be exhaustive — the parent agent will use this to create the implementation plan.
    """,
    toolsets=["terminal", "file"]  # read-only
)

# Step 2: Save analysis report for reference
write_file("openspec/analysis/topic.md", analysis_report)

# Step 3: Create the implementation plan
# (Hermes does this — cheap since the heavy file reading was done by the subagent)
```

### Benefits over diving straight in
- Subagent runs on cheap models by default (deepseek-v4-flash)
- Hermes only pays for creating the SDD/plan from the structured report
- Analysis is saved as a reference document for future sessions
- The user said "primeramente delegando a cursor el analisis para no quemar tokens"

### Pitfalls
- Give the subagent VERY detailed context (exact file paths, specific questions to answer)
- Require the full report in the summary, not a file write (or you won't see the details)
- The subagent context is the output of this step — make it count

Use this pattern when you need to **analyze multiple codebases, modules, or concerns in parallel** for code quality, anti-patterns, SOLID violations, security issues, or architecture review. Each subagent owns one scope and returns a structured report — the parent agent consolidates all results.

**Core principle:** One subagent per project/module/concern. All read independently. Parent consolidates.

### When to Use

- User asks "review this project for bad patterns" or "analyze code quality"
- User wants a cross-project audit (frontend + backend simultaneously)
- User wants parallel analysis of multiple concerns (security + architecture + performance)
- Before a major refactoring effort to understand what needs fixing

### The Pattern

#### Step 1: Map the Scope

Identify independent analysis units:
- One subagent per project (e.g., `jtrading-api`, `jtrading-app`)
- One subagent per concern (e.g., one for security, one for architecture, one for performance)
- One subagent per module for very large codebases

#### Step 2: Dispatch Parallel Analysis Subagents

```python
delegate_task(
    tasks=[
        {
            "goal": "Analyze the frontend project for SOLID violations, anti-patterns, clean code issues, security problems, and architecture problems. Return a comprehensive structured document.",
            "context": """
            PROJECT: jtrading-app (Angular 17 standalone)
            PATH: /home/ubuntu/projects/frontend/
            STRUCTURE:
            - src/app/core/services/* — services
            - src/app/features/* — feature modules
            - src/app/layout/* — shared components
            
            TECHNOLOGIES: Angular 17, TypeScript, RxJS, Tailwind CSS
            
            ANALYSIS CRITERIA:
            1. Read EVERY .ts file
            2. Check SOLID violations (S: SRP, O: OCP, L: LSP, I: ISP, D: DIP)
            3. Check anti-patterns (God Objects, dead code, magic numbers, mixed architectures)
            4. Check code smells (commented code, console.log, long functions, any types)
            5. Check security issues (hardcoded secrets, exposed errors, missing validation)
            6. Check architecture problems (coupling, missing abstractions, wrong patterns)
            
            For each finding: file, line, why it's a problem, and proposed solution.
            Classify as CRITICAL, IMPORTANT, or SUGGESTION.
            Return the FULL structured document in your response summary.
            """,
            "toolsets": ["terminal", "file"]
        },
        {
            "goal": "Analyze the backend project for SOLID violations, anti-patterns, clean code issues, security problems, and architecture problems.",
            "context": """... (similar, but for backend) ...""",
            "toolsets": ["terminal", "file"]
        }
    ]
)
```

#### Step 3: Consolidate Results

The parent agent receives both analysis documents and consolidates:
- Overall project health summary
- Cross-cutting concerns that affect both projects
- Prioritized remediation plan organized into phases

```python
# Create consolidated todo list based on all findings
todo([
    {"id": "phase-1", "content": "Security & Stability fixes", "status": "pending"},
    {"id": "phase-2", "content": "Architecture improvements", "status": "pending"},
    {"id": "phase-3", "content": "Quality & testing", "status": "pending"},
])
```

### Critical Rules for Analysis Subagents

1. **Subagent MUST return all findings in its `summary`** — do not make it write to a file. The parent agent needs the full text to consolidate.
2. **Provide exact file paths** in context so the subagent can `read_file` efficiently.
3. **Always include file listing commands** so the subagent knows the full scope: `find src -type f -name '*.ts'`.
4. **Specify the exact analysis criteria** — don't say "review the code", say "check SOLID, anti-patterns, clean code, security, architecture".
5. **Ask for severity classification** (CRITICAL / IMPORTANT / SUGGESTION) so you can phase the work.
6. **Set a high limit** on the number of files the subagent reads — some projects have 30+ files.

### Pitfalls

- **Subagent returns "summarized" findings by writing to a file** — your context never sees the details. Always require the full report in the `summary`/response.
- **Too large a project for one subagent** — split by module or concern. Each subagent's token budget is limited.
- **Subagent doesn't read all files** — include `find` commands in context so it knows how many files to expect.
- **Solitary vs structural problems** — a CRITICAL security issue (hardcoded password) can coexist with a project that is architecturally sound. Always classify by severity, not just count.

---

## Multi-Agent Refactoring & Remediation (Parallel)

After analysis is complete, use this pattern to **execute fixes in parallel phases** across multiple projects. Each phase is a batch of changes dispatched via `delegate_task(tasks=[...])`. All subagents in a phase run simultaneously.

**Core principle:** Phase by severity. Parallel within a phase. Subagents get exact, copy-pasteable instructions.

### When to Use

- After a code analysis has identified prioritized issues
- User says "proceed with Phase 1" or "fix the critical issues first"
- You have a clear remediation plan with specific file-by-file changes

### The Phases

#### Phase 1: Security & Stability (CRITICAL issues first)
Fix hardcoded secrets, missing error handlers, unvalidated config, exposed internals.

**Subagent context structure — be extremely specific about what to change:**

```python
delegate_task(
    tasks=[
        {
            "goal": "Execute Phase 1 security fixes for the backend API",
            "context": """
            All files at /home/ubuntu/projects/backend/
            
            CHANGE 1: seed.ts — Move hardcoded password to env var
            File: src/prisma/seed.ts
            - Change 'hardcoded_password_here' → process.env.ADMIN_PASSWORD || 'changeme_in_production'
            
            CHANGE 2: app.ts — Add helmet and global error handler
            File: src/app.ts
            - Import helmet
            - Add app.use(helmet()) after cors
            - Add global error middleware at end:
            ```typescript
            app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
              console.error('[ErrorHandler]', err.message);
              res.status(500).json({ message: 'Internal server error' });
            });
            ```
            
            CHANGE 3: config/redis.ts — Validate REDIS_URL
            File: src/config/redis.ts
            - If REDIS_URL is missing, throw error instead of using fallback
            
            Read each file BEFORE modifying it.
            Commit all changes: git add -A && git commit -m "FASE 1: Security fixes"
            """,
            "toolsets": ["terminal", "file"]
        },
        {
            "goal": "Execute Phase 1 security fixes for the frontend",
            "context": """... (similar) ...""",
            "toolsets": ["terminal", "file"]
        }
    ]
)
```

#### Phase 2: Architecture (IMPORTANT issues)
asyncHandler, repository pattern, RESTful routes, DI, service decomposition, interfaces.

#### Phase 3: Quality (SUGGESTION issues)
Tests, logger, dead code removal, naming consistency.

### Rules for Remediation Subagents

1. EXACT instructions - don't say fix security issues. Say Change line 10 of seed.ts from X to Y.
2. Tell the subagent to read the file first - it needs fresh state, not stale cached data from your context.
3. Group related changes into one subagent (all controller asyncHandlers in one task, all PrismaClient singletons in another).
4. Include the EXACT code to write - copy-pasteable blocks in the context.
5. Require a commit at the end so each phase is individually revertible.
6. One subagent per project, not one subagent per file - parallel efficiency.

### Red Flags

- Vague instructions (fix the architecture) - subagent wastes time figuring out what to do. Be specific.
- Skipping phase ordering - don't fix naming conventions before fixing security holes.
- Cross-project dependencies - if API route changes affect frontend API calls, handle them in the same phase.
- Not telling the subagent to read files first - it may operate on stale cache from your context and reintroduce bugs.

---

## Sprint Execution Mode

Use this mode when a multi-sprint implementation plan (from writing-plans) needs to be executed. Each sprint is dispatched as ONE subagent with ALL its tasks bundled together, using the subagent's judgment to order work within the sprint. No 2-stage review per task - review happens at sprint boundaries.

**Core principle:** One subagent per sprint. Bundle all tasks. Verify build after each sprint.

### When to Use

- Executing a phased implementation plan (4 phases, 10 sprints)
- Each sprint's tasks touch related files (no need to isolate per file)
- The plan already has exact task descriptions, effort estimates, and file paths
- Build verification at sprint boundaries is sufficient quality control

### When NOT to Use (use the default Sequential Implementation mode instead)

- Tasks are experimental or exploratory (need spec compliance check per task)
- Each task has unique failure modes that need individual review
- The subagent is expected to make design decisions during implementation
- The project lacks build tooling (no type-checker, no linter, no test suite)
### The Pattern

#### Step 1: Inventory the Sprint

Pull the sprint table from the plan. Identify dependencies within the sprint:

```python
# Sprint 1 tasks from the plan:
# 1.1 Create status enum (30min, no deps)
# 1.2 Add ownership validation (1h, no deps)
# 1.3 Encapsulate polling service (30min, no deps)
# 1.4 Unify JWT expiration (15min, no deps)
# All independent -> can be bundled in one subagent
```

#### Step 2: Dispatch ONE Subagent per Sprint

```python
delegate_task(
    goal="Execute ALL tasks from Sprint 1 of the implementation plan. Read each file before modifying it. Commit at the end with message 'Sprint 1: summary'.",
    context="
    PROJECT: /home/ubuntu/projects/jtrading-api
    BUILD COMMAND: cd /home/ubuntu/projects/jtrading-api && npx tsc --noEmit
    
    SPRINT 1 TASKS:
    
    1.1 Create status enum
    Files: prisma/schema.prisma, src/modules/strategy/dto/strategy.dto.ts
    - Add enum StrategyStatus { RUNNING, PAUSED, STOPPED } in schema.prisma
    - Change status String to status StrategyStatus @default(STOPPED)
    - Add STRATEGY_STATUSES constant and StrategyStatusValue type in dto
    - Update Zod schema to z.enum(['RUNNING', 'PAUSED', 'STOPPED'])
    
    1.2 Add ownership validation
    Files: src/modules/strategy/strategy.controller.ts
    - Add validateAccountOwnership() helper using prisma
    - Validate in create, update, delete handlers
    
    1.3 Encapsulate polling service
    Files: src/services/websocket/polling.service.ts, src/main.ts
    - Convert to class with private currentSymbols
    - Create singleton export
    - Update main.ts import
    
    IMPORTANT: Read each file BEFORE modifying it. Run build command at the end.
    Commit all changes: git add -A && git commit -m 'Sprint 1: status enum, ownership validation, encapsulate polling'
    ",
    toolsets=['terminal', 'file']
)
```

#### Step 3: Verify Build After Each Sprint

After the subagent returns, verify the build:

```bash
npx tsc --noEmit
```

If the build fails, read the error output and dispatch a fix subagent with the specific error message. Do NOT fix in the parent context (context pollution).

### Sprint Sequence Governance

```python
# Phase 1: Sprint 1 + Sprint 2
tasks = [
    {'goal': 'Execute Sprint 1...', 'context': '...'},
    {'goal': 'Execute Sprint 2...', 'context': '...'},
]
results = delegate_task(tasks=tasks)

# Phase 2 starts ONLY after Phase 1 exits
# (cross-phase dependencies exist)
```

**Important:** If sprints within a phase have no cross-dependencies, dispatch them in parallel via delegate_task(tasks=[...]). If Sprint 2 depends on Sprint 1 (files were modified by Sprint 1 that Sprint 2 needs), dispatch them sequentially.

### Pitfalls

- **Build breaks across sprints** - always verify build after each sprint. A subagent may leave the project in a broken intermediate state if the code it wrote references types not yet created.
- **Naming conflicts with generated code** - Prisma enum names may collide with your custom types. Rename the DTO type (e.g., StrategyStatusValue instead of StrategyStatus) to avoid collision with the Prisma-generated type of the same name.
- **Subagent model constraints** - a cheap model (e.g. deepseek-v4-flash) can handle mechanical refactoring well (find-and-replace, add imports, convert to class) but may miss nuanced type errors. Always run the build after the subagent finishes.
- **Direct DB operations in subagent** - if a sprint includes a database migration (npx prisma migrate), the subagent may not have DATABASE_URL configured. Prefer schema-only changes (prisma generate) over full migrations in subagent tasks.
- **Stale cache on sibling files** - if the subagent modifies 3 files but a 4th file has a stale reference (e.g., an import of a now-renamed type), the build will fail. This is expected - fix in the next sprint or dispatch a quick fix subagent.

---

## Remember

```
Fresh subagent per task
Two-stage review every time
Spec compliance FIRST
Code quality SECOND
Never skip reviews
Catch issues early
```

**Quality is not an accident. It's the result of systematic process.**

## Further reading (load when relevant)

When the orchestration involves significant context usage, long review loops, or complex validation checkpoints, load these references for the specific discipline:

- **`references/context-budget-discipline.md`** — Four-tier context degradation model (PEAK / GOOD / DEGRADING / POOR), read-depth rules that scale with context window size, and early warning signs of silent degradation. Load when a run will clearly consume significant context (multi-phase plans, many subagents, large artifacts).
- **`references/gates-taxonomy.md`** — The four canonical gate types (Pre-flight, Revision, Escalation, Abort) with behavior, recovery, and examples. Load when designing or reviewing any workflow that has validation checkpoints — use the vocabulary explicitly so each gate has defined entry, failure behavior, and resumption rules.
- **`references/code-analysis-criteria.md`** — Reusable checklists for SOLID violations, anti-patterns, code smells, security issues, and severity classification. Load when dispatching analysis subagents — copy-paste relevant sections into the goals.
- **`references/theme-token-migration.md`** — Systematic pattern for migrating a codebase from hardcoded dark/light colors to CSS theme tokens using parallel subagents. Includes discovery commands, token mapping table format, and verification steps. Load when doing large-scale UI theme refactoring across 30+ files.

References adapted from gsd-build/get-shit-done (MIT © 2025 Lex Christopherson).
