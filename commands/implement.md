---
description: Implement one or all tasks from a /plan-tasks plan using TDD. Single task runs inline. All-tasks mode orchestrates sequential subagents — one per task.
argument-hint: <feature-slug> <task-number | all>
---

# Implement Command

Executes tasks from a plan created by `/plan-tasks`.

**Usage:**
```
/implement <feature-slug> <N>      implement task N with TDD
/implement <feature-slug> all      implement all pending tasks (orchestrated)
```

Arguments: `$ARGUMENTS`

Parse the first token as `<feature-slug>` and the second as `<target>` (a task number or the literal `all`).

---

## Phase 0 — LOAD PLAN

Resolve the plan directory: `.claude/plans/<feature-slug>/`

Read `_index.md`. If the directory or index does not exist:
```
Error: No plan found at .claude/plans/<feature-slug>/
Run /plan-tasks <feature-description> to create one first.
```

Parse the task list and their current statuses from `_index.md`.

---

## Phase 1 — SINGLE TASK MODE

*Skip to Phase 2 if `<target>` is `all`.*

### 1a. Read and Validate

Read `.claude/plans/<feature-slug>/task-<NN>.md`.

Check `depends_on`: for each listed task number, verify its status is `done` in `_index.md`.
If a dependency is not done, stop:
```
Error: Task NN depends on task MM which is not yet complete.
Run /implement <feature-slug> MM first.
```

If the task status is already `done`, warn the user:
```
Task NN is already marked done. Re-implement? (yes / no)
```
Stop unless the user confirms.

### 1b. Mark In-Progress

Update `task-<NN>.md` frontmatter: `status: in-progress`
Update the corresponding row in `_index.md`.

### 1c. Implement with TDD

Apply the **tdd-workflow** to implement the task using the acceptance criteria as the test target:

1. **RED** — Write failing tests that cover each acceptance criterion
2. **GREEN** — Write the minimum implementation to make all tests pass
3. **REFACTOR** — Clean up the code without breaking tests
4. **VERIFY** — Run type-check and lint; resolve any issues before continuing

### 1d. Mark Done

Update `task-<NN>.md`:
- Set `status: done` in frontmatter
- Append the following section at the end of the file:

```markdown
## Implementation Notes
<!-- Added by /implement after completion -->

**Completed:** <YYYY-MM-DD>

**Files changed:**
- `path/to/file` — created | modified

**Deviations from plan:**
<Describe any deviations and why, or write "None".>

**Interface changes:**
<List any public interfaces, types, function signatures, or file paths that
differ from what the plan described. Downstream tasks may need updating.>
```

Update the corresponding row in `_index.md` to `done`.

### 1e. Update Downstream Tasks

Scan all task files with `status: pending` or `status: in-progress` that have a task number greater than NN.

For each downstream task, read its **Context** and **Acceptance Criteria** sections and compare against the **Interface changes** recorded in step 1d.

If any interface, file path, type name, or data structure introduced or changed in task NN affects a downstream task:
- Update the **Context** section of that task file to reflect the actual implementation
- Prepend a notice at the top of the affected section:
  ```
  > Updated after task NN: <brief reason for the change>
  ```
- Do NOT alter the task's **Goal** or **Acceptance Criteria** unless a criterion has become impossible or redundant due to the upstream change. If that happens, annotate the criterion with a strikethrough and a note rather than deleting it silently.

### 1f. Report

```
Task NN complete: <title>

  Tests:      N written, all passing
  Files:      <list of changed files>
  Downstream: tasks NN+1, NN+2 updated  (or "none affected")

Next: /implement <feature-slug> <NN+1>
```

---

## Phase 2 — ALL TASKS MODE (Orchestrated)

*Activated when `<target>` is `all`.*

You are now the orchestrator. You drive the plan from start to finish by running tasks sequentially — one at a time — using a dedicated subagent for each task. You own the loop, the dependency graph, and the downstream propagation. Subagents own individual task execution. **Do not implement tasks yourself — all coding happens inside subagents.**

### 2a. Pre-flight

1. Parse `_index.md` to extract the full task list: number, title, status, `depends_on`.
2. Collect all tasks with `status: pending` in ascending order. Skip tasks that are `done`.
3. If all tasks are already done, stop:
   ```
   All tasks are already complete. Nothing to implement.
   ```
4. Validate the dependency graph: every `depends_on` reference must point to a task number that exists in the index. If not, stop:
   ```
   Error: Task NN lists depends_on: MM, but task MM does not exist in the plan.
   ```

### 2b. Sequential Execution Loop

Process each pending task in ascending order. **Never run two tasks in parallel.** Tasks may depend on the outputs of previous tasks.

#### Step 1 — Dependency Check

Re-read `_index.md` (it may have been updated by a previous iteration).

Verify that every task listed in the current task's `depends_on` has `status: done`.

If a dependency is not done:
```
Error: Task NN depends on task MM which is not yet complete.
This indicates a plan ordering problem. Halting loop.
```
Halt and wait for user instruction.

#### Step 2 — Spawn Task Subagent

Launch a **generalPurpose** subagent (using the Task tool) for the current task. Build the prompt by combining:

- The full content of `task-<NN>.md`
- The full content of `_index.md` (current, as-of this iteration)
- The full content of all downstream `task-*.md` files (tasks with numbers > NN that have `status: pending` or `status: in-progress`)
- The task instructions below, with `<NN>` replaced by the actual task number and `<plan-dir>` replaced by `.claude/plans/<feature-slug>/`

**Task instructions to pass to each subagent:**

````
You are implementing task <NN> from the plan at <plan-dir>.

Execute the following steps in order. Do not skip any step.

## 1. Mark In-Progress
Update `task-<NN>.md` frontmatter: set `status: in-progress`.
Update the corresponding row in `_index.md` to `in-progress`.

## 2. Implement with TDD
Apply the tdd-workflow skill to implement this task using its acceptance criteria as the test target:
1. RED   — Write failing tests that cover each acceptance criterion
2. GREEN — Write the minimum implementation to make all tests pass
3. REFACTOR — Clean up without breaking tests
4. VERIFY — Run type-check and lint; resolve any issues

## 3. Mark Done
Update `task-<NN>.md`:
- Set `status: done` in frontmatter
- Append this section at the end of the file:

```markdown
## Implementation Notes
<!-- Added by /implement after completion -->

**Completed:** <YYYY-MM-DD>

**Files changed:**
- `path/to/file` — created | modified

**Deviations from plan:**
<Describe any deviations and why, or write "None".>

**Interface changes:**
<List any public interfaces, types, function signatures, or file paths that
differ from what the plan described. Downstream tasks may need updating.>
```

Update the corresponding row in `_index.md` to `done`.

## 4. Update Downstream Tasks
Scan all task files with `status: pending` or `status: in-progress` that have a task number greater than <NN>.

For each downstream task, compare its Context and Acceptance Criteria sections against the Interface changes you recorded in step 3.

If any interface, file path, type name, or data structure you introduced or changed affects a downstream task:
- Update the Context section of that downstream task file to reflect the actual implementation
- Prepend a notice at the top of the affected section:
  > Updated after task <NN>: <brief reason>
- Do NOT alter the task's Goal or Acceptance Criteria unless a criterion has become impossible or redundant. If so, annotate with a strikethrough and a note — never delete silently.

When done, return a short summary:
- Task status (done / failed)
- Tests written and passing
- Files changed
- Downstream tasks updated (or "none")
- Any deviations from the plan
````

#### Step 3 — Verify Completion

After the subagent returns, re-read `task-<NN>.md` and confirm `status: done`.

**If done:** continue to Step 4.

**If not done or the subagent reported an error:**
```
Task NN failed or was not marked done.

Subagent output:
<paste subagent summary here>

Options:
  continue — re-attempt this task with a new subagent
  skip     — mark task NN as skipped and move to NN+1
  abort    — stop the entire loop here
```
Pause and wait for user instruction before proceeding.

#### Step 4 — Propagate Downstream Updates

Read the **Interface changes** section written by the subagent in `task-<NN>.md`.

Check all downstream `task-*.md` files (pending or in-progress, number > NN) for references to any type, interface, function, file path, or data shape the subagent changed or added.

For any downstream task that needs updating and was **not** already updated by the subagent:
- Update its **Context** section to reflect the actual implementation
- Prepend: `> Updated after task NN: <brief reason>`
- Do not alter **Goal** or **Acceptance Criteria** silently

This step is necessary because subagents only see tasks immediately downstream; you hold the full graph view.

#### Step 5 — Advance

Move to the next pending task and repeat from Step 1.

### 2c. Final Report

After all tasks complete (or the loop is halted by a failure), output:

```
## Implementation Complete: .claude/plans/<feature-slug>/

| #  | Title            | Status          | Files Changed |
|----|------------------|-----------------|---------------|
| 01 | <title>          | done            | N             |
| 02 | <title>          | done            | N             |
| 03 | <title>          | skipped/failed  | —             |

Total: N tasks completed, M files created or modified

Suggested next steps:
  /code-review     review all changes before committing
  /prp-commit      commit with a structured message
```

---

## Invariants

These rules apply to both single-task and all-tasks mode:

- **Sequential only.** In all-tasks mode, never launch two task subagents simultaneously. Task N+1 must not start until task N is confirmed done.
- **No silent skips.** If a task cannot run (dependency not met, subagent failure), stop and report. Do not silently skip to the next task.
- **Read before you write.** Before updating any task file or `_index.md`, re-read the current state to avoid clobbering changes made by subagents.
- **Downstream propagation is mandatory.** After every task, always check whether interface changes affect later tasks — even if the subagent already updated some of them.
- **Do not implement yourself in all-tasks mode.** Your job is coordination, verification, and propagation. All actual coding happens inside subagents.
