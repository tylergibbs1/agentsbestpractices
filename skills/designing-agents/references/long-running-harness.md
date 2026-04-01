# Long-Running Agent Harness

How to build harnesses for agents that work across multiple context windows on tasks that take hours or days.

## Contents
- The Core Problem
- Two-Agent Architecture
- State File Design
- Session Initialization Protocol
- Failure Modes and Fixes
- Testing Strategy

## The Core Problem

Long-running agents operate in discrete sessions with no memory of previous work. Each new context window is like a new engineer starting a shift with no handoff documentation. Without structure, agents:

- Declare victory prematurely
- Leave broken or undocumented code
- Waste time rediscovering project state
- Mark tasks complete without testing
- Redo work they've already done

The solution: make the filesystem the persistent medium and agents disposable.

## Two-Agent Architecture

### Initializer Agent (First Session Only)

Sets up everything the worker agents need:

1. **Setup script** (`init.sh`) — starts dev servers, installs deps, restores environment. Every subsequent session runs this instead of figuring out setup from scratch.

2. **Feature list** (`feature_list.json`) — structured task list with all requirements. Each feature includes description, verification steps, and a `passes` field.

3. **Progress file** (`claude-progress.txt`) — freeform notes on what's been done, what's blocked, what's next.

4. **Initial git commit** — establishes baseline so workers can see what changed.

### Worker Agent (Every Subsequent Session)

Works incrementally, one feature per session:

```
1. pwd                              # Confirm working directory
2. Read claude-progress.txt         # Understand current state
3. Read feature_list.json           # Find next task
4. git log --oneline -20            # See recent work
5. Run init.sh                      # Restore environment
6. Run basic tests                  # Verify nothing is broken
7. Implement ONE feature            # Focused work
8. Test the feature                 # Self-verify
9. Update feature_list.json         # Mark passes: true
10. git commit                      # Checkpoint
11. Update claude-progress.txt      # Document what was done
```

## State File Design

### Feature List (JSON, not Markdown)

JSON is critical — markdown is prone to unintended overwrites and harder to parse programmatically.

```json
{
  "category": "functional",
  "description": "New chat button creates a fresh conversation",
  "steps": [
    "Navigate to main interface",
    "Click the 'New Chat' button",
    "Verify a new conversation is created",
    "Check that chat area shows welcome state",
    "Verify conversation appears in sidebar"
  ],
  "passes": false
}
```

**Rules for the feature list:**
- Agents may ONLY modify the `passes` field
- Removal or editing of tests is unacceptable — this prevents agents from "passing" by deleting tests
- Include 5+ verification steps per feature so agents can't shortcut testing
- Pre-populate with all features marked `passes: false`

### Progress File (Freeform Text)

```
Session 4 progress:
- Implemented chat message rendering with markdown support
- Fixed sidebar conversation list ordering (newest first)
- Next: message streaming with typing indicators
- Blocked: WebSocket connection drops after 30 seconds (need to investigate)
- Note: init.sh updated to also start WebSocket server
```

Freeform text works better for progress notes than structured formats — agents can express nuance, blockers, and context that structured fields can't capture.

### Git as Memory

Every commit is a checkpoint. Workers read `git log` to understand what happened in previous sessions. Descriptive commit messages act as a compressed history:

```
feat: implement chat message rendering with markdown
fix: sidebar conversation ordering (newest first)
chore: update init.sh to start WebSocket server
```

## Session Initialization Protocol

Every worker session MUST start by reading state before doing anything:

```
[Agent] I'll start by getting my bearings.
> pwd
> cat claude-progress.txt
> cat feature_list.json
> git log --oneline -20
> bash init.sh
> [run basic tests to verify existing features still work]
[Agent] Existing features verified. Starting on next task.
```

**Why verify before implementing:** If a previous session left something broken, the worker needs to fix it before adding new work. Otherwise breakage compounds across sessions.

**Why read state before asking the user:** The whole point is autonomous operation. The filesystem has everything the agent needs.

## Failure Modes and Fixes

| Failure Mode | Root Cause | Fix |
|---|---|---|
| **Declares victory prematurely** | No structured task list | Create feature list with 100+ features all marked `passes: false` |
| **Leaves broken code** | No verification step | Require running tests before AND after implementation |
| **Marks complete without testing** | No self-verification requirement | Include specific test steps in each feature; require browser testing |
| **Wastes time on setup** | No setup script | Initializer creates `init.sh`; workers run it first |
| **Redoes previous work** | Doesn't read history | Session protocol starts with git log + progress file |
| **Edits tests to make them pass** | Tests are modifiable | Lock down feature list — only `passes` field is editable |
| **Drifts from objectives** | No plan reinforcement | Worker reads feature list each session; picks highest-priority incomplete item |
| **Breaks existing features** | No regression testing | Run full test suite before starting new work |

## Testing Strategy

### Self-Verification

Agents must test their own work before marking features complete. This requires:

- **Browser automation** (e.g., Puppeteer MCP) for UI testing
- **End-to-end tests** that exercise features as a user would
- **Known limitations to document:** browser-native alert modals, complex animations, OAuth flows — things agents can't easily test

### Test Before and After

```
Session flow:
1. Run tests → verify baseline (catch regressions from previous sessions)
2. Implement feature
3. Run tests → verify new feature works AND old features still pass
4. Only then mark feature as passing
```

### Verification Steps in Feature List

Each feature should include concrete, testable steps — not vague descriptions:

```json
// Bad: vague
{"description": "Chat works", "steps": ["Test chat"], "passes": false}

// Good: specific
{
  "description": "Message persistence across page reload",
  "steps": [
    "Send 3 messages in a conversation",
    "Reload the page",
    "Verify all 3 messages appear in correct order",
    "Verify message timestamps are preserved",
    "Verify conversation is selected in sidebar"
  ],
  "passes": false
}
```

## Extending Beyond Coding

The initializer + worker pattern applies to any long-running task:

- **Research:** initializer creates research plan + sources list; workers tackle one source per session, update findings file
- **Data analysis:** initializer sets up data pipeline scripts; workers process one dataset segment per session
- **Content creation:** initializer creates outline + style guide; workers draft one section per session

The key insight is always the same: make files the persistent medium, make agents disposable, verify before extending.
