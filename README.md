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

**Disable Auto Plan Mode** — Claude Code may automatically enter Plan mode, which conflicts with the skill workflows. Add `EnterPlanMode` to your deny list in your project's `.claude/settings.json`:

```json
{
  "permissions": {
    "deny": ["EnterPlanMode"]
  }
}
```

**Block Commits With Incomplete Tasks** — The plugin includes a hook that blocks `git commit` when native tasks are still open. Add this to your `.claude/settings.local.json`:

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

> **Note:** The marketplace directory name may vary. After installation, run `ls ~/.claude/plugins/marketplaces/` to confirm the exact path and adjust the hook command if needed.

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
