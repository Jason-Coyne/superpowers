# Feature Analysis: superpowers-extended-cc vs obra/superpowers

**Date:** 2026-03-30
**Upstream:** obra/superpowers v5.0.6 (Jesse Vincent)
**Fork:** pcvelz/superpowers v5.2.3 (superpowers-extended-cc)
**Delta:** ~50 files changed, +1,156 / -5,695 lines

---

## Overview

This fork is a Claude Code-specific enhancement of the upstream superpowers project. The upstream is designed as a cross-platform toolkit working across Claude Code, Codex, OpenCode, and Gemini CLI. Features unique to Claude Code fall outside upstream's scope ([reference](https://github.com/obra/superpowers/pull/344#issuecomment-3795515617)), which is why this fork exists.

The fork tracks upstream releases (merged v5.0.6) while adding Claude Code-native features.

---

## 1. Native Task Management (replaces TodoWrite)

### What Changed

All `TodoWrite`/`TodoRead` references across every skill replaced with Claude Code's native task tools: `TaskCreate`, `TaskUpdate`, `TaskGet`, `TaskList` (available since CC v2.1.16+).

### Upstream Approach

Uses `TodoWrite` — markdown-based task lists that exist only as text in the conversation. No runtime visibility, no dependency enforcement, no persistence across sessions.

### Fork Approach

#### Structured Task Descriptions

Every task follows a rigid template:

```
Goal → Files → Acceptance Criteria → Verify → Steps → json:metadata
```

#### Embedded Metadata

A `json:metadata` code fence is placed at the end of each TaskCreate description containing machine-readable fields:

| Key | Type | Required | Purpose |
|-----|------|----------|---------|
| `files` | string[] | yes | Paths to create/modify/delete |
| `verifyCommand` | string | yes | Command to verify task completion |
| `acceptanceCriteria` | string[] | yes | List of testable criteria |
| `estimatedScope` | "small" / "medium" / "large" | no | Relative effort indicator |
| `requiresUserVerification` | boolean | yes | Whether completion needs explicit user approval |
| `userVerificationPrompt` | string | conditional | The question to ask (required when requiresUserVerification is true) |

Metadata is embedded in the description rather than the `metadata` parameter because the fork discovered that `TaskCreate`'s metadata parameter is accepted but **not returned by `TaskGet`**. Embedding in the description is the only reliable persistence mechanism.

#### Dependency Tracking

Tasks can declare `blockedBy` relationships via `TaskUpdate(addBlockedBy: [...])`. Execution skills respect these — a blocked task won't be started until its dependencies complete.

#### Real-Time UI Visibility

Native tasks appear in Claude Code's task panel with `pending`/`in_progress`/`completed` states, giving the user live progress without reading agent output.

#### Cross-Session Persistence

After each status change, the orchestrator writes a `.tasks.json` file alongside the plan document (e.g., `my-plan.md.tasks.json`). A new session can resume by reading this file and recreating native tasks from it.

The executing-plans skill has a "Step 0: Load Persisted Tasks" phase:
1. Call `TaskList` to check for existing native tasks
2. Look for `<plan-path>.tasks.json`
3. If tasks file exists but native tasks are empty: recreate from JSON
4. If native tasks exist: verify they match plan, resume from first pending/in_progress
5. If neither: bootstrap from plan document

#### Task Granularity Rules

New shared reference document (`skills/shared/task-format-reference.md`) defines:
- A task is "a coherent unit of work that produces a testable, committable outcome"
- TDD cycles happen within tasks, not across them
- Each task = one commit
- Scope test: Can it be verified independently? Does it touch more than one concern? Would it get its own commit?

### Files Modified

- `skills/writing-plans/SKILL.md` — major rewrite
- `skills/executing-plans/SKILL.md` — major rewrite
- `skills/subagent-driven-development/SKILL.md` — major rewrite
- `skills/brainstorming/SKILL.md` — added Native Task Integration section
- `skills/dispatching-parallel-agents/SKILL.md` — added Native Task Integration section
- `skills/using-superpowers/SKILL.md` — TodoWrite references replaced
- `skills/writing-skills/SKILL.md` — TodoWrite references replaced
- `skills/writing-skills/persuasion-principles.md` — TodoWrite references replaced
- `skills/shared/task-format-reference.md` — **new file** (129 lines)

---

## 2. User Verification Gates

### What Changed

Added a multi-layered enforcement system ensuring that when a user's original prompt requires human sign-off, that requirement survives through plan writing, task creation, and execution.

### Upstream Approach

No mechanism for requiring human sign-off on specific tasks.

### Fork Approach

#### Layer 1: Plan-Level Detection

The writing-plans skill requires the plan author to answer:

> Does the original prompt/spec require any form of user verification, user feedback, user confirmation, user approval, human sign-off, or human-in-the-loop validation?

If YES, the plan header must include a `User Verification` field, and at least one task must have `requiresUserVerification: true`. A `<HARD-GATE>` block prevents proceeding to execution handoff without this.

#### Layer 2: Task-Level Metadata

Tasks requiring human approval include:
```json
{
  "requiresUserVerification": true,
  "userVerificationPrompt": "How many errors do you see now?"
}
```

Plus a standard verification block in the description:
```yaml
AskUserQuestion:
  question: "[specific question]"
  header: "Verification"
  options:
    - label: "[positive outcome]"
      description: "[what this means]"
    - label: "[negative outcome / needs rework]"
      description: "[what happens next]"
```

#### Layer 3: Execution-Level Enforcement

The subagent-driven-development and executing-plans skills check each task's metadata before marking it complete. If `requiresUserVerification` is true, the controller must call `AskUserQuestion`. If the user selects the negative option, the task goes back for rework.

Key constraint: only the orchestrator/controller can call `AskUserQuestion` — subagents cannot. This ensures the human-in-the-loop gate cannot be bypassed by delegation.

#### Layer 4: Hook-Level Hard Enforcement

`hooks/pre-task-complete-check-verification` is a PreToolUse hook that intercepts `TaskUpdate(status: completed)` calls. It:
1. Reads the session transcript
2. Finds the task's `TaskCreate` call by counting TaskCreate occurrences to match task ID
3. Checks for `requiresUserVerification: true` in the description
4. Verifies that an `AskUserQuestion` was called after the task entered `in_progress`
5. If not, blocks completion: "TASK COMPLETION BLOCKED: Task #N has requiresUserVerification=true but no AskUserQuestion was called"
6. Falls through to allow on any error (fail-safe)

### Files Modified/Added

- `skills/writing-plans/SKILL.md` — User Verification Enforcement section, HARD-GATE blocks, Self-Review step 4
- `skills/executing-plans/SKILL.md` — user verification gate in Step 2
- `skills/subagent-driven-development/SKILL.md` — User Verification Gate section, updated process flowchart
- `skills/shared/task-format-reference.md` — verification task example
- `hooks/pre-task-complete-check-verification` — **new file** (84 lines)
- `hooks/hooks.json` — added PreToolUse matcher for TaskUpdate

---

## 3. Pre-Commit Task Gate

### What Changed

Added a hook that blocks `git commit` when native tasks are still incomplete.

### Upstream Approach

No commit-blocking mechanism.

### Fork Approach

`hooks/pre-commit-check-tasks` is a PreToolUse hook triggered on Bash tool calls containing `git commit`. It:
1. Reads the session transcript via `$TRANSCRIPT_PATH`
2. Uses Python to parse all `TaskCreate` and `TaskUpdate` calls, building a task status map
3. Counts tasks not in `completed`/`cancelled`/`deleted` state
4. If any remain open: "COMMIT BLOCKED: N incomplete native task(s). Finish tasks before committing."
5. Falls through to `{"decision": "allow"}` on any error (fail-safe)

An example version is also provided at `hooks/examples/pre-commit-check-tasks.sh` for users to adapt.

### Files Added

- `hooks/pre-commit-check-tasks` — **new file** (50 lines)
- `hooks/examples/pre-commit-check-tasks.sh` — **new file** (49 lines)
- `hooks/hooks.json` — added PreToolUse matcher for Bash

---

## 4. EnterPlanMode/ExitPlanMode Prohibition

### What Changed

Three skills now explicitly prohibit calling Claude Code's built-in `EnterPlanMode`/`ExitPlanMode` tools.

### Upstream Approach

Does not address plan mode.

### Fork Approach

The fork discovered that Claude Code's plan mode conflicts with the superpowers workflow:
- `EnterPlanMode` restricts Write/Edit tools, preventing implementation
- `ExitPlanMode` breaks the workflow and skips the user's execution choice
- There is no clean exit path from plan mode

Each prohibition is marked as a `CRITICAL CONSTRAINT` at the top of the skill:

**writing-plans:** "You MUST NOT call `EnterPlanMode` or `ExitPlanMode` at any point during this skill. Calling `EnterPlanMode` traps the session in plan mode where Write/Edit are restricted. Calling `ExitPlanMode` breaks the workflow and skips the user's execution choice."

**executing-plans:** "This skill operates in normal mode, executing a plan that already exists on disk. Plan mode is unnecessary and dangerous here — it restricts Write/Edit tools needed for implementation."

**brainstorming:** "This skill operates in normal mode. Plan mode restricts Write/Edit tools and has no clean exit."

The flowchart in using-superpowers was also updated — the node "About to EnterPlanMode?" was renamed to "About to EnterPlanMode? DON'T".

### Files Modified

- `skills/writing-plans/SKILL.md`
- `skills/executing-plans/SKILL.md`
- `skills/brainstorming/SKILL.md`
- `skills/using-superpowers/SKILL.md`

---

## 5. Execution Handoff via AskUserQuestion

### What Changed

Replaced plain-text execution options with Claude Code's native `AskUserQuestion` tool.

### Upstream Approach

After writing a plan, presents execution options as markdown text for the user to respond to.

### Fork Approach

Uses a structured `AskUserQuestion` call:

```yaml
AskUserQuestion:
  question: "Plan complete and saved to docs/superpowers/plans/<filename>.md. How would you like to execute it?"
  header: "Execution"
  options:
    - label: "Subagent-Driven (this session)"
      description: "I dispatch fresh subagent per task, review between tasks, fast iteration"
    - label: "Parallel Session (separate)"
      description: "Open new session in worktree with executing-plans, batch execution with checkpoints"
```

This creates a clickable UI element in Claude Code rather than relying on the user to type a response.

A `<HARD-GATE>` block then requires the agent to invoke the corresponding skill immediately after the user chooses — no freelancing allowed. The previous `ExitPlanMode` path is explicitly forbidden.

### Files Modified

- `skills/writing-plans/SKILL.md`

---

## 6. Enhanced Implementer Prompt Template

### What Changed

The subagent dispatch template was restructured to pass parsed metadata fields rather than raw plan text.

### Upstream Approach

Single placeholder: `[FULL TEXT of task from plan - paste it here, don't make subagent read file]`

### Fork Approach

The orchestrator reads the task via `TaskGet`, parses the embedded `json:metadata`, and fills structured fields:

```
Goal: [from task description or metadata]
Files: [from task metadata.files or description Files section]
Acceptance Criteria: [from task metadata.acceptanceCriteria or description]
Verify: [from task metadata.verifyCommand or description]
Steps: [from task description Steps section]
```

The implementer's completion report is also enhanced:
```
Status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
Files changed: [list actual files]
Acceptance criteria status:
  - [criterion 1]: PASS/FAIL
  - [criterion 2]: PASS/FAIL
Verify command output: [paste actual output]
```

### Files Modified

- `skills/subagent-driven-development/implementer-prompt.md`
- `skills/subagent-driven-development/SKILL.md` — "Dispatching with Metadata" section added

---

## 7. Slash Command Reactivation

### What Changed

Three deprecated slash commands were reactivated.

### Upstream Approach

`/brainstorm`, `/execute-plan`, `/write-plan` display deprecation messages telling users to invoke skills directly.

### Fork Approach

All three reactivated as pass-throughs:

```markdown
# commands/brainstorm.md
---
description: "You MUST use this before any creative work..."
---
Invoke the superpowers-extended-cc:brainstorming skill and follow it exactly as presented to you
```

The descriptions were also updated from "Deprecated" to active descriptions matching the skill purpose.

### Files Modified

- `commands/brainstorm.md`
- `commands/execute-plan.md`
- `commands/write-plan.md`

---

## 8. Namespace Rebrand

### What Changed

All `superpowers:` skill references changed to `superpowers-extended-cc:` across the entire codebase.

### Scope

- Every skill SKILL.md file
- All command files
- Hook scripts
- Prompt templates (implementer, spec-reviewer, code-quality-reviewer)
- Documentation references
- Plugin metadata (plugin.json, marketplace.json)
- Author/homepage/repository fields

### Enforcement

`tests/claude-code/test-fork-validation.sh` enforces this via grep, checking that no bare `superpowers:` references remain (excluding legitimate mentions of the upstream repo URL).

### Files Modified

~30 files across skills/, commands/, hooks/, .claude-plugin/, .cursor-plugin/

---

## 9. Fork Validation Test Suite

### What Changed

Added a deterministic (no LLM) test script that validates fork-specific invariants.

### Upstream Approach

No fork-specific validation tests.

### Fork Approach

`tests/claude-code/test-fork-validation.sh` runs 6 checks:

| Test | What It Validates |
|------|-------------------|
| 1. Plugin name consistency | `superpowers-extended-cc` in plugin.json and marketplace.json |
| 2. No legacy task references | No `TodoWrite`/`TodoRead` in skills (excluding references/) |
| 3. Skill prefix consistency | All skill refs use `superpowers-extended-cc:` prefix |
| 4. No RELEASE-NOTES.md | Fork uses GitHub releases instead |
| 5. Native task sections present | Key skills (writing-plans, brainstorming, dispatching-parallel-agents) have "Native Task" sections |
| 6. No docs/plans/ tracked | Development plan artifacts not committed |

### Files Added

- `tests/claude-code/test-fork-validation.sh` — **new file** (100 lines)

---

## 10. Removed Content

### Deleted from Upstream

| File/Directory | Reason |
|----------------|--------|
| `RELEASE-NOTES.md` (1,083 lines) | Fork uses GitHub releases |
| `docs/plans/2025-11-22-opencode-support-design.md` | Development artifact |
| `docs/plans/2025-11-22-opencode-support-implementation.md` | Development artifact |
| `docs/plans/2025-11-28-skills-improvements-from-user-feedback.md` | Development artifact |
| `docs/plans/2026-01-17-visual-brainstorming.md` | Development artifact |
| `docs/superpowers/plans/2026-01-22-document-review-system.md` | Development artifact |
| `docs/superpowers/plans/2026-02-19-visual-brainstorming-refactor.md` | Development artifact |
| `docs/superpowers/plans/2026-03-11-zero-dep-brainstorm-server.md` | Development artifact |
| `docs/superpowers/specs/2026-01-22-document-review-system-design.md` | Development artifact |
| `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md` | Development artifact |
| `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md` | Development artifact |

### Added Content

| File | Purpose |
|------|---------|
| `skills/shared/task-format-reference.md` | Native task metadata schema and granularity guide |
| `hooks/pre-commit-check-tasks` | Pre-commit task gate hook |
| `hooks/pre-task-complete-check-verification` | User verification enforcement hook |
| `hooks/examples/pre-commit-check-tasks.sh` | Example hook for user adaptation |
| `tests/claude-code/test-fork-validation.sh` | Fork invariant validation |
| `hooks/hooks.json` (expanded) | Added PreToolUse hook configurations |
| `docs/screenshots/extended-cc-session.png` | Visual comparison screenshot |
| `docs/screenshots/vanilla-session.png` | Visual comparison screenshot |

---

## Summary of Behavioral Impact

| Behavior | Upstream | Fork |
|----------|----------|------|
| Task tracking | TodoWrite (markdown text) | Native TaskCreate/TaskUpdate (UI panel, dependencies, persistence) |
| Task dependencies | None enforced | `blockedBy` relationships prevent out-of-order execution |
| Session resume | Not supported | `.tasks.json` persistence files enable cross-session resume |
| User verification | Not supported | 4-layer enforcement (plan, task, execution, hook) |
| Commit gating | Not supported | Hook blocks commits with incomplete tasks |
| Plan mode | Not addressed | Explicitly prohibited (conflicts with Write/Edit) |
| Execution choice | Plain text prompt | AskUserQuestion with clickable options |
| Implementer dispatch | Raw plan text paste | Structured metadata fields |
| Slash commands | Deprecated (show message) | Active (invoke fork skills) |
| Skill namespace | `superpowers:` | `superpowers-extended-cc:` |
| Task granularity | 2-5 minute steps | Coherent units producing testable, committable outcomes |
