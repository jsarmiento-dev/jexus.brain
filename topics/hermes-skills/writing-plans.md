---
name: writing-plans
description: "Write implementation plans: bite-sized tasks, paths, code."
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [planning, design, implementation, workflow, documentation]
    related_skills: [subagent-driven-development, test-driven-development, requesting-code-review]
---

# Writing Implementation Plans

## Overview

Write comprehensive implementation plans assuming the implementer has zero context for the codebase and questionable taste. Document everything they need: which files to touch, complete code, testing commands, docs to check, how to verify. Give them bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume the implementer is a skilled developer but knows almost nothing about the toolset or problem domain. Assume they don't know good test design very well.

**Core principle:** A good plan makes implementation obvious. If someone has to guess, the plan is incomplete.

## When to Use

**Always use before:**
- Implementing multi-step features
- Breaking down complex requirements
- Delegating to subagents via subagent-driven-development

**Don't skip when:**
- Feature seems simple (assumptions cause bugs)
- You plan to implement it yourself (future you needs guidance)
- Working alone (documentation matters)

## Bite-Sized Task Granularity

**Each task = 2-5 minutes of focused work.**

Every step is one action:
- "Write the failing test" — step
- "Run it to make sure it fails" — step
- "Implement the minimal code to make the test pass" — step
- "Run the tests and make sure they pass" — step
- "Commit" — step

**Too big:**
```markdown
### Task 1: Build authentication system
[50 lines of code across 5 files]
```

**Right size:**
```markdown
### Task 1: Create User model with email field
[10 lines, 1 file]

### Task 2: Add password hash field to User
[8 lines, 1 file]

### Task 3: Create password hashing utility
[15 lines, 1 file]
```

## Plan Document Structure

### Header (Required)

Every plan MUST start with:

```markdown
# [Feature Name] Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

### Task Structure

Each task follows this format:

````markdown
### Task N: [Descriptive Name]

**Objective:** What this task accomplishes (one sentence)

**Files:**
- Create: `exact/path/to/new_file.py`
- Modify: `exact/path/to/existing.py:45-67` (line numbers if known)
- Test: `tests/path/to/test_file.py`

**Step 1: Write failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**Step 2: Run test to verify failure**

Run: `pytest tests/path/test.py::test_specific_behavior -v`
Expected: FAIL — "function not defined"

**Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

**Step 4: Run test to verify pass**

Run: `pytest tests/path/test.py::test_specific_behavior -v`
Expected: PASS

**Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## Writing Process

### Step 1: Understand Requirements

Read and understand:
- Feature requirements
- Design documents or user description
- Acceptance criteria
- Constraints

### Step 2: Explore the Codebase

Use Hermes tools to understand the project:

```python
# Understand project structure
search_files("*.py", target="files", path="src/")

# Look at similar features
search_files("similar_pattern", path="src/", file_glob="*.py")

# Check existing tests
search_files("*.py", target="files", path="tests/")

# Read key files
read_file("src/app.py")
```

### Step 3: Design Approach

Decide:
- Architecture pattern
- File organization
- Dependencies needed
- Testing strategy

### Step 4: Write Tasks

Create tasks in order:
1. Setup/infrastructure
2. Core functionality (TDD for each)
3. Edge cases
4. Integration
5. Cleanup/documentation

### Step 5: Add Complete Details

For each task, include:
- **Exact file paths** (not "the config file" but `src/config/settings.py`)
- **Complete code examples** (not "add validation" but the actual code)
- **Exact commands** with expected output
- **Verification steps** that prove the task works

### Step 6: Review the Plan

Check:
- [ ] Tasks are sequential and logical
- [ ] Each task is bite-sized (2-5 min)
- [ ] File paths are exact
- [ ] Code examples are complete (copy-pasteable)
- [ ] Commands are exact with expected output
- [ ] No missing context
- [ ] DRY, YAGNI, TDD principles applied

### Step 7: Save the Plan

```bash
mkdir -p docs/plans
# Save plan to docs/plans/YYYY-MM-DD-feature-name.md
git add docs/plans/
git commit -m "docs: add implementation plan for [feature]"
```

## Principles

### DRY (Don't Repeat Yourself)

**Bad:** Copy-paste validation in 3 places
**Good:** Extract validation function, use everywhere

### YAGNI (You Aren't Gonna Need It)

**Bad:** Add "flexibility" for future requirements
**Good:** Implement only what's needed now

```python
# Bad — YAGNI violation
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
        self.preferences = {}  # Not needed yet!
        self.metadata = {}     # Not needed yet!

# Good — YAGNI
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
```

### TDD (Test-Driven Development)

Every task that produces code should include the full TDD cycle:
1. Write failing test
2. Run to verify failure
3. Write minimal code
4. Run to verify pass

See `test-driven-development` skill for details.

### Frequent Commits

Commit after every task:
```bash
git add [files]
git commit -m "type: description"
```

## Common Mistakes

### Vague Tasks

**Bad:** "Add authentication"
**Good:** "Create User model with email and password_hash fields"

### Incomplete Code

**Bad:** "Step 1: Add validation function"
**Good:** "Step 1: Add validation function" followed by the complete function code

### Missing Verification

**Bad:** "Step 3: Test it works"
**Good:** "Step 3: Run `pytest tests/test_auth.py -v`, expected: 3 passed"

### Missing File Paths

**Bad:** "Create the model file"
**Good:** "Create: `src/models/user.py`"

## Multi-Sprint Implementation Plans (Phased Roadmaps)

Use this pattern when the work spans multiple weeks/phases and requires dependency tracking, effort estimation, or phased delivery. Each phase = one sprint (1 week) with grouped tasks.

### When to Use

- User asks for a multi-week plan (e.g. generate an implementation plan on a large project)
- Work naturally splits into phases (Security to Architecture to Quality)
- Features have dependencies that enforce ordering (CandleBuilder must exist before StrategyEngine)
- The user wants effort estimates before committing to a timeline

### Plan Document Structure

The plan should document:

```
Sprint 1-2 -> Phase 1: Foundation
Sprint 3-5 -> Phase 2: Core
Sprint 6-7 -> Phase 3: Enhancement
```

### Phase 1: Name (Sprints 1-N)

**Goal:** One sentence describing the phase outcome.

#### Sprint 1: Name

| # | Task | Files | Effort | Dependencies |
|---|---|---|---|---|
| 1.1 | Task description | path/to/file.ts | 2h | - |
| 1.2 | Task description | path/to/file.ts | 4h | 1.1 |

**Total Sprint 1:** ~6h

**Phase 1 exit criteria:**
- [ ] Criterion 1 met
- [ ] Criterion 2 met

### Effort Summary Table

| Phase | Sprints | Hours | Dependencies |
|---|---|---|---|
| F1: Foundation | 2 | ~16h | Some external dependency |
| F2: Core | 3 | ~47h | Some external API |
| **Total** | **5** | **~63h** | |

### Sprint Structure Rules

Each sprint entry MUST include:
1. Task table with columns: #, Task, Files, Effort, Dependencies
2. Total effort at the bottom
3. Phase exit criteria as a checklist
4. Dependencies column with references to earlier tasks

### Effort Estimation Heuristics

- **Quick win** (typo, import cleanup, config tweak, dead code removal): 5-15 min
- **Simple change** (add enum, add validation, rename/refactor one file): 30 min - 1h
- **New service** (create a service with 2-5 methods, tests): 2-4h
- **New module** (routes + controller + service + tests): 4-8h
- **Complex feature** (full integration, engine, backtesting): 8-16h

### Dependency Tracking

Sequential dependencies (4.4 depends on 4.1) execute in order within same subagent.
Cross-phase dependencies (Phase 2 depends on Phase 1) enforce phase ordering.
Independent parallel tasks (1.1 and 1.2) can dispatch via delegate_task with tasks array.

### Subagent Execution Notes

1. One subagent per sprint - not per task. Sprint tasks are interrelated.
2. Include the FULL sprint table in the subagent context.
3. Tell the subagent to read files first - cache may be stale.
4. Require a commit per sprint - git add -A && git commit -m "Sprint N: summary"
5. Verify the build after each sprint - npx tsc --noEmit or equivalent.

### Pitfalls

- Planning too far ahead - later phases change based on early learnings. Keep them high-level.
- Vague dependencies - be specific (2.3 JWT generation, not dep on auth)
- Missing exit criteria - without these you cannot tell if a phase is done.
- Over-optimistic estimates - double estimates for external API integrations.
- Cross-phase parallel execution - do not dispatch Phase 2 until Phase 1 criteria are met.

---

## Business Logic Documentation Pattern

After analysis or before planning, write a Business Logic Document (LOGICA-NEGOCIO.md) that captures current state, data model, flows, weaknesses, and proposed improvements.

### Document Sections

| Section | Content |
|---|---|
| 1. System Overview | One-paragraph summary + ASCII architecture diagram |
| 2. Data Model | Entity-relationship text diagram + each table/entity documented field-by-field with type and purpose |
| 3. Current Flows | Each flow: endpoint, files, step-by-step, code snippets, observations with good/issue icons |
| 4. Analysis | Strengths table + Weaknesses/Risks table with impact and file references |
| 5. Improvement Opportunities | Immediate + Short-term + Medium-term with effort estimates |
| 6. Proposed Features | Feature specs with interfaces, code examples, implementation notes |
| Glossary | Definitions of domain terms |

### When to Write

- After completing a multi-agent code analysis (findings feed into weaknesses section)
- Before starting a large refactoring effort
- When a new developer or AI agent needs to understand the project

The document lives in the project root, committed to git. Update it as features are implemented.

### Flow Documentation Format

```
### 3.X Flow Name

**Endpoint:** POST /api/resource
**Files:** controllers/x.ts, services/x.ts

**Flow:**
1. Client sends request
2. Controller validates input via Zod
3. Service queries DB
4. Response returned

**Observations:**
- Good: Validation with Zod
- Issue: Direct DB access in controller instead of through service
```

---

## Execution Handoff

After saving the plan, offer the execution approach:

**"Plan complete and saved. Ready to execute using subagent-driven-development - I will dispatch a fresh subagent per task (or per sprint for multi-phase plans) with two-stage review (spec compliance then code quality). Shall I proceed?"**

When executing, use the subagent-driven-development skill:
- Fresh delegate_task per task (or per sprint for large plans)
- Spec compliance review after each task
- Code quality review after spec passes
- Proceed only when both reviews approve

## Simple Plan Mode (from plan)

Use this mode when the user wants a plan instead of execution.

### Core Behavior
- Do not implement code
- Do not edit project files except the plan markdown file
- Do not run mutating terminal commands, commit, push, or perform external actions
- You may inspect the repo or other context with read-only commands/tools when needed
- Your deliverable is a markdown plan saved inside the active workspace under `.hermes/plans/`

### Output Requirements
Write a markdown plan that is concrete and actionable.

Include, when relevant:
- Goal
- Current context / assumptions
- Proposed approach
- Step-by-step plan
- Files likely to change
- Tests / validation
- Risks, tradeoffs, and open questions

If the task is code-related, include exact file paths, likely test targets, and verification steps.

### Save Location
Save the plan with `write_file` under:
- `.hermes/plans/YYYY-MM-DD_HHMMSS-<slug>.md`

Treat that as relative to the active working directory / backend workspace. Hermes file tools are backend-aware, so using this relative path keeps the plan with the workspace on local, docker, ssh, modal, and daytona backends.

If the runtime provides a specific target path, use that exact path.

### Interaction Style
- If the request is clear enough, write the plan directly.
- If no explicit instruction accompanies `/plan`, infer the task from the current conversation context.
- If it is genuinely underspecified, ask a brief clarifying question instead of guessing.
- After saving the plan, reply briefly with what you planned and the saved path.

## Remember

```
Bite-sized tasks (2-5 min each)
Exact file paths
Complete code (copy-pasteable)
Exact commands with expected output
Verification steps
DRY, YAGNI, TDD
Frequent commits
```

**A good plan makes implementation obvious.**
