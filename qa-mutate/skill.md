---
name: qa-mutate
description: >-
  Perform assertion-driven mutation testing to evaluate test suite effectiveness.
  Use this skill when the user wants to assess test quality via mutation testing,
  find test blind spots, get a confidence score for their test suite, or verify
  that tests truly catch business rule violations. Triggers on phrases like
  "mutation testing", "qa mutate", "变异测试", "test quality score",
  "测试信心评分", "test effectiveness".
---

# QA Mutate

You are an experienced QA engineer performing mutation testing. Your job is to introduce deliberate, meaningful mutations into the codebase — guided by the assertion checklist — and check whether the test suite detects them. Surviving mutations reveal test blind spots.

**You MUST restore all code changes after each mutation.** The codebase must be identical to its original state when you finish.

---

## Core Principle

Traditional mutation testing applies random syntactic changes. This skill is different: every mutation is **assertion-driven** — it targets a specific business rule. A surviving mutation doesn't just mean "a test is weak" — it means "this specific business rule is not effectively verified by tests."

---

## Phase 0: Determine Scope & Load Inputs

### Scope

Check if the user specified a module or category (e.g., `qa-mutate billing`, `qa-mutate account-activation`).

- **Scoped run**: Only mutate code related to the specified module/category. **Max 10 mutations.**
- **Full run**: If no scope specified, mutate across all categories. **Max 20 mutations.**
- **If project has > 50 source files and no scope**: Warn and suggest scoped run.

### Required inputs

1. **`_qa/qa-analysis.md`** — assertion list with coverage mapping
   - Also check `_qa/assertions.yaml` if it exists
   - If neither exists, STOP and tell the user to run `qa-analyze` first

2. **Project code and tests** — scan for:
   - Source code locations (where to apply mutations)
   - Test commands (how to run tests after mutation)
   - Build tool

### Pre-flight checks

1. **Run tests in clean state** — all tests must pass before mutation testing begins
   - If tests fail in clean state, STOP and report: "Cannot perform mutation testing — baseline tests fail. Run `qa-review` first to diagnose."
2. **Verify git is available** — needed for clean rollback
   - If not a git repo, use file backup/restore instead
3. **Record baseline** — save the current test results as the reference point

---

## Phase 1: Plan Mutations

### Select target assertions

Filter and prioritize:

1. **Only** assertions with `risk_weight` >= high
2. **Only** assertions where code implementation exists
3. **Prefer** assertions that have test coverage (the point is to test the tests)
4. Sort by: risk_weight desc, then confidence desc

### Budget allocation

| Scope | Max mutations | Max tool calls target |
|-------|--------------|----------------------|
| Scoped (single module) | 10 | < 60 |
| Full run | 20 | < 120 |

**Design exactly 1 mutation per assertion.** Only add a second mutation for a critical-risk assertion if the first one is too obvious (e.g., deleting an entire method).

### Design mutations

For each selected assertion, design one **meaningful** mutation:

#### Mutation types (by business relevance)

| Type | Description | Example |
|------|-------------|---------|
| **condition_negate** | Flip a business condition | `if (status == ACTIVE)` → `if (status != ACTIVE)` |
| **return_value** | Change a return/response value | `return 403` → `return 200` |
| **logic_remove** | Delete a business logic block | Remove the device conflict check entirely |
| **boundary_shift** | Change boundary values | `gracePeriodDays <= 7` → `gracePeriodDays <= 6` |
| **state_change** | Alter state transition | `status = FROZEN` → `status = ACTIVE` |
| **event_suppress** | Remove event publishing | Delete `outboxPublisher.publish(event)` line |

A mutation is **meaningful** if a customer or downstream system would be affected. Skip:
- Renaming variables, changing logs/comments/formatting
- Mutating test code or configuration files
- Changes with no observable business impact

### Mutation plan format

Present the plan as a table before executing:

| MUT ID | Assertion | Type | File:Line | What changes |
|--------|-----------|------|-----------|-------------|
| MUT-001 | ACTIV-006 | condition_negate | AccountAppService.java:131 | `== DISABLED` → `!= DISABLED` |

---

## Phase 2: Execute Mutations

### Skip compilation-only verification

For simple mutations (condition flip, line deletion, value change), **do NOT compile separately before running tests**. The test run itself will catch compilation errors. Only compile separately if the mutation is structurally complex (e.g., changing method signature).

### Execution cycle per mutation

```
1. Apply mutation (Edit tool)
2. Run ONLY related tests (not all tests)
   - Use assertion's coverage refs to identify which test class/feature
   - Timeout: max 60 seconds per test run
3. Record result:
   - Tests FAIL → KILLED ✅
   - Tests PASS → SURVIVED ❌
   - Compile error → COMPILE_ERROR (note and continue)
   - Tests ERROR → ERRORED
4. Restore: git checkout -- <file>
5. (Do NOT verify restoration after each mutation — verify once at the end)
```

### Running related tests only

To keep execution time reasonable:
- For unit tests: run **only** the specific test class(es) that cover the mutated assertion
- For integration tests: run **only** the relevant feature/scenario
- Use the coverage ref fields to identify related tests
- If you can't determine which tests to run, run tests at the corresponding level for that module only

### Batch same-file mutations

If multiple mutations target the same file:
1. Apply mutation 1, run its related tests, record result
2. Restore file
3. Apply mutation 2, run its related tests, record result
4. Restore file
5. (Keeps file I/O minimal — no need to re-read the file each time)

---

## Phase 3: Report

### Final code integrity verification

After ALL mutations are complete:
1. Run `git diff` to verify zero changes remain
2. If any changes remain, restore immediately and report
3. **This is the only mandatory restoration check** — not per-mutation

### Calculate metrics

```
Kill Rate = killed / (killed + survived) * 100%
(Exclude compile_error and errored from the calculation)
```

### Confidence scoring

| Kill Rate | Rating | Interpretation |
|-----------|--------|---------------|
| > 90% | High | Tests effectively verify business rules |
| 70-90% | Medium | Tests catch most issues but have blind spots |
| 50-70% | Low | Significant gaps in test effectiveness |
| < 50% | Very Low | Tests provide false confidence |

### Generate report

Create `_qa/qa-mutate-report.md`:

```markdown
# QA Mutation Testing Report

## Summary

| Metric | Value |
|--------|-------|
| Scope | [full / module name] |
| Total mutations | N |
| Killed | X |
| Survived | Y |
| **Kill rate** | **Z%** |
| **Confidence** | **[Rating]** |

## Kill Rate by Category

| Category | Mutations | Killed | Survived | Kill Rate |
|----------|-----------|--------|----------|-----------|

## Kill Rate by Risk Level

| Risk | Mutations | Killed | Survived | Kill Rate |
|------|-----------|--------|----------|-----------|

## Surviving Mutations (Blind Spots)

### MUT-XXX — SURVIVED ❌

- **Assertion**: [ID] ([Rule])
- **Mutation**: [What was changed]
- **File**: [path:line]
- **Business impact**: [What could go wrong if this code were actually changed this way]
- **Recommended fix**: [Specific test to add or strengthen]

[Repeat for each survivor — this is the most valuable section]

## Killed Mutations (summary table)

| MUT ID | Assertion | Type | Killed By |
|--------|-----------|------|-----------|

## Code Integrity
- Git diff after all mutations: **CLEAN** ✅ / ❌

## Top Recommendations
1. [Most important blind spot to fix]
2. [Second]
3. [Third]
```

**Do NOT generate a separate `mutations.yaml` file.** The markdown report is the only output. If a downstream process needs structured data, it can be generated on demand.

---

## Safety Rules (non-negotiable)

1. **ALWAYS restore code after each mutation.** No exceptions.
2. **Final verification**: After ALL mutations, run `git diff` to confirm zero changes.
3. **Never mutate test code** — only production/application code.
4. **Never mutate configuration files** (application.yml, pom.xml, etc.).
5. **Timeout**: If a test run takes > 60 seconds, kill it and mark as errored.
6. **If restoration fails**: STOP immediately, report the issue, help the user restore.

### Recommended execution approach

```bash
# Use git checkout for restoration (simplest)
git checkout -- <mutated-file>
```

---

## Guidelines

### Do
- Design mutations that test business rules, not syntax
- Explain surviving mutations in business terms (not just "test didn't catch it")
- Provide specific fix recommendations for each survivor
- Keep total mutations within budget (10 scoped / 20 full)

### Don't
- Apply random/syntactic mutations without business meaning
- Leave any code changes after completion
- Mutate code not covered by any assertion
- Run ALL tests for every mutation — target related tests only
- Compile separately for simple mutations — let the test run catch compile errors

### On interpreting results
- **High kill rate + survivors in one category** = that category's tests are weak
- **Low kill rate on critical assertions** = urgent, even if overall rate is OK
- **All killed** = either tests are excellent OR mutations were too obvious — check quality

---

## Integration with QA Workflow

### Upstream
- **qa-analyze**: provides assertion list that drives mutation design
- **qa-review**: confirms baseline tests pass before mutation testing

### Downstream
- Surviving mutations feed back to **qa-cover** (generate tests to catch them)

### Output files

| File | Purpose |
|------|---------|
| `_qa/qa-mutate-report.md` | Human-readable mutation testing report |
