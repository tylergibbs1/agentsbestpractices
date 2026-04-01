---
name: designing-agents
description: Designs agentic AI systems by selecting the right architecture pattern and context engineering strategy. Triggers on "build an agent," "agent architecture," "agent design," "multi-agent," "agent pattern," "orchestration," "context engineering," "sub-agents," "agent workflow," "Ralph loop," "ReAct," "swarm," "coordinator pattern," "how should I structure my agent," "my agent loses context," "agent keeps failing on long tasks." For tool-level design, see designing-agent-tools.
metadata:
  version: 1.0.0
---

# Designing Agents

Expert guidance for choosing architecture patterns and context strategies for agentic AI systems.

## Before Designing

Gather this context (ask if not provided):

1. **Task shape** — Is the workflow predictable/structured or open-ended? How many steps? Can steps run in parallel?
2. **Constraints** — Latency budget? Cost sensitivity? Need for human approval at checkpoints?
3. **Context pressure** — How much information must the agent reason over? Will tasks exceed a single context window?
4. **Existing infrastructure** — What models, tools, and frameworks are already in play?

---

## Core Principle: Context Is Everything

Effective agent design boils down to context management. Models degrade as context grows — every token depletes an "attention budget." The central design question is:

> How do you fill the context window with just the right information for the next step?

All patterns below are strategies for managing this constraint.

---

## Start Simple, Add Complexity Only When Measured

The most successful agent implementations use simple, composable patterns — not complex frameworks. Start with the simplest approach that could work:

1. **Single LLM call** with retrieval and examples — sufficient for many tasks
2. **Prompt chaining** — decompose into fixed sequential steps
3. **Single agent with tools** — when the task needs autonomous tool use
4. **Multi-agent** — only when a single agent demonstrably fails

Add complexity only when it demonstrably improves outcomes. Measure first.

---

## Pattern Selection

### Decision Framework

| Your situation | Start with |
|---|---|
| Task has fixed, known steps | **Prompt chaining** or **sequential agents** |
| Input determines which specialist handles it | **Routing** |
| Independent sub-tasks that can run simultaneously | **Parallel agents** |
| Open-ended task needing autonomous tool use | **Single agent** (ReAct loop) |
| Single agent struggles with too many responsibilities | **Coordinator** with specialized sub-agents |
| Output needs iterative quality improvement | **Evaluator-optimizer** loop |
| Ambiguous problem requiring multi-step decomposition | **Hierarchical task decomposition** |
| High-stakes decisions requiring human judgment | **Human-in-the-loop** checkpoints |

For detailed pattern descriptions and trade-offs: See [references/design-patterns.md](references/design-patterns.md)

---

## Context Engineering Strategies

These strategies apply across all patterns. Mix and match based on your context pressure.

### Give Agents a Computer

Agents benefit from OS-level primitives — filesystem and shell. The filesystem gives persistent context. The shell lets agents run utilities, CLIs, scripts, or code they write.

The fundamental agent abstraction is the CLI. Think of it as "AI for your operating system."

### Multi-Layer Action Space

Keep the tool-calling layer small (10-20 tools). Push actions to the computer:

```
Layer 1: Tool calls (small set of atomic tools — bash, file read/write, search)
Layer 2: Shell utilities and CLIs (progressively disclosed via --help)
Layer 3: Code the agent writes and executes (chains many actions, saves tokens)
```

Writing and executing code chains many actions without processing intermediate tool results — significant token savings.

### Progressive Disclosure

Show only essential information upfront, reveal details on demand:

- **Tool definitions** — index and retrieve on demand, not all upfront
- **CLI tools** — list available utilities in instructions, agent reads `--help` when needed
- **MCP servers** — sync descriptions to a folder, agent reads full spec only if needed
- **Skills** — YAML frontmatter loaded into context, full SKILL.md read only when triggered

### Offload Context to Filesystem

Write information out of the context window into files:

- Old tool results → files (read back only if needed)
- Agent plans → plan file (read periodically to reinforce objectives)
- Agent trajectories → log files (available for review without consuming context)

Apply summarization only after offloading has diminishing returns. Summarization loses information; offloading preserves it.

### Cache Context

Prompt caching lets agents resume from a prefix without re-processing. Cache hit rate is the most important metric for production agents — a higher-capacity model with caching can be cheaper than a lower-cost model without it.

Design context mutations (adding/removing blocks) with cache-friendliness in mind.

### Isolate Context with Sub-Agents

Delegate tasks to sub-agents with isolated context windows, tools, and instructions:

- **Parallelizable tasks** — fan out to independent sub-agents (e.g., code review checking different issues)
- **Long-running tasks** — context lives in files, progress communicated via git history
- **Map-reduce** — split work, combine results (e.g., lint rule updates across many files)

### The Ralph Loop

For tasks that exceed a single context window, use two specialized agent types:

**Initializer agent** (runs once):
- Sets up the environment (dev server scripts, git repo)
- Creates a structured feature/task list (JSON, not markdown — less prone to overwrites)
- Writes progress tracking file
- Commits initial state to git

**Worker agent** (loops until done):
1. Reads git log + progress file to understand current state
2. Runs setup scripts (e.g., `init.sh`) to restore environment
3. Verifies existing work still passes before starting new work
4. Tackles one task from the plan
5. Self-verifies with tests before marking complete
6. Commits progress with descriptive messages
7. Updates progress file

**Critical rules:**
- Agents may only update the `passes` field on tasks — never remove or edit tests
- Use JSON for state files (structured, machine-parseable, less corruption risk)
- Each session starts by reading state, not by asking the user what to do
- End each session in a production-ready state

For implementation details and failure modes: See [references/long-running-harness.md](references/long-running-harness.md)

### Evolve Context Over Time

Agents can learn from experience by reflecting over past sessions:

- **Task-specific prompts** — score trajectories, reflect on failures, propose prompt variants
- **Memory** — distill sessions into entries, reflect across entries, update persistent context
- **Skills** — reflect over trajectories to distill reusable procedures, save as new skills

For detailed context strategies: See [references/context-engineering.md](references/context-engineering.md)

---

## Agent Design Workflow

Copy this checklist and track progress:

```
Agent Design Progress:
- [ ] Step 1: Define task shape (structured vs open-ended, step count, parallelism)
- [ ] Step 2: Start with simplest viable pattern
- [ ] Step 3: Build single-agent prototype with core tools
- [ ] Step 4: Test on real prompts — identify where it fails
- [ ] Step 5: Apply context strategies (offloading, progressive disclosure, caching)
- [ ] Step 6: If single agent insufficient, decompose into sub-agents
- [ ] Step 7: Add human-in-the-loop checkpoints for high-stakes decisions
- [ ] Step 8: Measure: latency, cost, accuracy, context utilization
- [ ] Step 9: Iterate — add complexity only where measurements justify it
```

---

## Anti-Patterns

- **Premature multi-agent** — Adding agents before proving a single agent can't handle it
- **Framework over understanding** — Using complex frameworks without understanding the underlying prompts and responses
- **Monolithic prompt** — One agent with 50 tools and a giant system prompt instead of decomposing
- **Context stuffing** — Loading everything into context upfront instead of progressive disclosure
- **Summarize-first** — Compressing context before trying to offload it to files
- **No caching strategy** — Running agents without prompt caching, making them cost-prohibitive
- **Unverified loops** — Long-running agent loops without checkpoints or exit conditions

---

## Output Format

When designing an agent system, deliver:

1. **Architecture diagram** — which agents, their responsibilities, how they communicate
2. **Pattern justification** — why this pattern fits the task shape and constraints
3. **Context strategy** — how context is managed (disclosure, offloading, caching, isolation)
4. **Tool inventory** — what tools each agent needs (keep it small)
5. **Human checkpoints** — where human review is needed
6. **Exit conditions** — how loops terminate, how the system knows it's done

---

## Related Skills

- **designing-agent-tools**: For tool-level design (schemas, descriptions, responses, evaluation)
