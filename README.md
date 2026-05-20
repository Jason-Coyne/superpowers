# Superpowers — Tyler Technologies Reviewed Fork

This is Tyler Technologies' reviewed fork of [pcvelz/superpowers](https://github.com/pcvelz/superpowers) (itself a fork of [obra/superpowers](https://github.com/obra/superpowers)). It exists to reduce risk when using the Superpowers plugin across Tyler projects and environments.

---

## What Is Superpowers?

[Superpowers](https://github.com/obra/superpowers) is an open-source skills library for AI coding agents, created by Jesse Vincent. It provides a structured development workflow that activates automatically when an agent starts working:

1. **Brainstorming** — Before writing code, the agent asks what you're building, refines the design through questions, and presents it in sections for approval.
2. **Planning** — Breaks approved designs into bite-sized implementation tasks with exact file paths, code snippets, and verification steps.
3. **Execution** — Dispatches subagents per task with two-stage code review (spec compliance, then code quality), or executes in batches with human checkpoints.
4. **Testing** — Enforces test-driven development (RED-GREEN-REFACTOR) during implementation.
5. **Completion** — Verifies all tests pass, presents merge/PR options, and cleans up.

The skills trigger automatically — the agent checks for applicable skills before every action. This enforces disciplined workflows rather than ad-hoc coding.

Superpowers is cross-platform, designed to work across Claude Code, Codex, OpenCode, and Gemini CLI.

## What Is the Claude Code Fork (pcvelz)?

The [pcvelz/superpowers](https://github.com/pcvelz/superpowers) fork adds features specific to Claude Code that fall outside the upstream project's cross-platform scope. Key enhancements:

- **Native Task Management** — Replaces markdown-based TodoWrite with Claude Code's native `TaskCreate`/`TaskUpdate` tools, providing real-time progress visibility in the UI, dependency tracking (`blockedBy`), and cross-session persistence via `.tasks.json` files.
- **User Verification Gates** — A 4-layer enforcement system (plan, task metadata, execution logic, hook) ensuring that when a prompt requires human sign-off, that requirement survives through planning and execution. Tasks with `requiresUserVerification: true` cannot be marked complete without calling `AskUserQuestion`.
- **User-Thrown Gate Enforcement** — `userGate` / `user-gate` tag + opt-in hooks force re-validation when Claude closes a user-ordered verification task. See the User-Thrown Gate Enforcement section below.
- **Pre-Commit Task Gate** — A hook that blocks `git commit` when native tasks are still incomplete.
- **Structured Task Metadata** — Every task carries machine-readable fields (files, verifyCommand, acceptanceCriteria) embedded as `json:metadata` in the task description.
- **EnterPlanMode Prohibition** — Prevents Claude Code's built-in plan mode from interfering with skill workflows.

For full details, see [docs/feature-analysis-vs-upstream.md](docs/feature-analysis-vs-upstream.md).

## What Is This Fork (Tyler)?

This fork is a **reviewed, pinned snapshot** of pcvelz/superpowers intended for use in Tyler Technologies projects and environments. Its purpose is to ensure no malicious or unwanted code is introduced into our development workflows through the Superpowers plugin.

### Why This Fork Exists

Superpowers is a third-party plugin that injects instructions into every AI coding session. It includes:
- Session-start hooks that inject skill content into agent context
- PreToolUse hooks that intercept Bash and TaskUpdate calls
- Shell scripts that parse session transcripts
- Persuasion techniques documented for maximizing AI compliance with workflows

While our [security audit](docs/security-audit.md) found no malicious behavior, the nature of this plugin — running hooks on every tool call, injecting instructions at session start, and shaping agent behavior through assertive prompting — means we should not blindly track an upstream we do not control.

### What This Fork Provides

- **Security audit on record** — A full audit covering data exfiltration, prompt injection, hidden code, and supply chain risks. See [docs/security-audit.md](docs/security-audit.md).
- **Feature analysis on record** — A detailed comparison of every behavioral difference from upstream. See [docs/feature-analysis-vs-upstream.md](docs/feature-analysis-vs-upstream.md).
- **Controlled updates** — Upstream changes from pcvelz (and transitively obra) are merged only after review. No automatic tracking.
- **No surprise code** — The commit history in this fork is limited to the reviewed baseline plus any Tyler-specific adjustments.

### Update Policy

Updates from pcvelz/superpowers are pulled manually and reviewed before merging. The review should cover:

1. New or modified hook scripts (anything in `hooks/`)
2. Changes to session-start injection (`hooks/session-start`)
3. New shell/Python code execution patterns
4. Changes to plugin metadata (`.claude-plugin/`, `.cursor-plugin/`)
5. New external URLs or network calls
6. Changes to skill behavior that could affect agent autonomy

### How to Update from Upstream

```bash
# Fetch latest from both upstreams
git fetch obra
git fetch pcvelz

# Review what changed
git log --oneline pcvelz/main..HEAD   # local-only commits
git log --oneline HEAD..pcvelz/main   # upstream-only commits
git diff HEAD..pcvelz/main --stat     # file-level summary

# After review, merge
git merge pcvelz/main
```

## Installation

### Step 1: Register the marketplace

```bash
/plugin marketplace add Jason-Coyne/superpowers
```

### Step 2: Install the plugin

```bash
/plugin install superpowers-extended-cc@tyler-superpowers
```

### Stay Updated (recommended)

Third-party marketplaces don't auto-update by default — installs stay frozen on the original version until you refresh. To get future fixes and new optional hooks automatically:

1. Run `/plugin`
2. Open the **Marketplaces** tab
3. Toggle **Enable auto-update** on `superpowers-extended-cc-marketplace`

Or refresh manually any time:

```
/plugin marketplace update superpowers-extended-cc-marketplace
```

### Verify Installation

```bash
/help
```

You should see skills prefixed with `superpowers-extended-cc:` (brainstorming, writing-plans, executing-plans, etc.).

### Updating

```bash
/plugin update superpowers-extended-cc@tyler-superpowers
```

Note: Updates only pull from this reviewed fork, not directly from upstream.

### Recommended Configuration

2. **using-git-worktrees** - Activates after design approval. Creates isolated workspace on new branch, runs project setup, verifies clean test baseline.

3. **writing-plans** - Activates with approved design. Breaks work into bite-sized tasks (2-5 minutes each). Every task has exact file paths, complete code, verification steps. *Creates native tasks with dependencies.*

4. **subagent-driven-development** or **executing-plans** - Activates with plan. Dispatches fresh subagent per task with two-stage review (spec compliance, then code quality), or executes in batches with human checkpoints.

5. **test-driven-development** - Activates during implementation. Enforces RED-GREEN-REFACTOR: write failing test, watch it fail, write minimal code, watch it pass, commit. Deletes code written before tests.

6. **requesting-code-review** - Activates between tasks. Reviews against plan, reports issues by severity. Critical issues block progress.

7. **finishing-a-development-branch** - Activates when tasks complete. Verifies tests, presents options (merge/PR/keep/discard), cleans up worktree.

**The agent checks for relevant skills before any task.** Mandatory workflows, not suggestions.

## How Native Tasks Work

When `writing-plans` creates tasks, each task carries structured metadata that survives across sessions and subagent dispatch:

```yaml
TaskCreate:
  subject: "Task 1: Add price validation to optimizer"
  description: |
    **Goal:** Validate input prices before optimization runs.

    **Files:**
    - Modify: `src/optimizer.py:45-60`
    - Create: `tests/test_price_validation.py`

    **Acceptance Criteria:**
    - [ ] Negative prices raise ValueError
    - [ ] Empty price list raises ValueError
    - [ ] Valid prices pass through unchanged

    **Verify:** `pytest tests/test_price_validation.py -v`

    ```json:metadata
    {"files": ["src/optimizer.py", "tests/test_price_validation.py"],
     "verifyCommand": "pytest tests/test_price_validation.py -v",
     "acceptanceCriteria": ["Negative prices raise ValueError",
       "Empty price list raises ValueError",
       "Valid prices pass through unchanged"]}
    ```
```

The `json:metadata` block is embedded in the description because `TaskGet` returns the description but not the `metadata` parameter. This ensures metadata is always available — for `executing-plans` verification, `subagent-driven-development` dispatch, and `.tasks.json` cross-session resume.

## User-Thrown Gate Enforcement — Optional Flow

*Canonical design doc: [`docs/user-gate-flow.md`](docs/user-gate-flow.md). The section below is a reader-facing summary.*

This flow addresses a recurring failure: the user says "add a gate" or "verify it works" without specifying **how**, the agent invents a verification method, then finds it expensive at execution time and walks around it — closing the gate with an inline shortcut. The fix is a three-layer architecture that *never bothers the user during planning* and only surfaces a forced question when the agent genuinely can't proceed without one.

**The whole flow is opt-in.** It activates only when you register the hooks below in `.claude/settings.local.json`. The slash command sits dormant without the hook — installing it alone does nothing.

### Design principle — don't bombard the user during planning

Users who want questions will say "brainstorm". Users who ask for a gate during planning just want the work done, they don't want a four-question interrogation. So `writing-plans` is silent here: it applies the **stricter definition** of a gate and tags liberally. Better to over-tag and let the execution-time hook filter than to over-question the user mid-plan.

### The three layers

| Layer | When | What it does |
|-------|------|--------------|
| **Write-plan (silent tagging)** | Plan authoring | Detects gate-language in the brief ("verify", "prove", "gate", "first on one then all", "make sure", "don't proceed until"). Tags the resulting task with `userGate: true` + `tags: ["user-gate"]`. No user questions. Uses the stricter definition: strict user gates AND strict agent gates AND gray-in-between all get tagged. |
| **Execute-plan (hard trigger via hook)** | Task close / stop | The PostToolUse + Stop hooks fire when a tagged task is closed. The agent must then assess each criterion and choose one of two paths (below). |
| **`/specify-gate` slash command (dormant unless hook active)** | Execute-plan, only when the agent cannot proceed | Asked 3-4 structured questions to the user that lock down the HOW: observable outcome, proof mechanism, scope, failure policy. Produces a structured verify spec the agent consumes. |

### Agent decision at execute time

When a tagged task comes up, the agent asks itself: **do I know *how* to verify this?**

- **"Verify the `/health` endpoint returns 200"** → the HOW is self-evident. Agent just hits the endpoint, captures the output, posts `AC: <criterion> — PROVEN BY <evidence>`. No slash command needed. The hook sees the proof and passes.
- **"Check it works"** → the HOW is vague. Agent invokes `/specify-gate`, which asks the user the 3-4 minimal questions, then uses the answers to execute real verification. No silent invention, no inline shortcut.
- **Write-plan explicitly flagged `requiresUserSpecification: true`** → same path: invoke `/specify-gate`, ask the user.

The user is only interrupted at execute time, and only when the alternative is the agent making something up.

### Activation

Register both hooks in `.claude/settings.local.json` (see "Recommended Configuration" below for the exact JSON). Without them:
- `writing-plans` still tags gates (harmless extra metadata).
- `/specify-gate` still exists but is never triggered automatically.
- Nothing enforces evidence at close — behavior is identical to vanilla.

Install one hook or both. The PostToolUse hook catches per-task closures; the Stop hook catches end-of-plan "everything done" claims. They compose — both firing on the same session is fine.

### Escape hatches

Both hooks fail-open on errors and have env-var kill switches (`SUPERPOWERS_USERGATE_GUARD=0`, `SUPERPOWERS_USERGATE_STOP_GUARD=0`) for one-off session bypasses without editing settings.

### Verify it's working

Tail the hook trace log while a tagged gate task is closing: `tail -F /tmp/claude-hooks/user-gate-trace.log`. See [Hook Trace Logging](#hook-trace-logging) for the full schema.

---

## What's Inside

### Skills Library

**Testing**
- **test-driven-development** - RED-GREEN-REFACTOR cycle (includes testing anti-patterns reference)

**Debugging**
- **systematic-debugging** - 4-phase root cause process (includes root-cause-tracing, defense-in-depth, condition-based-waiting techniques)
- **verification-before-completion** - Ensure it's actually fixed

**Collaboration**
- **brainstorming** - Socratic design refinement + *native task creation*
- **writing-plans** - Detailed implementation plans + *native task dependencies*
- **executing-plans** - Batch execution with checkpoints
- **dispatching-parallel-agents** - Concurrent subagent workflows
- **requesting-code-review** - Pre-review checklist
- **receiving-code-review** - Responding to feedback
- **using-git-worktrees** - Parallel development branches
- **finishing-a-development-branch** - Merge/PR decision workflow
- **subagent-driven-development** - Fast iteration with two-stage review (spec compliance, then code quality)

**User-Thrown Gates** (optional flow, see "User-Thrown Gate Enforcement" above)
- **checking-gates** - Do-I-know-HOW self-check for user-gate tasks; runs verify + posts `AC:…PROVEN BY` evidence, or hands off to specifying-gates
- **specifying-gates** - Interactive 4-question AskUserQuestion flow that locks down the HOW for a vague user-gate

**Meta**
- **writing-skills** - Create new skills following best practices (includes testing methodology)
- **using-superpowers** - Introduction to the skills system

## Philosophy

- **Test-Driven Development** - Write tests first, always
- **Systematic over ad-hoc** - Process over guessing
- **Complexity reduction** - Simplicity as primary goal
- **Evidence over claims** - Verify before declaring success

Read more: [Superpowers for Claude Code](https://blog.fsck.com/2025/10/09/superpowers/)

## Contributing

Contributions for Claude Code-specific enhancements are welcome!

1. Fork this repository
2. Create a branch for your enhancement
3. Follow the `writing-skills` skill for creating and testing new skills
4. Submit a PR

See `skills/writing-skills/SKILL.md` for the complete guide.

## Recommended Configuration

### Disable Auto Plan Mode

Claude Code may automatically enter Plan mode during planning tasks, which conflicts with the structured skill workflows in this plugin. To prevent this, add `EnterPlanMode` to your permission deny list.

**In your project's `.claude/settings.json`:**

```json
{
  "permissions": {
    "deny": ["EnterPlanMode"]
  }
}
```

This blocks the model from calling `EnterPlanMode`, ensuring the brainstorming and writing-plans skills operate correctly in normal mode. See [upstream discussion](https://github.com/anthropics/claude-code/issues/23384) for context.

### Block Commits With Incomplete Tasks

Optional `PreToolUse` hook that blocks `git commit` while a native task is `in_progress`. Pending tasks pass through, so per-task commit flows work as intended.

Opt in via `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/plugins/marketplaces/tyler-superpowers/hooks/examples/pre-commit-check-tasks.sh"
          }
        ]
      }
    ]
  }
}
```

See the header of `hooks/examples/pre-commit-check-tasks.sh` for how it parses the session transcript and which task states count as open.

### Force Re-Validation on User-Thrown Gate Close

Optional `PostToolUse` hook that blocks when Claude closes a **user-thrown gate** task without capturing concrete evidence for every acceptance criterion. A user-thrown gate is any task that carries `"userGate": true` or a `"user-gate"` entry in `tags` inside its `json:metadata` fence — set by `writing-plans` when the user explicitly asked for a verification step ("make sure to verify X", "add a gate", "prove it on one, then all").

Non-gate tasks pass through silently. The hook only fires when `TaskUpdate` sets status to `completed`.

Opt in via `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "TaskUpdate",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/plugins/marketplaces/superpowers-extended-cc-marketplace/hooks/examples/post-task-complete-revalidate.sh"
          }
        ]
      }
    ]
  }
}
```

See the header of `hooks/examples/post-task-complete-revalidate.sh` for how it parses `json:metadata` and the `USER-ORDERED GATE` banner, and how the `SUPERPOWERS_USERGATE_GUARD=0` escape hatch works.

### Re-Validate Gates on "Plan Complete" Claims

Optional `Stop` hook that complements the PostToolUse hook above. It fires when Claude signals plan completion ("plan complete", "both gates passed", "implementation complete", etc.) but the transcript shows user-thrown gate tasks were closed without subsequent per-criterion proof. Requires Claude to post evidence in the form `AC: <criterion> — PROVEN BY <evidence>` before it can stop.

Opt in via `.claude/settings.local.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/plugins/marketplaces/superpowers-extended-cc-marketplace/hooks/examples/stop-revalidate-user-gates.sh"
          }
        ]
      }
    ]
  }
}
```

See the header of `hooks/examples/stop-revalidate-user-gates.sh` for the full list of completion keywords and the `SUPERPOWERS_USERGATE_STOP_GUARD=0` escape hatch.

### Enforce blockedBy Ordering on in_progress

Optional `PreToolUse` hook on `TaskUpdate` that refuses to move a task into `status=in_progress` while its `blockedBy` list still points at tasks that are not yet `completed`. Motivation: observed failure mode — a coordinator jumps to a later task ("this one is simpler, zero setup") even though its declared prerequisites feed it. The plan meant V0.x to catalog state before V1.x replays consume it; without the catalog, the replay runs blind.

The hook does not silently refuse. Its stderr invites self-assessment first ("is this a hallucination — did you already do this work informally?"), offers three escalation paths (do the blocker, cancel it if truly obsolete, or raise the ordering to the user with AskUserQuestion), and explicitly warns against the bypass move of closing the blocker with status=completed without doing the work.

Opt in via `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "TaskUpdate",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/plugins/marketplaces/superpowers-extended-cc-marketplace/hooks/examples/pre-task-blockedby-enforce.sh"
          }
        ]
      }
    ]
  }
}
```

See the header of `hooks/examples/pre-task-blockedby-enforce.sh` for the transcript-walking logic and the `SUPERPOWERS_BLOCKEDBY_GUARD=0` escape hatch.

### Enforce per-task LLM/dispatch requirements

Optional `PreToolUse` hook on `Agent` that reads the currently in_progress task's `json:metadata` fence and refuses Agent calls that disagree with its `subagentType`, `model`, or `dispatchBrief`. Use when a plan's tasks are sensitive to which tier runs them — empirical measurements, coordinator-quality work, zero-cost batches.

If a task's metadata carries `{"model": "haiku"}` and the coordinator dispatches `model: "opus"`, this hook blocks the call with a stderr explaining the mismatch and three response options (retry with the required params, update metadata transparently, or escalate via AskUserQuestion).

When the task has no dispatch requirement in metadata, the hook passes silently.

Opt in via `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Agent",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/plugins/marketplaces/superpowers-extended-cc-marketplace/hooks/examples/pre-agent-task-dispatch-validate.sh"
          }
        ]
      }
    ]
  }
}
```

See the header of `hooks/examples/pre-agent-task-dispatch-validate.sh` for the transcript-walking logic and the `SUPERPOWERS_DISPATCH_GUARD=0` escape hatch. Metadata keys are documented in `skills/shared/task-format-reference.md`.

### Force Subagent Evidence on Return

Optional `PostToolUse` hook on `Agent` that fires the moment a subagent's `tool_result` arrives — before the coordinator absorbs it and reports upward. If the in_progress task carries `requireEvidenceTokens` (multi-axis evidence requirement) or the `requireABCompare: true` shortcut, the hook checks that the subagent's report contains at least one token from each axis. Missing axes → block with stderr naming them, forcing immediate re-dispatch rather than "looks good" at close time.

When the task has no evidence requirement in metadata, the hook passes silently.

Opt in via `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Agent",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/plugins/marketplaces/superpowers-extended-cc-marketplace/hooks/examples/post-agent-return-validate.sh"
          }
        ]
      }
    ]
  }
}
```

See the header of `hooks/examples/post-agent-return-validate.sh` for the metadata schema and the `SUPERPOWERS_AGENT_RETURN_GUARD=0` escape hatch.

### Hook trace log

All three user-gate hooks (post-complete revalidate, stop revalidate, pre-blockedby enforce) write one-line decision traces to `/tmp/claude-hooks/user-gate-trace.log` (override via `SUPERPOWERS_USERGATE_TRACE_LOG`). Tail during development with:

```
tail -F /tmp/claude-hooks/user-gate-trace.log
```

Each line is pipe-separated: `TIMESTAMP | hook-name | task=N | event | reason`. Events include `enter`, `skip`, `parsed`, `scanned`, `pass`, `block`, `error`. Skip reasons identify the short-circuit (e.g. `tool=Bash`, `status=pending`, `superpowers-active`, `guard=0`). This is the fastest way to see why a hook did or did not fire on a specific task.

### Block Low-Context Stop Excuses

Optional `Stop`-event hook that blocks "fresh session later" / "context is full" deflections when real context usage is below 50%.

Opt in via `.claude/settings.local.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/plugins/marketplaces/tyler-superpowers/hooks/examples/stop-deflection-guard.sh"
          }
        ]
      }
    ]
  }
}
```

See the header of `hooks/examples/stop-deflection-guard.sh` for the full list of blocked phrases, configuration environment variables, and fail-open behavior.

## Documentation

| Document | Description |
|----------|-------------|
| [docs/security-audit.md](docs/security-audit.md) | Full security audit of the pcvelz fork |
| [docs/feature-analysis-vs-upstream.md](docs/feature-analysis-vs-upstream.md) | Detailed comparison of all behavioral differences from obra upstream |
| [skills/shared/task-format-reference.md](skills/shared/task-format-reference.md) | Native task metadata schema |

## Remotes

| Remote | Repository | Purpose |
|--------|-----------|---------|
| `origin` | Jason-Coyne/superpowers | This fork (Tyler reviewed) |
| `pcvelz` | pcvelz/superpowers | Claude Code fork (upstream) |
| `obra` | obra/superpowers | Original project (root upstream) |

## License

MIT License — see LICENSE file for details.
