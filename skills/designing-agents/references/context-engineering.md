# Context Engineering Guide

Strategies for managing the finite resource of agent context.

## Contents
- Why Context Engineering Matters
- Multi-Layer Action Space
- Progressive Disclosure
- Context Offloading
- Prompt Caching
- Context Isolation
- The Ralph Loop
- Evolving Context

## Why Context Engineering Matters

Models get worse as context grows. Context has diminishing marginal returns — like humans with limited working memory. The goal is to fill the context window with **just the right information for the next step**, not everything the agent might ever need.

## Multi-Layer Action Space

Keep tool-calling layer small. Push actions to the computer:

**Layer 1 — Tool calls** (10-20 atomic tools):
- File read/write, bash, search, browser
- Each tool has a definition loaded into context

**Layer 2 — Shell utilities and CLIs:**
- Listed by name in agent instructions (not full definitions)
- Agent calls `--help` to learn usage when needed
- No upfront context cost beyond the list

**Layer 3 — Code execution:**
- Agent writes and executes code for complex operations
- Chains many actions without returning intermediate results to context
- Biggest token savings — intermediate outputs stay in the execution environment

### Token Savings Example

Tool call chain (3 calls, each returns results to context):
```
search_logs(query="error") → 2000 tokens result
get_customer(id=123) → 500 tokens result  
update_ticket(id=456, status="resolved") → 200 tokens result
Total added to context: 2700 tokens
```

Code execution (1 call, intermediate results stay in execution):
```python
# Agent writes and executes this — only final output returns to context
logs = search_logs("error")
customer = get_customer(logs[0].customer_id)
update_ticket(customer.open_ticket_id, status="resolved")
print(f"Resolved ticket for {customer.name}")
# Total added to context: ~30 tokens
```

## Progressive Disclosure

Only load information when the agent needs it. Four levels:

### Tool Definitions
- **Upfront:** Load 10-20 core tool definitions
- **On-demand:** Index remaining tools, agent searches when needed

### CLIs and Utilities
- **Upfront:** Short list of available command names
- **On-demand:** Agent runs `tool --help` to learn usage

### MCP Servers
- **Upfront:** Short description of each server's capabilities
- **On-demand:** Agent reads full tool definitions from synced folder

### Skills and Documentation
- **Upfront:** YAML frontmatter (name + description) from all skills
- **On-demand:** Agent reads full SKILL.md, then references/ as needed

### Implementation Pattern

```markdown
## Available resources

The following are available — read their docs only when needed for the current task:

- **analytics/** — BigQuery table schemas and query patterns
- **integrations/** — API docs for Slack, GitHub, Jira
- **runbooks/** — Incident response procedures
```

Agent reads `analytics/revenue.md` only when asked about revenue metrics.

## Context Offloading

Move information out of the context window into the filesystem.

### What to Offload

| Content | Strategy |
|---|---|
| Old tool results | Write to files, reference by path |
| Agent plans | Write to plan.md, re-read periodically |
| Agent trajectories | Append to log file |
| Intermediate data | Write to working files |
| Search results | Save to files, summarize only what's needed |

### Offloading vs Summarization

Offloading preserves full information — the agent can read it back if needed. Summarization permanently loses detail. **Offload first, summarize only when offloading has diminishing returns.**

```
Preferred order:
1. Offload to file (lossless, retrievable)
2. Offload + index (searchable later)
3. Summarize + offload original (compressed in context, full version available)
4. Summarize only (last resort — information loss)
```

### Steering Long-Running Agents

Write a plan file early. Re-read it periodically to:
- Reinforce objectives (prevent drift)
- Track progress (check off completed items)
- Verify work against original goals

## Prompt Caching

Prompt caching lets agents resume from a prompt prefix without reprocessing. Critical for cost and latency.

### Why It Matters

- Cache hit rate is the most important metric for production agents
- A higher-capacity model WITH caching can be cheaper than a lower-cost model WITHOUT it
- Coding agents would be cost-prohibitive without caching

### Design for Cache-Friendliness

- Keep the system prompt and tool definitions stable (they form the cacheable prefix)
- Append new messages — don't mutate earlier ones
- If you must mutate context (removing old messages), do it at the end, not the beginning
- Batch context mutations to minimize cache invalidation

## Context Isolation

Delegate to sub-agents with isolated context windows when:

### Parallelizable Work
Fan out to independent sub-agents, each with focused context:
```
Main agent → Sub-agent A (review security)
           → Sub-agent B (review performance)  
           → Sub-agent C (review style)
           → Combine results
```

### Long-Running Tasks
Context lives in files. Each sub-agent gets a fresh window:
```
Iteration 1: Agent reads plan → does work → writes results → exits
Iteration 2: New agent reads plan → reads results → continues → exits
...
```
Progress communicated via git history and shared files.

### Map-Reduce
Split bulk work across sub-agents:
```
Task: Update lint rules across 50 files
→ Sub-agent 1: files 1-10
→ Sub-agent 2: files 11-20
→ ...
→ Merge results
```

## The Ralph Loop

Named pattern for tasks exceeding a single context window:

```
1. Initializer agent:
   - Analyzes task
   - Creates plan.md (checklist of sub-tasks)
   - Creates progress.md (tracking file)
   - Commits to git

2. Worker loop (repeats until plan satisfied):
   - Fresh agent reads plan.md + progress.md
   - Tackles next unchecked item
   - Updates progress.md
   - Commits work to git
   - Stop hook verifies work quality

3. Exit when:
   - All plan items checked off
   - Maximum iterations reached
   - Stop hook rejects work N times
```

### Key Design Decisions

- **Plan granularity:** Items should be completable in a single agent session
- **Progress format:** Machine-readable (checkboxes, JSON) so agents can parse state
- **Verification:** Stop hooks run after each iteration to catch problems early
- **Git as memory:** Each commit is a checkpoint; agents can review git log for history

## Evolving Context

Agents can learn from experience through context evolution:

### Task-Specific Prompt Optimization
1. Collect agent trajectories (successes and failures)
2. Score outcomes
3. Reflect on failures — identify what went wrong
4. Propose prompt variants
5. Test variants, keep improvements

### Memory Learning
1. Distill agent sessions into structured entries
2. Reflect across entries for patterns
3. Update persistent memory (e.g., CLAUDE.md, memory files)
4. Agent loads relevant memories in future sessions

### Skill Discovery
1. Reflect over trajectories for reusable procedures
2. Distill into skill files (SKILL.md format)
3. Save to filesystem
4. Agent discovers and loads skills via progressive disclosure

### The Sleep-Time Pattern
Agents think offline about their own context — consolidate memories, prepare for future tasks, clean up stale information. Like human sleep for memory consolidation.
