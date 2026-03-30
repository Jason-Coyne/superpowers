# Security Audit: superpowers-extended-cc

**Date:** 2026-03-30
**Repository:** pcvelz/superpowers (fork of obra/superpowers)
**Scope:** Full repository scan for malicious behavior, prompt injection, data exfiltration, and hidden/trojan code
**Methodology:** 4 parallel automated agents + manual diff review against upstream (obra/main)

---

## Executive Summary

**Verdict: No malicious behavior detected.**

The repository is a legitimate Claude Code-specific enhancement of the upstream obra/superpowers project. All code changes are transparent, auditable, and user-protective. No evidence of data exfiltration, prompt injection attacks, credential harvesting, or hidden backdoors.

---

## 1. Data Exfiltration Analysis

### Network & External Communication

| Category | Finding |
|----------|---------|
| Outbound HTTP/HTTPS calls | None added by fork |
| External URLs | Only GitHub repo links (obra/superpowers changed to pcvelz/superpowers) |
| Brainstorm WebSocket server | Binds to **localhost only** (127.0.0.1), random ephemeral port (49152-65535) |
| Telemetry/analytics | None. Searched for "analytics", "tracking", "beacon", "collect" — zero results |
| External reporting | No code sends data outside the local machine |

### Credential & Sensitive File Access

| Category | Finding |
|----------|---------|
| .env / API keys / tokens | Not accessed |
| ~/.ssh, /etc/passwd | Not accessed |
| AWS/cloud credentials | Not accessed |
| Environment variables read | Only `BRAINSTORM_PORT`, `BRAINSTORM_DIR`, `BRAINSTORM_HOST`, `BRAINSTORM_URL_HOST`, `BRAINSTORM_OWNER_PID` — all legitimate brainstorm server config |
| Hook transcript access | Hooks read `$TRANSCRIPT_PATH` from Claude Code's own JSON — session files only |

### Data Storage

All data is local:
- `/tmp/brainstorm-*` (ephemeral)
- `.superpowers/brainstorm/` (project-local)
- `.tasks.json` files alongside plan documents

**Risk Level: None**

---

## 2. Code Execution & Dynamic Code Analysis

| Pattern | Finding |
|---------|---------|
| `eval()` | Not used anywhere |
| `Function()` constructor | Not used |
| `setTimeout`/`setInterval` with eval | Not used |
| `exec`/`execSync` | Only in `render-graphs.js` to run GraphViz `dot` command (diagram rendering) |
| `spawn` | Only in test files to start server processes |
| Shell scripts | All legitimate — test runners, server lifecycle, hook validators |
| Python in hooks | `python3 -c` used in pre-commit and pre-task-complete hooks to parse JSON transcripts |

### Hook Script Detail

The fork adds three hook scripts. All are benign task-enforcement mechanisms:

**`hooks/pre-commit-check-tasks`** — Blocks `git commit` when native tasks are incomplete. Reads Claude Code transcript, counts open TaskCreate/TaskUpdate entries. Falls through to `{"decision": "allow"}` on any error (fail-safe design).

**`hooks/pre-task-complete-check-verification`** — Blocks TaskUpdate(completed) when task has `requiresUserVerification: true` but no `AskUserQuestion` was called. Same fail-safe pattern.

**`hooks/examples/pre-commit-check-tasks.sh`** — Example/documentation version of the above.

### Minor Note: Path Interpolation in Hooks

The hook scripts use `python3 -c` with `$TRANSCRIPT_PATH` interpolated into Python code:
```python
for line in open('$TRANSCRIPT_PATH'):
```
This is technically a path injection vector — if a transcript path contained Python code fragments, it could execute. However, `$TRANSCRIPT_PATH` comes from Claude Code's own JSON output via `jq -r '.transcript_path'`, not from user input. **Practical risk: negligible.**

**Risk Level: None**

---

## 3. Base64 & Obfuscation Analysis

| Category | Finding |
|---------|---------|
| Base64 encoding | Only in WebSocket handshake (`computeAcceptKey` in server.cjs) — standard RFC 6455 |
| Obfuscated strings | None found |
| Encoded payloads | None found |
| Hidden content | None found |
| Zero-width characters | None found |

**Risk Level: None**

---

## 4. Prompt Injection Analysis

### Emphatic Language Patterns

The fork uses assertive language extensively:
- `EXTREMELY-IMPORTANT`, `CRITICAL CONSTRAINTS`, `HARD-GATE`
- "YOU MUST", "This is not negotiable", "You cannot rationalize your way out of this"
- "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST"

This is characteristic of the upstream project as well — it is a deliberate technique for ensuring AI agents follow skill workflows. The fork amplifies this pattern but does not weaponize it.

### Instruction Hierarchy

The `skills/using-superpowers/SKILL.md` explicitly states priority order:
1. **User's explicit instructions** (CLAUDE.md, direct requests) — highest priority
2. **Superpowers skills** — override default system behavior
3. **Default system prompt** — lowest priority

This preserves user control — the opposite of a prompt injection attack.

### Persuasion Principles Documentation

`skills/writing-skills/persuasion-principles.md` explicitly documents psychological persuasion techniques (authority, commitment, scarcity, social proof, unity, reciprocity, liking) for writing skills. It references academic research showing these techniques doubled AI compliance rates (33% to 72%).

This is **transparently documented** with ethical guidelines:
> "Would this technique serve the user's genuine interests if they fully understood it?"

This is notable for awareness — the repo is intentionally designed to maximize AI compliance with its workflows — but it is not hidden or malicious.

### No Evidence Of

- Hidden instructions in HTML comments
- Zero-width characters or encoded directives
- Instructions to hide information from users
- Attempts to extract system prompts
- Jailbreak-style patterns
- Instructions redirecting behavior to benefit a third party
- Tool definitions that override safety measures
- Instructions embedded in non-obvious places (variable names, code comments)

**Risk Level: None**

---

## 5. Hidden/Trojan Code Analysis

### Git Hooks

| Location | Finding |
|----------|---------|
| `.git/hooks/` | Only sample files (*.sample) — no active hooks |
| `hooks/` directory | Claude Code hook configurations (JSON + shell scripts) — all legitimate |

### Package Scripts

| File | Finding |
|------|---------|
| `package.json` (root) | No scripts section — no postinstall/preinstall |
| `tests/brainstorm-server/package.json` | Only "test" script |

### System Modifications

| Category | Finding |
|----------|---------|
| Shell config (~/.bashrc, ~/.zshrc, ~/.profile) | Not modified |
| Git config | Not modified |
| PATH / environment variables | Not modified (only internal brainstorm server vars) |
| Cron jobs / scheduled tasks | None created |
| Network listeners | Brainstorm server on localhost only |

### Automatic Git Operations

The `finishing-a-development-branch` skill explicitly presents 4 options to the user before any git operation:
1. Merge locally
2. Push and create PR
3. Keep branch as-is
4. Discard work

Multiple skills contain: "Never start implementation on main/master branch without explicit user consent."

**Risk Level: None**

---

## 6. Dependencies & Supply Chain

| Category | Finding |
|----------|---------|
| External npm dependencies | Minimal — `ws` (WebSocket library) for brainstorm server tests only |
| Suspicious packages | None |
| Core functionality | Uses only Node.js built-ins: `crypto`, `http`, `fs`, `path` |
| Binary files | Only `docs/screenshots/*.png` (legitimate documentation images) |

**Risk Level: None**

---

## 7. Attribution & Licensing

| Field | Upstream (obra) | Fork (pcvelz) |
|-------|-----------------|----------------|
| Author name | Jesse Vincent | pcvelz |
| Author email | jesse@fsck.com | pcvelz@users.noreply.github.com |
| Homepage | github.com/obra/superpowers | github.com/pcvelz/superpowers |
| License | MIT | MIT (unchanged) |
| Plugin name | superpowers | superpowers-extended-cc |

The fork changes all author references but maintains the MIT license and links back to the upstream repo in the README. This is standard forking practice, though the complete removal of the original author from metadata files is worth noting.

---

## 8. Risk Summary Table

| Category | Risk Level | Notes |
|----------|-----------|-------|
| Data exfiltration | **None** | No outbound network calls, no credential access |
| Prompt injection | **None** | Assertive language is user-protective, not weaponized |
| Hidden backdoors | **None** | All code changes are transparent and auditable |
| Trojan/hidden code | **None** | No install scripts, no system modifications |
| Supply chain risk | **Low** | Rebranded marketplace entry could confuse users expecting upstream |
| Path interpolation in hooks | **Negligible** | `$TRANSCRIPT_PATH` from trusted Claude Code JSON |
| Attribution concerns | **Low** | Original author removed from metadata; MIT license maintained |
| Persuasion techniques | **Informational** | Transparently documented, ethically framed, but worth awareness |

---

## 9. Conclusion

This repository is safe to use. The fork makes no malicious modifications to the upstream codebase. All changes serve legitimate purposes: native task management, user verification enforcement, namespace rebranding, and Claude Code-specific workflow improvements. The hook scripts are fail-safe (allow on error) and user-protective (block premature commits and unverified task completion).

The only items warranting ongoing awareness are:
1. The transparent but intentional use of persuasion psychology to maximize AI compliance with skill workflows
2. The complete replacement of upstream author attribution in metadata files
